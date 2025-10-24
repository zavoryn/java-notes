# Spring

## Spring核心概念 ★★★★★

IOC, DI, AOP

1. Spring的核心优势(解耦、依赖注入、面向切面编程、事务管理等)
2. IOC(控制反转)的概念、作用及实现原理(容器负责对象创建与依赖管理)
3. DI(依赖注入)的概念、注入方式(构造器注入、setter 注入、字段注入)及优缺点
4. AOP(面向切面编程)的概念、作用(日志、事务、权限等横切逻辑)及核心术语(切面、通知、连接点、切点、目标对象等)
5. IOC与AOP的底层实现技术(反射、动态代理等)

## Spring IOC容器 ★★★☆☆ 一般不作问，了解下即可

1. Spring容器的核心接口(BeanFactory、ApplicationContext)及区别
2. ApplicationContext的常见实现类(ClassPathXmlApplicationContext、FileSystemXmlApplicationContext、AnnotationConfigApplicationContext)及区别
3. Bean的生命周期(实例化、属性注入、初始化、销毁)及关键回调方法(InitializingBean、DisposableBean、@PostConstruct、@PreDestroy)
4. Bean的作用域(singleton、prototype、request、session、globalSession)及使用场景，singleton与prototype的区别
5. Bean的配置方式(XML配置、注解配置：@Component、@Service、@Controller、@Repository、Java配置类@Configuration +@Bean)
6. Bean的依赖解决(@Autowired、@Resource、@Qualifier的区别与使用)
7. Spring容器初始化过程(资源定位、加载、注册、刷新)

## Spring AOP ★★★☆☆ 一般不作问，了解下即可

1. AOP的实现原理(JDK动态代理vs CGLIB代理的区别、适用场景)
2. Spring AOP的通知类型(前置通知、后置通知、返回通知、异常通知、环绕通知)及执行顺序
3. 切入点表达式(execution、within、this、target、args)的写法与使用场景
4. 切面的优先级控制(@Order注解)
5. Spring AOP与AspectJ的区别与联系
6. AOP的实际应用场景(事务管理、日志记录、接口权限校验、性能监控)

## 核心问题与优化 ★★★★★ 问的比较多

1. Spring循环依赖的产生原因及解决方式(三级缓存：singletonObjects、earlySingletonObjects、singletonFactories)
2. @Autowired与@Resource的区别(来源、注入方式、匹配规则)
3. Spring Bean是线程安全的吗?如何保证线程安全?(无状态Bean、ThreadLocal、同步机制)
4. Spring性能优化手段(Bean懒加载、连接池配置、缓存机制、减少反射开销)
