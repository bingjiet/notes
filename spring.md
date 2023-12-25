ApplicationContext 和 BeanFactory 

ApplicationContext 继承 BeanFactory
同时继承多个接口，每个接口扩展一个能力
- MessageSource 提供国际化功能，输出不同的语言数据
- ResourceLoader 获取资源
- EnvironmentCapable 获取环境信息
- ApplicationEventPublisher 分发事件，容器中所有的组件都可以用与处理事件，只要加上 @EventListener 注解即可对事件进行监听处理

### Bean 的生命周期

- 实例化构造方法
- 依赖注入
- 初始化
- 销毁

BeanPostProcessor 用于给 Bean 生命周期中的各个阶段提供扩展
- 实例化前后
- 初始化前后
- 销毁前

### BeanPostProcessor

- AutowiredAnnotationBeanPostProcessor 解析 @Autowired @Value 注解
- CommonAnnotationBeanPostProcessor    解析 @PostConstruct @Resource @PreDestroy 注解

### BeanFacotryPostProcessor

ConfigurationClassPostProcessor 解析 @ComponentScan @Bean @Import @ImportResource 等注解

#### 模拟 @ComponentScan 扫描
- 是否有 @ComponentScan 注解
- 获取扫描路径
- 获取这个路径下所有类的信息
- 查看是否包含 @Component以及派生的注解
- 生成 BeanDefinition 
- 注册到 BeanFactory 中

```java
  public class ComponentScanPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
      try {
        ComponentScan componentScan = AnnotationUtils.findAnnotation(Config.class, ComponentScan.class);
        // 用于读取类的元信息
        CachingMetadataReaderFactory cachingMetadataReaderFactory = new CachingMetadataReaderFactory();
        // bean name 生成器
        AnnotationBeanNameGenerator annotationBeanNameGenerator = new AnnotationBeanNameGenerator();
        if (componentScan != null) {
          // 遍历扫描路径，转成类路径
          for (String p : componentScan.basePackages()) {
            String path = "classpath*:" + p.replace(".", "/") + "/**/*.class";
            // 读取类路径下的资源
            Resource[] resources = new PathMatchingResourcePatternResolver().getResources(path);
            for (Resource resource : resources) {
              // 获取元数据
              MetadataReader metadataReader = cachingMetadataReaderFactory.getMetadataReader(resource);
              AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
              String componentName = Component.class.getName();
              // 查找这个类的注解信息是否包含 @Component 注解或者 @Component 派生注解，即 @Controller 这种间接引用 @Component 的注解
              if (annotationMetadata.hasAnnotation(componentName) || annotationMetadata.hasMetaAnnotation(componentName)) {
                // 生成 BeanDefinition
                AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder
                                    .genericBeanDefinition(metadataReader.getClassMetadata().getClassName())
                                    .getBeanDefinition();
                String beanName = annotationBeanNameGenerator.generateBeanName(beanDefinition, beanDefinitionRegistry);  
                // 注册到 BeanFactory 中
                beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
              }
            }
          }
        }
      } catch (Exception ex) {
        ex.printStackTrace();
      }
    }
  }
```

#### 模拟 @Bean 注解的解析

```java
  public class AtBeanPostProcessor implements BeanDefinitionRegistryPostProcessor {
  
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
      try {
        CachingMetadataReaderFactory metadataReaderFactory = new CachingMetadataReaderFactory();
        MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(new ClassPathResource("com/example/demo/a05/config/Config.class"));
        Set<MethodMetadata> annotatedMethods = metadataReader.getAnnotationMetadata().getAnnotatedMethods(Bean.class.getName());
        for (MethodMetadata annotatedMethod : annotatedMethods) {
          // 注解中还有很多属性，类似这样子解析
          String initMethod = annotatedMethod.getAnnotationAttributes(Bean.class.getName()).get("initMethod").toString();
          BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition();
          // 工厂方法，这个 Config 类类似一个工厂，通过这个方法名读取获取对象
          beanDefinitionBuilder.setFactoryMethodOnBean(annotatedMethod.getMethodName(), "config");
          // 自动装配
          beanDefinitionBuilder.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
          if (initMethod.length() > 0) {
            beanDefinitionBuilder.setInitMethodName(initMethod);
          }
          AbstractBeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();
          beanDefinitionRegistry.registerBeanDefinition(annotatedMethod.getMethodName(), beanDefinition);
        }
      } catch(Exception ex) {
        ex.printStackTrace();
      }
    }
  }
```

#### 模拟 @MapperScan 注解

```java
  public class MapperPostProcessor implements BeanDefinitionRegistryPostProcessor {
  
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
       try {
          AnnotationBeanNameGenerator nameGenerator = new AnnotationBeanNameGenerator();
          CachingMetadataReaderFactory metadataReaderFactory = new CachingMetadataReaderFactory();
          // 路径解析
          PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
          Resource[] resources = resolver.getResources("classpath*:com/example/demo/a05/mapper/**/*.class");
          for (Resource resource : resources) {
            MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(resource);
            ClassMetadata classMetadata = metadataReader.getClassMetadata();
            if (classMetadata.isInterface()) {
              // Mapper 实际上是与 MapperFactoryBean 做关联，实际生成的对象是 MapperFactoryBean 
              AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(MapperFactoryBean.class)
                      .addConstructorArgValue(classMetadata.getClassName())
                      .setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE)
                      .getBeanDefinition();
              AbstractBeanDefinition beanDefinition1 = BeanDefinitionBuilder.genericBeanDefinition(classMetadata.getClassName())
                      .getBeanDefinition();
              String name = nameGenerator.generateBeanName(beanDefinition1, beanDefinitionRegistry);
              beanDefinitionRegistry.registerBeanDefinition(name, beanDefinition);
            }
          }
       } catch (Exception ex) {
          ex.printStackTrace();
       }
    }
  }
```

各种 Aware 回调接口和初始化接口 InitializingBean
流程就是先调用 Aware 回调接口，再调用初始化方法

ApplicationContextAware 与 @Autowried 注入的区别，ApplicationContextAware 不需要后置处理器也会被 Spring 执行，而后者必须需要AutowiredAnnotationBeanPostProcessor 解析，其他同理

@Autowried 失效 
配置类加了一个 BeanFactoryPostProcsser 会引起失效，这是因为会导致优先处理这个类，将它生成出来，这样导致没有注入到其他的 BeanPostProcessor 
TODO

初始化方式
- @PostControut
- InitializingBean
- @Bean initMethod 属性 指定初始化方法
执行顺序  @PostControut，InitializingBean， @Bean

Bean 销毁方式
- @PreDestroy
- 实现接口
- @Bean注解指定 destroyMethod 指定

执行顺序 @PreDestroy 实现接口，@Bean

### Scop 作用域
- singleton
- prototype
- request 每次请求过来的时候会创建一个，请求结束会被销毁
- session 每个会话一个对象
- application 每个 web 应用一个对象

单例注入其他作用域的时候，会出现失效，这是因为单例在依赖注入的时候只被执行一次，因此会导致用的都是同一个对象
解决方法
- @Lazy 注解代理对象是同一个，但是当每次使用代理的对象的任意方法时候，由代理创建新的对象
- 在 SCOPE 中使用 proxyMode = ScopedProxyMode.TARGET_CLASS 也是创建代理
- 通过 ObjectFactory<T> 依赖注入解决，泛型就是所需要的对象
- 通过 ApplicationContext 去获取，这个可以通过 @Autowried 或者 实现 ApplicationContextAware 接口回调获取


1. 初始化
   1. 往容器中注册 RequestMappingHandlerMapping ，在 Bean 初始化的时候会解析所有的 Controller/RequestMapping 注解的类，将类中所有标有 @RequestMapping 注解的方法进行解析，RequestMapping 注解的信息封装成 RequestMappingInfo 对象
      方法封装成 HandleMethod ，然后注册到 mappingRegistry 中进行保存


在请求的时候，会根据当前的请求和RequestMappingInfo 进行匹配获取到 RequestMappingInfo ，然后到 mapRegistry 中获取 HandleMethod 
获取适配器 HandleMappingAdpater
