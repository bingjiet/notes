ApplicationContext 和 BeanFactory 

ApplicationContext 继承 BeanFactory
同时继承多个接口，每个接口扩展一个能力
- MessageSource 提供国际化功能，输出不同的语言数据
- ResourceLoader 获取资源
- EnvironmentCapable 获取环境信息
- ApplicationEventPublisher 分发事件，容器中所有的组件都可以用与处理事件，只要加上 @EventListener 注解即可对事件进行监听处理
