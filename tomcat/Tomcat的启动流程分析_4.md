## Tomcat的线程模型：Poller

前面讲了tomcat的线程模型，如图：
![tomcat线程模型](../../assets/tomcat_thread_poller.png)
下面具体将Poller线程部分处理过程。

### Poller线程启动执行
每个poller持有一个selector，在初始化poller时创建selector。下面分析下Poller，线程启动时执行的方法为：
```java
public void run() {
    // Loop until destroy() is called
    while (true) {

        boolean hasEvents = false;

        try {
            if (!close) {
								///////同步队列中是否有PollerEvent请求，为非阻塞等待
                hasEvents = events();
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //如果wakeupCounter返回值大于0，说明有事件等待，需要立即返回
                    keyCount = selector.selectNow();
                } else {
										//阻塞等待：等待超时或者有事件返回
                    keyCount = selector.select(selectorTimeout);
                }
                wakeupCounter.set(0);
            }
            if (close) {
								//关闭时，将剩余的任务执行完成
                events();
								//执行socket事件相关操作
                timeout(0, false);
                try {
										//关闭selector
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error("",x);
            continue;
        }
        //either we timed out or we woke up, process events first
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());

        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        /////// 处理准备好的keys，即需要处理的事件到来
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();

            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                ///////处理socket事件
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}
```
#### events()判断
```java
//events()方法：其中events队列为：SynchronizedQueue<PollerEvent>
public boolean events() {
    boolean result = false;

    PollerEvent pe = null;
    while ( (pe = events.poll()) != null ) {
        result = true;
        try {
            pe.run();
            pe.reset();
            if (running && !paused) {
                eventCache.push(pe);
            }
        } catch ( Throwable x ) {
            log.error("",x);
        }
    }

    return result;
}
```
#### PollerEvent结构和功能
Acceptor线程接到socket连接请求后，会将socket与Poller绑定，即向Poller的Selector注册刚兴趣的事件。
```java
//PollerEvent类用于socket向Selector上面注册感兴趣的事件，具体何时将PollerEvent放入
//SynchronizedQueue中的，在讲Acceptor会讲到
public static class PollerEvent implements Runnable {
	private NioChannel socket;
  private int interestOps;
  private NioSocketWrapper socketWrapper;
	public void run() {
	  if (interestOps == OP_REGISTER) {
	      try {
	          socket.getIOChannel().register(
	                  socket.getPoller().getSelector(), SelectionKey.OP_READ, socketWrapper);
	      } catch (Exception x) {
	          log.error(sm.getString("endpoint.nio.registerFail"), x);
	      }
	  } else {
	      final SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
	      try {
	          if (key == null) {
	              // The key was cancelled (e.g. due to socket closure)
	              // and removed from the selector while it was being
	              // processed. Count down the connections at this point
	              // since it won't have been counted down when the socket
	              // closed.
	              socket.socketWrapper.getEndpoint().countDownConnection();
	          } else {
	              final NioSocketWrapper socketWrapper = (NioSocketWrapper) key.attachment();
	              if (socketWrapper != null) {
	                  //we are registering the key to start with, reset the fairness counter.
	                  int ops = key.interestOps() | interestOps;
	                  socketWrapper.interestOps(ops);
	                  key.interestOps(ops);
	              } else {
	                  socket.getPoller().cancelledKey(key);
	              }
	          }
	      } catch (CancelledKeyException ckx) {
	          try {
	              socket.getPoller().cancelledKey(key);
	          } catch (Exception ignore) {}
	      }
	  }
	}
}
```
#### processKey处理到达的事件
处理selector轮询到的准备好的事件（即读、写请求），
```java
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        if ( close ) {
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            if (sk.isReadable() || sk.isWritable() ) {
                if ( attachment.getSendfileData() != null ) {
                    ////////处理文件任务
                    processSendfile(sk,attachment, false);
                } else {
                    unreg(sk, attachment, sk.readyOps());
                    boolean closeSocket = false;
                    // Read goes before write
                    if (sk.isReadable()) {
                        ////////处理读事件
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            closeSocket = true;
                        }
                    }
                    if (!closeSocket && sk.isWritable()) {
                        ////////处理写事件
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("",t);
    }
}
```
#### processSocket处理socket读写请求
对于socket读写事件，然后封装成SocketProcessor交由线程池处理，并且将SocketProcessor实例加入缓存，在下次处理请求中直接从processorCache中获取并重置，避免了重复创建线程实例。
```java
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
    try {
        if (socketWrapper == null) {
            return false;
        }
        SocketProcessorBase<S> sc = processorCache.pop();
        if (sc == null) {
						////////创建socketprocessor，执行完毕后会将此processor放入processorCache中
            sc = createSocketProcessor(socketWrapper, event);
        } else {
						////////如果缓存中存在，则重置此processor
            sc.reset(socketWrapper, event);
        }
				//使用线程池执行任务
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            ////////线程池处理
            executor.execute(sc);
        } else {
            sc.run();
        }
    } catch (RejectedExecutionException ree) {
        return false;
    } catch (Throwable t) {
        return false;
    }
    return true;
}
```
#### SocketProcessor读写事件处理socket线程
```java
protected class SocketProcessor extends SocketProcessorBase<NioChannel> {

    public SocketProcessor(SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
        super(socketWrapper, event);
    }

    @Override
    protected void doRun() {
        NioChannel socket = socketWrapper.getSocket();
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

        try {
            int handshake = -1;

            try {
								//判断socket连接握手情况
                if (key != null) {
                    if (socket.isHandshakeComplete()) {
                        handshake = 0;
                    } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                            event == SocketEvent.ERROR) {
                        handshake = -1;
                    } else {
                        handshake = socket.handshake(key.isReadable(), key.isWritable());
                        event = SocketEvent.OPEN_READ;
                    }
                }
            } catch (IOException x) {
                handshake = -1;
                if (log.isDebugEnabled()) log.debug("Error during SSL handshake",x);
            } catch (CancelledKeyException ckx) {
                handshake = -1;
            }
            if (handshake == 0) {
                SocketState state = SocketState.OPEN;
                // 处理socket请求：调用AbstractProtocol的process方法
                if (event == null) {
                    state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                } else {
										//
                    state = getHandler().process(socketWrapper, event);
                }
								//关闭socket
                if (state == SocketState.CLOSED) {
                    close(socket, key);
                }
            } else if (handshake == -1 ) {
                close(socket, key);
            } else if (handshake == SelectionKey.OP_READ){
                socketWrapper.registerReadInterest();
            } else if (handshake == SelectionKey.OP_WRITE){
                socketWrapper.registerWriteInterest();
            }
        } catch (CancelledKeyException cx) {
            socket.getPoller().cancelledKey(key);
        } catch (VirtualMachineError vme) {
            ExceptionUtils.handleThrowable(vme);
        } catch (Throwable t) {
            log.error("", t);
            socket.getPoller().cancelledKey(key);
        } finally {
            socketWrapper = null;
            event = null;
            //return to cache
            if (running && !paused) {
								//将SocketProcessor实例放入缓存
                processorCache.push(this);
            }
        }
    }
}
```
#### Protocol处理SocketEvent
看socket处理的情况，即`getHandler().process(socketWrapper, event);`进入AbstractProtocol类：
```java
public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
    //...

    S socket = wrapper.getSocket();

    ///////协议处理器Http11Processor
    Processor processor = connections.get(socket);
    //...
    if (processor != null) {
        // 如果不为空，去掉等待的processor，防止AsyncTimeout重复触发
        getProtocol().removeWaitingProcessor(processor);
    } else if (status == SocketEvent.DISCONNECT || status == SocketEvent.ERROR) {
        return SocketState.CLOSED;
    }

    ContainerThreadMarker.set();

    try {
        if (processor == null) {
					  //TODO：是否升级协议
            String negotiatedProtocol = wrapper.getNegotiatedProtocol();
            if (negotiatedProtocol != null) {
                UpgradeProtocol upgradeProtocol =
                        getProtocol().getNegotiatedProtocol(negotiatedProtocol);
                if (upgradeProtocol != null) {
                    processor = upgradeProtocol.getProcessor(
                            wrapper, getProtocol().getAdapter());
                } else if (negotiatedProtocol.equals("http/1.1")) {

                } else {

                    return SocketState.CLOSED;

                }
            }
        }
        if (processor == null) {
						//release processor时，如果不是upgrade请求，则放入recycledProcessors重用
            processor = recycledProcessors.pop();
        }
        if (processor == null) {
						//不存在processor的情况，则创建新的processor
            processor = getProtocol().createProcessor();
            register(processor);
        }

        processor.setSslSupport(
                wrapper.getSslSupport(getProtocol().getClientCertProvider()));

        // Associate the processor with the connection
        connections.put(socket, processor);

        SocketState state = SocketState.CLOSED;
        do {
						//执行AbstractProcessorLight.process方法
            state = processor.process(wrapper, status);
						//如果是升级协议请求的情况：此块暂不分析
            if (state == SocketState.UPGRADING) {
                // Get the HTTP upgrade handler
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                // Retrieve leftover input
                ByteBuffer leftOverInput = processor.getLeftoverInput();
                if (upgradeToken == null) {
                    // Assume direct HTTP/2 connection
                    UpgradeProtocol upgradeProtocol = getProtocol().getUpgradeProtocol("h2c");
                    if (upgradeProtocol != null) {
                        processor = upgradeProtocol.getProcessor(
                                wrapper, getProtocol().getAdapter());
                        wrapper.unRead(leftOverInput);
                        // Associate with the processor with the connection
                        connections.put(socket, processor);
                    } else {
                        return SocketState.CLOSED;
                    }
                } else {
                    HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                    // Release the Http11 processor to be re-used
                    release(processor);
                    // Create the upgrade processor
                    processor = getProtocol().createUpgradeProcessor(wrapper, upgradeToken);

                    wrapper.unRead(leftOverInput);

                    wrapper.setUpgraded(true);

                    connections.put(socket, processor);

                    if (upgradeToken.getInstanceManager() == null) {
                        httpUpgradeHandler.init((WebConnection) processor);
                    } else {
                        ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                        try {
                            httpUpgradeHandler.init((WebConnection) processor);
                        } finally {
                            upgradeToken.getContextBind().unbind(false, oldCL);
                        }
                    }
                }
            }
        } while ( state == SocketState.UPGRADING);

        ///////
        if (state == SocketState.LONG) {
            longPoll(wrapper, processor);
            if (processor.isAsync()) {
                getProtocol().addWaitingProcessor(processor);
            }
        } else if (state == SocketState.OPEN) {
            connections.remove(socket);
            release(processor);
            wrapper.registerReadInterest();
        } else if (state == SocketState.SENDFILE) {
        } else if (state == SocketState.UPGRADED) {
            if (status != SocketEvent.OPEN_WRITE) {
                longPoll(wrapper, processor);
            }
        } else {
            connections.remove(socket);
            if (processor.isUpgrade()) {
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                InstanceManager instanceManager = upgradeToken.getInstanceManager();
                if (instanceManager == null) {
                    httpUpgradeHandler.destroy();
                } else {
                    ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                    try {
                        httpUpgradeHandler.destroy();
                    } finally {
                        try {
                            instanceManager.destroyInstance(httpUpgradeHandler);
                        } catch (Throwable e) {
                            ExceptionUtils.handleThrowable(e);
                            getLog().error(sm.getString("abstractConnectionHandler.error"), e);
                        }
                        upgradeToken.getContextBind().unbind(false, oldCL);
                    }
                }
            } else {
                release(processor);
            }
        }
        return state;
    } catch(java.net.SocketException e) {
    } catch (java.io.IOException e) {
    } catch (ProtocolException e) {
    } catch (Throwable e) {
    } finally {
        ContainerThreadMarker.clear();
    }
    connections.remove(socket);
    release(processor);
    return SocketState.CLOSED;
}
```
##### 执行AbstractProcessorLight（Http11Processor）的process
```java
public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
            throws IOException {

    SocketState state = SocketState.CLOSED;
    Iterator<DispatchType> dispatches = null;
    do {
        if (dispatches != null) {
            DispatchType nextDispatch = dispatches.next();
            ////////
            state = dispatch(nextDispatch.getSocketStatus());
        } else if (status == SocketEvent.DISCONNECT) {
            // Do nothing here, just wait for it to get recycled
        } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
            ////////
            state = dispatch(status);
            if (state == SocketState.OPEN) {
                state = service(socketWrapper);
            }
        } else if (status == SocketEvent.OPEN_WRITE) {
            // Extra write event likely after async, ignore
            state = SocketState.LONG;
        } else if (status == SocketEvent.OPEN_READ){
						////////根据上面传入的时间，第一次先执行读取事件，执行Http11Processor的service方法
            state = service(socketWrapper);
        } else {
            // Default to closing the socket if the SocketEvent passed in
            // is not consistent with the current state of the Processor
            state = SocketState.CLOSED;
        }

        if (state != SocketState.CLOSED && isAsync()) {
            state = asyncPostProcess();
        }

        if (getLog().isDebugEnabled()) {
            getLog().debug("Socket: [" + socketWrapper +
                    "], Status in: [" + status +
                    "], State out: [" + state + "]");
        }

        if (dispatches == null || !dispatches.hasNext()) {
            // Only returns non-null iterator if there are
            // dispatches to process.
            dispatches = getIteratorAndClearDispatches();
        }
    } while (state == SocketState.ASYNC_END ||
            dispatches != null && state != SocketState.CLOSED);

    return state;
}
```
#### AbstractProcessor的dispatch方法
```java
public final SocketState dispatch(SocketEvent status) {

    if (status == SocketEvent.OPEN_WRITE && response.getWriteListener() != null) {
        asyncStateMachine.asyncOperation();
        try {
            //如果缓冲区还有内容需要发送，返回state=LONG，保持连接
            if (flushBufferedWrite()) {
                return SocketState.LONG;
            }
        } catch (IOException ioe) {
            if (getLog().isDebugEnabled()) {
                getLog().debug("Unable to write async data.", ioe);
            }
            status = SocketEvent.ERROR;
            request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ioe);
        }
    } else if (status == SocketEvent.OPEN_READ && request.getReadListener() != null) {
        dispatchNonBlockingRead();
    } else if (status == SocketEvent.ERROR)

        if (request.getAttribute(RequestDispatcher.ERROR_EXCEPTION) == null) {

            request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, socketWrapper.getError());
        }

        if (request.getReadListener() != null || response.getWriteListener() != null) {

            asyncStateMachine.asyncOperation();
        }
    }

    ///////......

    if (getErrorState().isError()) {
        request.updateCounters();
        return SocketState.CLOSED;
    } else if (isAsync()) {
        return SocketState.LONG;
    } else {
        request.updateCounters();
        //最终会调用到AbstractProcessor类的action()的commit处理分支，响应请求内容
        return dispatchEndRequest();
    }
}
```
#### Http11Processor的service方法处理请求
http11processor的service方法，处理具体的socket请求内容：
```java
public SocketState service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
    RequestInfo rp = request.getRequestProcessor();
    rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

		//初始化读取和写入缓冲区
    setSocketWrapper(socketWrapper);
    inputBuffer.init(socketWrapper);
    outputBuffer.init(socketWrapper);

    // Flags
    keepAlive = true;
    openSocket = false;
    readComplete = true;
    boolean keptAlive = false;
    SendfileState sendfileState = SendfileState.DONE;

    while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
            sendfileState == SendfileState.DONE && !endpoint.isPaused()) {


        try {
						//解析http header： GET http/1.1 uri
            if (!inputBuffer.parseRequestLine(keptAlive)) {
								//为-1的情况表示是http2请求，当前协议为http1.1，需要进行协议升级
                if (inputBuffer.getParsingRequestLinePhase() == -1) {
                    return SocketState.UPGRADING;
                } else if (handleIncompleteRequestLineRead()) {
                    break;
                }
            }

            if (endpoint.isPaused()) {
                // 503 - Service unavailable
                response.setStatus(503);
                setErrorState(ErrorState.CLOSE_CLEAN, null);
            } else {
                keptAlive = true;
                request.getMimeHeaders().setLimit(endpoint.getMaxHeaderCount());
                ////////解析http协议的请求头，如果解析完成，则终止循环
                if (!inputBuffer.parseHeaders()) {

                    openSocket = true;
                    readComplete = false;
                    break;
                }
                if (!disableUploadTimeout) {
                    socketWrapper.setReadTimeout(connectionUploadTimeout);
                }
            }
        } catch (IOException e) {
            setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
            break;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // 400 - Bad Request
            response.setStatus(400);
            setErrorState(ErrorState.CLOSE_CLEAN, t);
        }

        // 判断请求头中是否有“Connection:upgrade”参数
        Enumeration<String> connectionValues = request.getMimeHeaders().values("Connection");
        boolean foundUpgrade = false;
        while (connectionValues.hasMoreElements() && !foundUpgrade) {
            foundUpgrade = connectionValues.nextElement().toLowerCase(
                    Locale.ENGLISH).contains("upgrade");
        }

				//如果当前请求表示升级协议的情况
        if (foundUpgrade) {
            // Check the protocol
            String requestedProtocol = request.getHeader("Upgrade");

            UpgradeProtocol upgradeProtocol = httpUpgradeProtocols.get(requestedProtocol);
            if (upgradeProtocol != null) {
                if (upgradeProtocol.accept(request)) {
                    // TODO Figure out how to handle request bodies at this
                    // point.
                    response.setStatus(HttpServletResponse.SC_SWITCHING_PROTOCOLS);
                    response.setHeader("Connection", "Upgrade");
                    response.setHeader("Upgrade", requestedProtocol);
										//关闭socket，刷新缓冲区
                    action(ActionCode.CLOSE,  null);
                    getAdapter().log(request, response, 0);

                    InternalHttpUpgradeHandler upgradeHandler =
                            upgradeProtocol.getInternalUpgradeHandler(
                                    getAdapter(), cloneRequest(request));
                    UpgradeToken upgradeToken = new UpgradeToken(upgradeHandler, null, null);
                    action(ActionCode.UPGRADE, upgradeToken);
                    return SocketState.UPGRADING;
                }
            }
        }

        if (!getErrorState().isError()) {
            // Setting up filters, and parse some request headers
            rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
            try {
								///////////1.处理请求协议
                prepareRequest();
            } catch (Throwable t) {
                // 500 - Internal Server Error
                response.setStatus(500);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
            }
        }

        //...

        // Process the request in the adapter
        if (!getErrorState().isError()) {
            try {
                rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
								///////////2.执行具体的请求：执行CoyoteAdpter.service方法
                getAdapter().service(request, response);

                if(keepAlive && !getErrorState().isError() && !isAsync() &&
                        statusDropsConnection(response.getStatus())) {
                    setErrorState(ErrorState.CLOSE_CLEAN, null);
                }
            } catch (InterruptedIOException e) {
                setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
            } catch (HeadersTooLargeException e) {

                if (response.isCommitted()) {
                    setErrorState(ErrorState.CLOSE_NOW, e);
                } else {
                    response.reset();
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, e);
                    response.setHeader("Connection", "close"); // TODO: Remove
                }
            } catch (Throwable t) {
                // 500 - Internal Server Error
                response.setStatus(500);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
                getAdapter().log(request, response, 0);
            }
        }

        // Finish the handling of the request
        rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);
        if (!isAsync()) {
            ///////////3.结束请求
            endRequest();
        }
        rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);

        // If there was an error, make sure the request is counted as
        // and error, and update the statistics counter
        if (getErrorState().isError()) {
            response.setStatus(500);
        }

        if (!isAsync() || getErrorState().isError()) {
            request.updateCounters();
            if (getErrorState().isIoAllowed()) {
                inputBuffer.nextRequest();
                outputBuffer.nextRequest();
            }
        }

        if (!disableUploadTimeout) {
            int soTimeout = endpoint.getSoTimeout();
            if(soTimeout > 0) {
                socketWrapper.setReadTimeout(soTimeout);
            } else {
                socketWrapper.setReadTimeout(0);
            }
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_KEEPALIVE);

        sendfileState = processSendfile(socketWrapper);
    }

    rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);

    if (getErrorState().isError() || endpoint.isPaused()) {
        return SocketState.CLOSED;
    } else if (isAsync()) {
        return SocketState.LONG;
    } else if (isUpgrade()) {
        return SocketState.UPGRADING;
    } else {
        if (sendfileState == SendfileState.PENDING) {
            return SocketState.SENDFILE;
        } else {
            if (openSocket) {
                if (readComplete) {
                    return SocketState.OPEN;
                } else {
                    return SocketState.LONG;
                }
            } else {
                return SocketState.CLOSED;
            }
        }
    }
}
```
#### CoyoteAdapter的service方法
```java
public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {

    Request request = (Request) req.getNote(ADAPTER_NOTES);
    Response response = (Response) res.getNote(ADAPTER_NOTES);

    ///////...

    boolean async = false;
    boolean postParseSuccess = false;

    req.getRequestProcessor().setWorkerThreadName(THREAD_NAME.get());

    try {
        //////// 处理请求参数，cookie、session以及tomcat认证
        postParseSuccess = postParseRequest(req, request, res, response);
        if (postParseSuccess) {
            //check valves if we support async
            request.setAsyncSupported(
                    connector.getService().getContainer().getPipeline().isAsyncSupported());
            //////// Pipeline链式处理
            //处理过程如下：StandardServiceValue->StandardEngineValue->StandardHostValue->StandardContextValue
            connector.getService().getContainer().getPipeline().getFirst().invoke(
                    request, response);
        }

        //如果请求是异步的
        if (request.isAsync()) {
            async = true;
            ReadListener readListener = req.getReadListener();
            if (readListener != null && request.isFinished()) {
                ClassLoader oldCL = null;
                try {
                    oldCL = request.getContext().bind(false, null);
                    if (req.sendAllDataReadEvent()) {
                        req.getReadListener().onAllDataRead();
                    }
                } finally {
                    request.getContext().unbind(false, oldCL);
                }
            }

            Throwable throwable =
                    (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

            if (!request.isAsyncCompleting() && throwable != null) {
                request.getAsyncContextInternal().setErrorState(throwable, true);
            }
        } else {
            //同步请求
            //如果状态码为415：表示请求超出服务处理的限制
            request.finishRequest();
            //刷新缓冲区，close请求
            response.finishResponse();
        }

    } catch (IOException e) {
        // Ignore
    } finally {
        //...省略

        req.getRequestProcessor().setWorkerThreadName(null);

        // Recycle the wrapper request and response
        if (!async) {
            request.recycle();
            response.recycle();
        }
    }
}
```
#### CoyoteAdapter的postParseRequest方法
此方法用于解析请求协议、通信端口、支持的请求方法、session设置等。
```java
protected boolean postParseRequest(org.apache.coyote.Request req, Request request,
            org.apache.coyote.Response res, Response response) throws IOException, ServletException {

    //1、设置请求协议：http或https
    //....

    //2、设置服务端口
    //....

    //3、根据请求路径判断请求内容
    MessageBytes undecodedURI = req.requestURI();

    //如果是 OPTIONS请求则返回false
    //...

    MessageBytes decodedURI = req.decodedURI();

    //4、URI类型为bytes类型
    if (undecodedURI.getType() == MessageBytes.T_BYTES) {
        // Copy the raw URI to the decodedURI
        decodedURI.duplicate(undecodedURI);

        // 解析请求路径参数
        parsePathParameters(req, request);

        // 请求路径decode，如果异常，设置400状态，返回false
        // %xx decoding of the URL
        //...
    } else {
        //如果为char或者String类型
    }

    // 5、设置servername
    //...

    String version = null;
    Context versionContext = null;
    boolean mapRequired = true;

    //6、设置sessionId，并且寻找对应的上下文
    while (mapRequired) {
        // This will map the the latest version by default
        connector.getService().getMapper().map(serverName, decodedURI,
                version, request.getMappingData());

        // 判断是否有应用上下文，否则设置404，法规false
        //...

        // 设置sessionId,会从cookie和ssl中查找sessionid标志的值

        sessionID = request.getRequestedSessionId();

        mapRequired = false;
        //查找sessionID对应的应用上下文，如果找到则退出循环

    }

    // 7、判断是否有redirectPath，如果有，则设置response的redirect，并返回false
    //...

    // 8、过滤trace方法：如果不支持Trace请求，而当前方法为Trace请求，则设置状态码为405，返回405
    //...

    //9、判断是否需要验证tomcat的realm
    doConnectorAuthenticationAuthorization(req, request);

    return true;
}
```
#### StandardEngineValue
上述代码中
> connector.getService().getContainer().getPipeline().getFirst().invoke(
                    request, response);

可以看到pipeline使用链式处理，service的容器为StandardEngine，getFirst会判断first，
如果为null，则取basic，而初始化的时候设置了basic为StandardEngineValue。下面进入invoke中：
```java
//取子容器host处理请求，如果host为空，说明不存在对应的服务，即返回404,；如果存在host，则调用host处理请求
public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Select the Host to be used for this Request
        Host host = request.getHost();
        if (host == null) {
            response.sendError
                (HttpServletResponse.SC_BAD_REQUEST,
                 sm.getString("standardEngine.noHost",
                              request.getServerName()));
            return;
        }
        if (request.isAsyncSupported()) {
            request.setAsyncSupported(host.getPipeline().isAsyncSupported());
        }

        // 调用host处理请求：同StandardEngineValue,会进入到StandardHostValue中处理
        host.getPipeline().getFirst().invoke(request, response);
    }
}
```
#### StandardHostValue
```java
//选择Host的子容器Context，即应用上下问进行处理请求，如果不存在应用上下文，则返回状态码500
public final void invoke(Request request, Response response)
        throws IOException, ServletException {

    // Select the Context to be used for this Request
    Context context = request.getContext();
    if (context == null) {
        response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
             sm.getString("standardHost.noContext"));
        return;
    }

    if (request.isAsyncSupported()) {
        request.setAsyncSupported(context.getPipeline().isAsyncSupported());
    }

    boolean asyncAtStart = request.isAsync();
    boolean asyncDispatching = request.isAsyncDispatching();

    try {
        //设置context的classloader为web应用的classloader
        context.bind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);

        //...

        //交由Context处理请求，如果该请求为异步状态或者没有找到对应的资源，则路由到应用定义的错误页面
        try {
            if (!asyncAtStart || asyncDispatching) {
                //////调用StandardContextValue执行invoke方法
                context.getPipeline().getFirst().invoke(request, response);
            } else {
                // Make sure this request/response is here because an error
                // report is required.
                if (!response.isErrorReportRequired()) {
                    throw new IllegalStateException(sm.getString("standardHost.asyncStateError"));
                }
            }
        } catch (Throwable t) {
          //...
        }

        //...

        //发起请求销毁监听事件：重置local和requestAttributes，执行请求销毁回调函数，更新session attibutes，
        //其中回调函数何时注册可以参考下图
        if (!request.isAsync() && !asyncAtStart) {
            context.fireRequestDestroyEvent(request.getRequest());
        }
    } finally {
        // Access a session (if present) to update last accessed time, based
        // on a strict interpretation of the specification
        if (ACCESS_SESSION) {
            request.getSession(false);
        }

        context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
    }
}
```
注册request销毁的回调函数：
![](../../assests/tomcat_request_destruction_callback_register.png)

#### StandardContextValue
> StandardContextValue会调用StandardWrapper处理请求
```
public final void invoke(Request request, Response response)
        throws IOException, ServletException {

    // 判断资源是否在 WEB-INF or META-INF下，这两个路径下为保护资源，不作处理
    MessageBytes requestPathMB = request.getRequestPathMB();
    if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
            || (requestPathMB.equalsIgnoreCase("/META-INF"))
            || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
            || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // 选择StandardWrapper
    Wrapper wrapper = request.getWrapper();
    if (wrapper == null || wrapper.isUnavailable()) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // 发送ACK响应：HTTP/1.1 100，表示已接收到请求，正在处理
    try {
        response.sendAcknowledgement();
    } catch (IOException ioe) {
        //...
        return;
    }

    //设置是否支持异步：为true

    //调用StandardWrapperValue处理
    wrapper.getPipeline().getFirst().invoke(request, response);
}
```
#### StandardWrapperVal
请求处理工作终于传递到最后一环servlet容器，进入到真正处理请求的地方~^~，在StandardWrapper中会执行servlet和filter等servlet容器相关的东西
处理过程如下：
1、校验Context容器是否可用，如果不可用返回状态码503
2、校验Wrapper是否可用，如果不可用，根据available大小，返回状态码503或者404
```
public final void invoke(Request request, Response response)
        throws IOException, ServletException {

    // 服务是否可用
    boolean unavailable = false;
    //记录异常
    Throwable throwable = null;
    //处理请求开始的时间
    long t1=System.currentTimeMillis();
    //记录请求数
    requestCount.incrementAndGet();
    StandardWrapper wrapper = (StandardWrapper) getContainer();
    Servlet servlet = null;
    Context context = (Context) wrapper.getParent();

    // 1、校验Context容器是否可用，如果不可用返回503
    if (!context.getState().isAvailable()) {
        response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                       sm.getString("standardContext.isUnavailable"));
        unavailable = true;
    }

    // 2、校验Wrapper是否可用，如果不可用，根据available大小，返回状态码503或者404
    if (!unavailable && wrapper.isUnavailable()) {
        container.getLogger().info(sm.getString("standardWrapper.isUnavailable",
                wrapper.getName()));
        long available = wrapper.getAvailable();
        if ((available > 0L) && (available < Long.MAX_VALUE)) {
            response.setDateHeader("Retry-After", available);
            response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                    sm.getString("standardWrapper.isUnavailable",
                            wrapper.getName()));
        } else if (available == Long.MAX_VALUE) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    sm.getString("standardWrapper.notFound",
                            wrapper.getName()));
        }
        unavailable = true;
    }

    // Allocate a servlet instance to process this request
    try {
        if (!unavailable) {
            //获取servlet：对于SpringMVC，此处应该是DispatchServlet，servlet的执行就与执行DispatchServlet的处理流程了
            //1、首先判断servlet是否单线程模式singleThreadModel，如果是多线程，并且已经初始化，则返回当前实例；
            //2、首次使用servlet，加载servlet，设置MultipartConfig和ServletSecurity注解配置，并执行servlet的init()方法；
            //3、如果当前servlet是多线程的，会记录servlet的处理请求数；如果是单线程的，则会创建后的实例放入到堆栈中，每次从堆栈中请求。
            servlet = wrapper.allocate();
        }
    } catch (UnavailableException e) {
        //...
    } catch (ServletException e) {
        //...
    } catch (Throwable e) {
        //...
    }

    MessageBytes requestPathMB = request.getRequestPathMB();
    DispatcherType dispatcherType = DispatcherType.REQUEST;
    if (request.getDispatcherType()==DispatcherType.ASYNC) dispatcherType = DispatcherType.ASYNC;
    request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
    request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
            requestPathMB);

    //创建Filter职责链：适用于当前URI和servlet
    ApplicationFilterChain filterChain =
            ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

    try {
        if ((servlet != null) && (filterChain != null)) {
            // 是否将system.out and system.err输出logger
            if (context.getSwallowOutput()) {
                try {
                    SystemLogHandler.startCapture();
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else {
                        filterChain.doFilter(request.getRequest(),
                                response.getResponse());
                    }
                } finally {
                    String log = SystemLogHandler.stopCapture();
                    if (log != null && log.length() > 0) {
                        context.getLogger().info(log);
                    }
                }
            } else {
                //是否异步：
                if (request.isAsyncDispatching()) {
                    request.getAsyncContextInternal().doInternalDispatch();
                } else {
                    //执行过滤器链:
                    //1、执行filter的doFilter方法
                    //2、执行servlet的service方法
                    filterChain.doFilter
                        (request.getRequest(), response.getResponse());
                }
            }

        }
    } catch (ClientAbortException e) {

    } catch (IOException e) {

    } catch (UnavailableException e) {

    } catch (ServletException e) {

    } catch (Throwable e) {

    }

    // 释放filter chain
    if (filterChain != null) {
        filterChain.release();
    }

    // 如果为多线程，则将当前活跃请求数减1；如果为单线程，则将servlet实例放入栈中，并通知等待线程
    try {
        if (servlet != null) {
            wrapper.deallocate(servlet);
        }
    } catch (Throwable e) {

    }

    // 如果设置了容器不可用，即设置StandardWrapper的available为Long.MAX_VALUE,
    // 则卸载servlet实例，执行servlet的destroy方法
    try {
        if ((servlet != null) &&
            (wrapper.getAvailable() == Long.MAX_VALUE)) {
            wrapper.unload();
        }
    } catch (Throwable e) {

    }

    //设置请求处理时间
    long t2=System.currentTimeMillis();

    long time=t2-t1;
    processingTime += time;
    if( time > maxTime) maxTime=time;
    if( time < minTime) minTime=time;

}
```
至此，Poller线程的处理流程已经分析完毕，这个的处理过程如下：
![](../../assets/tomcat_poller_process.png)
