### SpringBoot关于AOP的代码分析

我们知道SpringBoot自动化配置的关键在于注解`org.springframework.boot.autoconfigure.EnableAutoConfiguration`，各组件
的可插拔特性也在于这个注解的实现，其中AOP功能的实现类为`AopAutoConfiguration`。
