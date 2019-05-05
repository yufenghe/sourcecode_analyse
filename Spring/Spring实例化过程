### Spring实例化过程

上篇文章讲了[Spring循环依赖处理分析](Spring/Spring循环依赖处理分析)包含了bean的实例化流程，
本篇文章具体分析下bean的初始化，其中主要部分代码为`AbstractAutowireCapableBeanFactory`的的doCreateBean部分。
其中doCreateBean中主要分为以下几个部分：
- [x] createBeanInstance(beanName, mbd, args)
- [x] applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName)
- [x] addSingletonFactory(beanName, ObjectFactory)
- [x] populateBean(beanName, mbd, instanceWrapper)
- [x] initializeBean(beanName, exposedObject, mbd)
- [x] getSingleton(beanName, false)
- [x] registerDisposableBeanIfNecessary

#### 1.createBeanInstance
此方法主要是根据beanClass的构造函数，然后进行实例化。在此实例化过程中还分为两种情况：
- 配置factory-method，则使用工厂方法创建，其中方法必须是static修饰的。
- 根据参数找到对应的构造函数，使用构造函数实例化bean。

#### 2.applyMergedBeanDefinitionPostProcessors
此方法用于获取依赖注入的属性，进行预解析，代码如下：
```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
```
主要是处理BeanPostProcessor中类型为`MergedBeanDefinitionPostProcessor`，主要有以下几种类型：
- AntowiredAnnotationBeanPostProcessor：历bean的Field属性，寻找标注有@Autowired和@Value注解的字段，并完成属性注入（如果能加载到javax.inject.Inject注解的话，也会扫描该注解）；
- RequiredAnnotationBeanPostProcessor：检查属性的setter方法上是否标注有@Required注解，如果带有此注解，但是未配置属性（配置文件中的<property>元素），那么就会抛出异常；
- CommonAnnotationBeanPostProcessor：遍历bean的字段，寻找通用的依赖（javax中规定的）注解，完成属性注入。通用注解包括有：javax.annotation.Resource，javax.xml.ws.WebServiceRef，javax.ejb.EJB；
- InitDestoryAnnotationBeanPostProcessor：处理`@PostConstruct`和`@PreDestroy`注解的方法，生成LifecycleMetadata；

#### 3.addSingletonFactory
bean简单实例化后，通过缓存singletonFactories暴露bean的引用，为了处理循环引用。

#### 4.populateBean
此方法进行bean的依赖注入，处理依赖的元素，下面进行具体分析：
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
        // pvs是一个MutablePropertyValues实例，里面实现了PropertyValues接口，提供属性的读写操作实现，同时可以通过调用构造函数实现深拷贝
       // 在本例中，里面存在一个propertyValueList，
		PropertyValues pvs = mbd.getPropertyValues();

		if (bw == null) {
			if (!pvs.isEmpty()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// 给InstantiationAwareBeanPostProcessors最后一次机会在属性注入前修改Bean的属性值
	    // 具体通过调用postProcessAfterInstantiation方法，如果调用返回false,表示不必继续进行依赖注入，直接返回
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

        // 根据Bean配置的依赖注入方式完成注入，默认是0，即不走以下逻辑，所有的依赖注入都需要在xml文件中有显式的配置
        // 如果设置了相关的依赖装配方式，会遍历Bean中的属性，根据类型或名称来完成相应注入，无需额外配置
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            //深拷贝当前已有的配置
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

            //处理byName的类型
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// 处理byType的类型，根据引用的类型，选择TypeConverter进行实例化
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
            //结合注入后的配置，覆盖当前配置
			pvs = newPvs;
		}
        //容器是否注册了InstantiationAwareBeanPostProcessor
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
        //是否进行依赖检查
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                        //如果有相关的后置处理器，进行后置处理，autowired类型的参数会在此装配
                       //实行依赖注入，调用AutowiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor及
                       //RequiredAnnotationBeanPostProcessor进行元素赋值
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
        //依赖检查
			if (needsDepCheck) {
                // 检查是否满足相关依赖关系，对应的depends-on属性，需要确保所有依赖的Bean先完成初始化
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

        // 将配置的<property>属性注入到bean中
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
```
##### 4.1autowireByName和autowireByType
先看autowireByName和autowireByType如何实现的。
```java
protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
        //根据bw的PropertyDescriptors过滤出所有可写的（即setter方法存在），不存在于BeanDefinition里的PropertyValues,
        //不包括简单类型，如Date/Local/CharSequence/Number等
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);

        //对尚未处理的property，即不存在于PropertyValues的属性进行处理，如果在BeanFactory包含该bean，
        //则通过getBean获取bean的实例（依赖bean实例化过程）
		for (String propertyName : propertyNames) {
			if (containsBean(propertyName)) {
          //初始化
				Object bean = getBean(propertyName);
				pvs.add(propertyName, bean);
				registerDependentBean(propertyName, beanName);
				if (logger.isDebugEnabled()) {
					logger.debug("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}
```
下面分析autowireByType的实现：
```java
protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    //获取类型转换器
	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}

	Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
    // 类似的，过滤出满足装配条件的Bean属性
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		try {
			PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
			// 装配非Object类型的属性
			if (Object.class != pd.getPropertyType()) {
                //获取相关的写方法参数
				MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
				// 定义是否允许懒加载
				boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
				DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                // 这里会根据传入desc里的入参类型，作为依赖装配的类型
			    // 再根据这个类型在BeanFacoty中查找所有类或其父类相同的BeanName
			    // 最后根据BeanName获取或初始化相应的类，然后将所有满足条件的BeanName填充到autowiredBeanNames中。
				Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
				if (autowiredArgument != null) {
					pvs.add(propertyName, autowiredArgument);
				}
				for (String autowiredBeanName : autowiredBeanNames) {
                    //注册依赖
					registerDependentBean(autowiredBeanName, beanName);
					if (logger.isDebugEnabled()) {
						logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
								propertyName + "' to bean named '" + autowiredBeanName + "'");
					}
				}
				autowiredBeanNames.clear();
			}
		}
		catch (BeansException ex) {
			throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
		}
	}
}
```
##### 4.2applyPropertyValues的实现
对于通过配置文件中给<bean>配置<property>节点来完成注入的代码实现在applyPropertyValues中：
```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    // 为空直接返回
	if (pvs == null || pvs.isEmpty()) {
		return;
	}

	MutablePropertyValues mpvs = null;
	List<PropertyValue> original;

	if (System.getSecurityManager() != null) {
		if (bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}
	}

	if (pvs instanceof MutablePropertyValues) {
		mpvs = (MutablePropertyValues) pvs;
		if (mpvs.isConverted()) {
			// 如果已被设置转换完成，直接完成配置
			try {
				bw.setPropertyValues(mpvs);
				return;
			}
			catch (BeansException ex) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Error setting property values", ex);
			}
		}
		original = mpvs.getPropertyValueList();
	}
	else {
		original = Arrays.asList(pvs.getPropertyValues());
	}

	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
        //如果类型转换器为null，则使用BeanWrapper作为类型转换器，因为BeanWrapper实现了TypeConverter接口
		converter = bw;
	}
	// 创建BeanDefinitionValueResolver解析器，用来解析未被解析的PropertyValue。
	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

	// 开始遍历检查original中的属性，对未被解析的先解析/已解析的直接加入deepCopy中，最后再填充到具体的Bean实例中
	List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
	boolean resolveNecessary = false;
	for (PropertyValue pv : original) {
	    // 如果属性已经转化，直接添加
		if (pv.isConverted()) {
			deepCopy.add(pv);
		}
		else {
			String propertyName = pv.getName();
			Object originalValue = pv.getValue();
			// 核心逻辑，解析获取实际的值
			// 解析原始值，例如RuntimeBeanReference类型，会获取BeanFactory中其真正的bean；String或TypedStringValue类型会视为一个表达式进行求值等
			Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
			Object convertedValue = resolvedValue;
			//是否能进行类型转换 ，只有没有'.'或者'[' 符号的才为true
			boolean convertible = bw.isWritableProperty(propertyName) &&
					!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
			if (convertible) {
			    // 将value转化为对应的类型
				convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
			}
			//为了避免每次创建bean的时候都重复转换类型，Spring会尽可能地将转换好的值放入对应的BeanDefinition中
			if (resolvedValue == originalValue) {
				if (convertible) {
					pv.setConvertedValue(convertedValue);
				}
				deepCopy.add(pv);
			}
			else if (convertible && originalValue instanceof TypedStringValue &&
					!((TypedStringValue) originalValue).isDynamic() &&
					!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
				pv.setConvertedValue(convertedValue);
				deepCopy.add(pv);
			}
			else {
				resolveNecessary = true;
				deepCopy.add(new PropertyValue(pv, convertedValue));
			}
		}
	}
	if (mpvs != null && !resolveNecessary) {
		mpvs.setConverted();
	}

	// 将解析、转换好的属性设置到BeanWrapper中
  //之所以bean需要BeanWrapper包装一次，也是为了方便给bean填充属性
	try {
		bw.setPropertyValues(new MutablePropertyValues(deepCopy));
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Error setting property values", ex);
	}
}
```
最终是通过BeanWrapper的setPropertyValues来完成属性的注入，而前面的工作主要是解析配置文件配置的原始值，
因为原始值可能是最终注入的值（String类型），也可能需要被解析后才能被应用，所以Spring花费大量工作来解析原始值。
解析完成后，如果有必要还得转换值的类型，最终将解析、转换好的值注入到bean中。
```java
public void setPropertyValues(PropertyValues pvs) throws BeansException {
	setPropertyValues(pvs, false, false);
}

public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
			throws BeansException {
    //保存注入时抛出的PropertyAccessException
	List<PropertyAccessException> propertyAccessExceptions = null;
	//获取需要注入的propertyValue列表
	List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues ?
			((MutablePropertyValues) pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));
	for (PropertyValue pv : propertyValues) {
		try {
			//注入属性
			setPropertyValue(pv);
		}
		catch (NotWritablePropertyException ex) {
			if (!ignoreUnknown) {
				throw ex;
			}
			// Otherwise, just ignore it and continue...
		}
		catch (NullValueInNestedPathException ex) {
			if (!ignoreInvalid) {
				throw ex;
			}
			// Otherwise, just ignore it and continue...
		}
		catch (PropertyAccessException ex) {
			if (propertyAccessExceptions == null) {
				propertyAccessExceptions = new LinkedList<PropertyAccessException>();
			}
			propertyAccessExceptions.add(ex);
		}
	}

	// 如果注入时抛出的PropertyAccessException不为空，Spring会收集起来，在这里统一抛出
	if (propertyAccessExceptions != null) {
		PropertyAccessException[] paeArray =
				propertyAccessExceptions.toArray(new PropertyAccessException[propertyAccessExceptions.size()]);
		throw new PropertyBatchUpdateException(paeArray);
	}
}
```
在上面的代码中主要是进行统一的异常处理和遍历属性列表，真正的属性注入还需要继续向下调用设置单个属性的方法setPropertyValue：
```java
public void setPropertyValue(PropertyValue pv) throws BeansException {
	setPropertyValue(pv.getName(), pv.getValue());
}

public void setPropertyValue(String propertyName, Object value) throws BeansException {
	AbstractNestablePropertyAccessor nestedPa;
	try {
		//将属性访问的工作交给AbstractNestablePropertyAccessor完成
		//当配置的属性名propertyName中包含'.'或者'['这样字符时，代表需要设置嵌套属性
		//如果存在嵌套属性，Spring会递归向下获取最终设置的属性，比如：a.b.c，Spring会递归调用获取到b，c是需要设置的属性
		//如果没有嵌套属性的话，会返回BeanWrapper自身
		nestedPa = getPropertyAccessorForPropertyPath(propertyName);
	}
	catch (NotReadablePropertyException ex) {
		throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
				"Nested property in path '" + propertyName + "' does not exist", ex);
	}
	PropertyTokenHolder tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
	//设置属性
	nestedPa.setPropertyValue(tokens, new PropertyValue(propertyName, value));
}
```
上面主要是解析propertyName，处理嵌套的属性，确定最终需要设置的属性。
```java
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
	//如果propertyName中存在'['的情况，即存在数组下标
	if (tokens.keys != null) {
		processKeyedProperty(tokens, pv);
	}
	else {
		//非数组、容器类型的属性设置
		processLocalProperty(tokens, pv);
	}
}

private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
	//属性设置交由PropertyHandler完成
    //Spring会使用反射获取属性的getter和setter方法
	PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
	//如果不存在属性或者属性不是可写的
	if (ph == null || !ph.isWritable()) {
		//如果属性是Optional类型，则直接返回
		if (pv.isOptional()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Ignoring optional value for property '" + tokens.actualName +
						"' - property not found on bean class [" + getRootClass().getName() + "]");
			}
			return;
		}
		else {
			throw createNotWritablePropertyException(tokens.canonicalName);
		}
	}

	Object oldValue = null;
	try {
		Object originalValue = pv.getValue();
		Object valueToApply = originalValue;
		if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
			//如果已经完成了类型转换，直接使用就可以了
			if (pv.isConverted()) {
				valueToApply = pv.getConvertedValue();
			}
			else {
				//如果需要读取旧值(默认为false) && 值是可读的
				if (isExtractOldValueForEditor() && ph.isReadable()) {
					try {
						oldValue = ph.getValue();
					}
					catch (Exception ex) {
						if (ex instanceof PrivilegedActionException) {
							ex = ((PrivilegedActionException) ex).getException();
						}
						if (logger.isDebugEnabled()) {
							logger.debug("Could not read previous value of property '" +
									this.nestedPath + tokens.canonicalName + "'", ex);
						}
					}
				}
				valueToApply = convertForProperty(
						tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
			}
			pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
		}
		//使用PropertyHandler完成属性的注入
		ph.setValue(this.wrappedObject, valueToApply);
	}
	catch (TypeMismatchException ex) {
		throw ex;
	}
	catch (InvocationTargetException ex) {
		PropertyChangeEvent propertyChangeEvent = new PropertyChangeEvent(
				this.rootObject, this.nestedPath + tokens.canonicalName, oldValue, pv.getValue());
		if (ex.getTargetException() instanceof ClassCastException) {
			throw new TypeMismatchException(propertyChangeEvent, ph.getPropertyType(), ex.getTargetException());
		}
		else {
			Throwable cause = ex.getTargetException();
			if (cause instanceof UndeclaredThrowableException) {
				// May happen e.g. with Groovy-generated methods
				cause = cause.getCause();
			}
			throw new MethodInvocationException(propertyChangeEvent, cause);
		}
	}
	catch (Exception ex) {
		PropertyChangeEvent pce = new PropertyChangeEvent(
				this.rootObject, this.nestedPath + tokens.canonicalName, oldValue, pv.getValue());
		throw new MethodInvocationException(pce, ex);
	}
}
```
##### 4.3postProcessPropertyValues
此方法主要Autowired注解的属性注入，重点讲解AutowiredAnnotationBeanPostProcessor：
```java
public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
	//遍历获取带有自动注入注解的字段和方法，并且将字段信息、注解信息
	//注解包括：@Value、@Autowired、@Inject(javax.inject.Inject,如果能加载到这个类的话)
	InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
	try {
		//属性注入
		metadata.inject(bean, beanName, pvs);
	}
	catch (BeanCreationException ex) {
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
	}
	return pvs;
}
```
上面代码的步骤为：
1、寻找带有@Value、@Autowired、@Inject(javax.inject.Inject,如果能加载到这个类的话)注解的字段或方法，并且将字段或方法、bean名、bean类型封装起来；
2、完成属性注入；
```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
	// 先尝试从缓存中查找，cacheKey为beanName或类名
	String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
	InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
	//如果缓存未空 或者 需要解析的类与缓冲的类不同，则需要刷新缓存
	if (InjectionMetadata.needsRefresh(metadata, clazz)) {
		synchronized (this.injectionMetadataCache) {
			//双重判断，避免多线程问题，与单例模式实现类似
			metadata = this.injectionMetadataCache.get(cacheKey);
			if (InjectionMetadata.needsRefresh(metadata, clazz)) {
				if (metadata != null) {
					metadata.clear(pvs);
				}
				try {
					//解析后放入缓存
					metadata = buildAutowiringMetadata(clazz);
					this.injectionMetadataCache.put(cacheKey, metadata);
				}
				//catch略
			}
		}
	}
	return metadata;
}
```
因为对类的解析需要大量用到反射，所以为了尽力提高性能，Spring会将解析好的字段、方法信息保存到缓存中，上面的代码也在主要是对缓存进行管理，具体的解析工作在buildAutowiringMetadata方法中完成：
```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
	//存放解析好的属性信息
	LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<InjectionMetadata.InjectedElement>();
	Class<?> targetClass = clazz;

	//Spring会一直递归向上遍历bean的父类
	do {
		//存放当前解析的类中带有注解的属性
		final LinkedList<InjectionMetadata.InjectedElement> currElements =
				new LinkedList<InjectionMetadata.InjectedElement>();

		//doWithLocalFields方法中，其实就是获取class的所有字段，对每个字段调用FieldCallback方法
		//
		ReflectionUtils.doWithLocalFields(targetClass, new ReflectionUtils.FieldCallback() {
			@Override
			public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
				AnnotationAttributes ann = findAutowiredAnnotation(field);
				if (ann != null) {
					if (Modifier.isStatic(field.getModifiers())) {
						return;
					}
					//如果注解中存在required属性，获取属性的值，并保存起来
					//如果不存在required属性则默认为true
					boolean required = determineRequiredStatus(ann);
					//使用AutowiredFieldElement类型，将字段信息封装起来
					currElements.add(new AutowiredFieldElement(field, required));
				}
			}
		});
		//与Field逻辑相同，都是遍历所有声明的方法，判断是否带有特定的注解
		//如果存在注解标注的方法，并使用AutowiredMethodElement封装起来
		ReflectionUtils.doWithLocalMethods(targetClass, new ReflectionUtils.MethodCallback() {
			@Override
			public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
				if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
					return;
				}
				AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
				if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
					if (Modifier.isStatic(method.getModifiers())) {
						//log...
						return;
					}
					if (method.getParameterTypes().length == 0) {
						//log...
					}
					boolean required = determineRequiredStatus(ann);
					//与field不同的一点就是，AutowiredMethodElement会获取属性描述符，一起封装
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new AutowiredMethodElement(method, required, pd));
				}
			}
		});

		elements.addAll(0, currElements);
		targetClass = targetClass.getSuperclass();
	}
	//Spring会一直递归向上遍历bean的父类
	while (targetClass != null && targetClass != Object.class);
	//将封装有注入属性的列表一起封装到InjectionMetadata中
	return new InjectionMetadata(clazz, elements);
}
```
上面代码中，Spring先从当前类开始解析其字段和方法，并且会一直递归向上解析父类，把解析到的字段或方法信息封装到AutowiredFieldElement或AutowiredMethodElement中。
显示查找带有特定注解的属性，已经完成了，下一步就是属性注入，inject,主要内容就是遍历先前解析好的AutowiredFieldElement或AutowiredMethodElement，调用其inject方法。
