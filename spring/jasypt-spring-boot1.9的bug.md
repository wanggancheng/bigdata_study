# 从jasypt-spring-boot1.9版本中一个bug及新版本变化得到的收获

## 1.bug的发现及分析过程

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

似乎与DEFAULT_LAZY\_ENCRYPTOR\_FACTORY相关。_

它实际就是一个Java的Function。

```java
 public static final Function<Environment, StringEncryptor> DEFAULT_LAZY_ENCRYPTOR_FACTORY = e -> {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword(getRequiredProperty(e, "jasypt.encryptor.password"));
        config.setAlgorithm(getProperty(e, "jasypt.encryptor.algorithm", "PBEWithMD5AndDES"));
        config.setKeyObtentionIterations(getProperty(e, "jasypt.encryptor.keyObtentionIterations", "1000"));
        config.setPoolSize(getProperty(e, "jasypt.encryptor.poolSize", "1"));
        config.setProviderName(getProperty(e, "jasypt.encryptor.providerName", "SunJCE"));
        config.setSaltGeneratorClassName(getProperty(e, "jasypt.encryptor.saltGeneratorClassname", "org.jasypt.salt.RandomSaltGenerator"));
        config.setStringOutputType(getProperty(e, "jasypt.encryptor.stringOutputType", "base64"));
        encryptor.setConfig(config);
        return encryptor;
    };
```

其中要求Environment提供一个“jasypt.encryptor.password"的属性值，也就是运行异常提到的password。这说明自定义的StringEncryptor未发挥作用。

重新分析日志，找到一个分析线索：

```java
Overriding bean definition for bean 'encryptorBean' with a different definition: replacing [Root bean: class [demo.DemoApplication]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=stringEncryptor; initMethodName=null; destroyMethodName=(inferred); defined in demo.DemoApplication] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=com.ulisesbocchio.jasyptspringboot.configuration.StringEncryptorConfiguration; factoryMethodName=stringEncryptor; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [com/ulisesbocchio/jasyptspringboot/configuration/StringEncryptorConfiguration.class]]
Registering new name 'encryptorBean' for Bean definition with placeholder name: ${jasypt.encryptor.bean:jasyptStringEncryptor}
String Encryptor custom Bean not found with name 'encryptorBean'. Initializing String Encryptor based on properties with name 'encryptorBean'
```

```
   从日志可以看出，名为"encryptorBean"的bean definition被替换过，可能是被前面提到的替换过。
```

只好重新学习源码。

找到了定义的地方并且与一个BeanFactoryProcessor相关。

```java
 @Conditional(OnMissingEncryptorBean.class)
    @Bean
    public static BeanNamePlaceholderRegistryPostProcessor beanNamePlaceholderRegistryPostProcessor(Environment environment) {
        return new BeanNamePlaceholderRegistryPostProcessor(environment);
    }

    @Conditional(OnMissingEncryptorBean.class)
    @Bean(name = ENCRYPTOR_BEAN_PLACEHOLDER)
    public StringEncryptor stringEncryptor(Environment environment) {
        String encryptorBeanName = environment.resolveRequiredPlaceholders(ENCRYPTOR_BEAN_PLACEHOLDER);
        LOG.info("String Encryptor custom Bean not found with name '{}'. Initializing String Encryptor based on properties with name '{}'",
                 encryptorBeanName, encryptorBeanName);
        return new LazyStringEncryptor(DEFAULT_LAZY_ENCRYPTOR_FACTORY, environment);
    }
```

这两个Bean生效与OnMissingEncryptorBean相关。

```java
 /**
     * Condition that checks whether the StringEncryptor specified by placeholder: {@link #ENCRYPTOR_BEAN_PLACEHOLDER} exists.
     * ConditionalOnMissingBean does not support placeholder resolution.
     */
    private static class OnMissingEncryptorBean implements ConfigurationCondition {

        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
            return !context.getBeanFactory().containsBean(context.getEnvironment().resolveRequiredPlaceholders(ENCRYPTOR_BEAN_PLACEHOLDER));
        }

        @Override
        public ConfigurationPhase getConfigurationPhase() {
            return ConfigurationPhase.REGISTER_BEAN;
        }
    }
```

经过断点调试分析，match方法永远返回true。BeanNamePlaceholderRegistryPostProcessor的主要处理如下：

```java
 @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
            DefaultListableBeanFactory bf = (DefaultListableBeanFactory) registry;
            Stream.of(bf.getBeanDefinitionNames())
                //Look for beans with placeholders name format: '${placeholder}' or '${placeholder:defaultValue}'
                .filter(name -> name.matches("\\$\\{[\\w\\.-]+(?>:[\\w\\.-]+)?\\}"))
                .forEach(placeholder -> {
                    String actualName = environment.resolveRequiredPlaceholders(placeholder);
                    BeanDefinition bd = bf.getBeanDefinition(placeholder);
                    bf.removeBeanDefinition(placeholder);
                    bf.registerBeanDefinition(actualName, bd);
                    LOG.debug("Registering new name '{}' for Bean definition with placeholder name: {}", actualName, placeholder);
                });
        }
```

就是在postProcessBeanDefinitionRegistry中把名为”encryptorBen"的自定义beanDefinition替换掉了。





