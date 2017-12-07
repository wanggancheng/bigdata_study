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





