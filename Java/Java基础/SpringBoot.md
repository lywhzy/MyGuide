## SpringBoot 原理

### 零配置原理

在ssm中我们有关于xml配置的文件总共有3个，web.xml，spring.xml,spring-mvc.xml。

其中最简单的便是spring.xml 在springboot中，可以完全使用注解方式去创建bean 以及指定扫包范围等 所以spring.xml可以被替代

接着 我们看web.xml 在web.xml中 我们配置了DispatcherServlet以及ContextLoaderListener，前者主要用于控制器的跳转，后者用于
开启spring-web层的容器，所以，我们只要实现了这两个功能，便可以替代web.xml

spring-mvc.xml 中主要配置了视图解析器，那么，springboot中已经默认配置好了，如果需要修改，需要在yml文件中更改即可

````java

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletCxt) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
        ac.register(AppConfig.class);
        ac.refresh();

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(ac);
        ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}

````


### 自动装配原理

Spring Boot启动的时候会通过@EnableAutoConfiguration注解找到META-INF/spring.factories配置文件中的所有自动配置类，并对其进行加载，而这些自动配置类都是以AutoConfiguration结尾来命名的，它实际上就是一个JavaConfig形式的Spring容器配置类，它能通过以Properties结尾命名的类中取得在全局配置文件中配置的属性如：server.port，而XxxxProperties类是通过@ConfigurationProperties注解与全局配置文件中对应的属性进行绑定的


### autowired注解原理

首先，我们找到为bean依赖注入的位置

````java

// Initialize the bean instance.
		Object exposedObject = bean;
		try {
		    //填充bean属性 ！！！！！！！！！！！！
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

````

我们在下面的代码中看到，spring为autowired的不同类型设置了不同的处理方法 byName，byType，当然我们选择的是默认的no。
在往下，我们看到，spring会遍历所有的BeanPostProcessor，通过调试，直到为AutowiredAnnotationBeanPostProcessor时，开始
为其注入属性


````java

if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
			    //遍历所有的BeanPostProcessor
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						//通过Processor为其注入属性
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

````


spring会在扫描的过程中扫描到的自动注入的信息（InjectionMetadata），在这时候起到了作用，spring根据bean的名称和class查找到
此信息对象，然后根据此对象对bean进行属性填充

````java

@Override
	public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
    
        // 该bean自动注入的数据信息
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass());
		try {
		    //注入信息
			metadata.inject(bean, beanName, pvs);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}

````


#### 结论

注解解析器：AutowiredAnnotationBeanPostProcessor

- Spring容器启动时，AutowiredAnnotationBeanPostProcessor被注册到容器；

- 扫描代码，如果带有@Autowired注解，则将依赖注入信息封装到InjectionMetadata中（见扫描过程）；

- 创建bean时（实例化对象和初始化），会调用各种BeanPostProcessor对bean初始化，AutowiredAnnotationBeanPostProcessor负责将相关的依赖注入进来；


