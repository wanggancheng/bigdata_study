# 从jasypt-spring-boot1.9版本中一个bug及新版本变化得到的收获

在使用jaspt-spring-boot-demo的DemoApplication中，无意发现jasypt-spring-boot的1.9版本的一个bug。

在DemoApplication中，自定义了一个StringEncryptor。代码如下：

```java
@Bean(name="encryptorBean")
    static public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("password");
        config.setAlgorithm("PBEWithMD5AndDES");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setStringOutputType("base64");
        encryptor.setConfig(config);
        return encryptor;
    }
```

在运行时出现了下列异常：

```
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'myService': Injection of autowired dependencies failed; nested exception is java.lang.IllegalArgumentException: Password cannot be set empty
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:372)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean(AbstractAutowireCapableBeanFactory.java:1264)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:553)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:483)
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:306)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:302)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:761)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:867)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:543)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:693)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:360)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:303)
	at org.springframework.boot.builder.SpringApplicationBuilder.run(SpringApplicationBuilder.java:134)
	at demo.DemoApplication.main(DemoApplication.java:54)
Caused by: java.lang.IllegalArgumentException: Password cannot be set empty
	at org.jasypt.commons.CommonUtils.validateIsTrue(CommonUtils.java:150)
	at org.jasypt.encryption.pbe.StandardPBEByteEncryptor.resolveConfigurationPassword(StandardPBEByteEncryptor.java:792)
	at org.jasypt.encryption.pbe.StandardPBEByteEncryptor.cloneAndInitializeEncryptor(StandardPBEByteEncryptor.java:486)
	at org.jasypt.encryption.pbe.StandardPBEStringEncryptor.cloneAndInitializeEncryptor(StandardPBEStringEncryptor.java:469)
	at org.jasypt.encryption.pbe.PooledPBEStringEncryptor.initialize(PooledPBEStringEncryptor.java:392)
	at org.jasypt.encryption.pbe.PooledPBEStringEncryptor.decrypt(PooledPBEStringEncryptor.java:489)
	at com.ulisesbocchio.jasyptspringboot.encryptor.LazyStringEncryptor.decrypt(LazyStringEncryptor.java:32)
	at org.jasypt.properties.PropertyValueEncryptionUtils.decrypt(PropertyValueEncryptionUtils.java:72)
	at com.ulisesbocchio.jasyptspringboot.EncryptablePropertySource.getProperty(EncryptablePropertySource.java:19)
	at com.ulisesbocchio.jasyptspringboot.wrapper.EncryptableMapPropertySourceWrapper.getProperty(EncryptableMapPropertySourceWrapper.java:28)
	at org.springframework.core.env.PropertySourcesPropertyResolver.getProperty(PropertySourcesPropertyResolver.java:80)
	at org.springframework.core.env.PropertySourcesPropertyResolver.getProperty(PropertySourcesPropertyResolver.java:61)
	at org.springframework.core.env.AbstractEnvironment.getProperty(AbstractEnvironment.java:530)
	at org.springframework.context.support.PropertySourcesPlaceholderConfigurer$1.getProperty(PropertySourcesPlaceholderConfigurer.java:132)
	at org.springframework.context.support.PropertySourcesPlaceholderConfigurer$1.getProperty(PropertySourcesPlaceholderConfigurer.java:129)
	at org.springframework.core.env.PropertySourcesPropertyResolver.getProperty(PropertySourcesPropertyResolver.java:80)
	at org.springframework.core.env.PropertySourcesPropertyResolver.getPropertyAsRawString(PropertySourcesPropertyResolver.java:71)
	at org.springframework.core.env.AbstractPropertyResolver$1.resolvePlaceholder(AbstractPropertyResolver.java:239)
	at org.springframework.util.PropertyPlaceholderHelper.parseStringValue(PropertyPlaceholderHelper.java:153)
	at org.springframework.util.PropertyPlaceholderHelper.replacePlaceholders(PropertyPlaceholderHelper.java:126)
	at org.springframework.core.env.AbstractPropertyResolver.doResolvePlaceholders(AbstractPropertyResolver.java:236)
	at org.springframework.core.env.AbstractPropertyResolver.resolveRequiredPlaceholders(AbstractPropertyResolver.java:210)
	at org.springframework.context.support.PropertySourcesPlaceholderConfigurer$2.resolveStringValue(PropertySourcesPlaceholderConfigurer.java:172)
	at org.springframework.beans.factory.support.AbstractBeanFactory.resolveEmbeddedValue(AbstractBeanFactory.java:831)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1086)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1066)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(AutowiredAnnotationBeanPostProcessor.java:585)
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:88)
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(AutowiredAnnotationBeanPostProcessor.java:366)
```

经过源码分析，可以与LazyStringEncryptor对象相关（从下图可以看出）

![](/assets/LazyStringEncryptor.png)



