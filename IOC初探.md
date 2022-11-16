# 站在设计者的角度考虑设计IOC容器

<a>https://pdai.tech/md/spring/spring-x-framework-ioc-source-1.html</a> 

- 加载Bean的配置（比如xml配置）
  
  - 比如不同类型资源的加载，解析成生成统一Bean的定义

- 根据Bean的定义加载生成Bean的实例，并放置在Bean容器中
  
  - 比如Bean的依赖注入，Bean的嵌套，Bean存放（缓存）等

- 除了基础Bean外，还有常规针对企业级业务的特别Bean
  
  - 比如国际化Message，事件Event等生成特殊的类结构去支撑

- 对容器中的Bean提供统一的管理和调用
  
  - 比如用工厂模式管理，提供方法根据名字/类的类型等从容器中获取Bean

- ...
