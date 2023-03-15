# 什么是 SpringBoot 自动装配
我们现在提到自动装配的时候，一般会和 Spring Boot 联系在一起。但是，实际上 Spring Framework 早就实现了这个功能。Spring Boot 只是在其基础上，通过 SPI 的方式，做了进一步优化。
> SpringBoot 定义了一套接口规范，这套规范规定：SpringBoot 在启动时会扫描外部引用 jar 包中的META-INF/spring.factories文件，将文件中配置的类型信息加载到 Spring 容器（此处涉及到 JVM 类加载机制与 Spring 的容器知识），并执行类中定义的各种操作。对于外部 jar 来说，只需要按照 SpringBoot 定义的标准，就能将自己的功能装置进 SpringBoot。

## 如何实现
`@SpringBootApplication`注解点进去可以看到是以下三个注解的集合

- `@EnableAutoConfiguration`：启动自动装配机制
- `@Configuration`：允许在上下文中注册额外的 bean 或导入其他配置类
- `@ComponentScan`：扫描被`@Component`注解的 bean

重点在`@EnableAutoConfiguration`：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class) // 加载自动装配类 AutoConfigurationImportSelector
public @interface EnableAutoConfiguration {

    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};

}
```
核心的功能实现在`AutoConfigurationImportSelector`类
```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {}
```
`AutoConfigurationImportSelector`实现了`DeferredImportSelector`接口，也就实现了`selectImports`方法，此方法主要用于**获取所有符合条件的类的全限定类名，并将这些类加载到 IOC 容器之中**
```java
	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```
此方法的调用链如下：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1678873290468-abe495f1-e332-4470-a7a6-76f27359dfcd.png#averageHue=%23f4f4f4&clientId=uf1c89f39-ce4e-4&from=paste&height=469&id=u968f90b8&name=image.png&originHeight=1032&originWidth=2014&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=177213&status=done&style=none&taskId=u5cab1f67-90ca-4883-9d01-a96f18c8b54&title=&width=915.454525612603)
最后一步会从 META-INF/spring.factories 中加载自动配置类

