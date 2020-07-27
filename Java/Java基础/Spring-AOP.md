##Spring-AOP源码解读


首先 我们由一个简单的例子进入到AOP的模块 在此代码中，创建IOC容器，并从容器中获取一个BookSevice的接口的实现类。BookService实现类中
自动注入BookMapper。并且有一个切面织入到BookService实现类的每一个方法中

````java

public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("spring/ApplicationContext.xml");

        BookService bookService = context.getBean(BookService.class);

        bookService.toString();
    }

````

在这里，我们抛出几个问题

- 我们知道，面向切面的底层其实就是动态代理，在动态代理中，就少不了目标对象和代理对象，那么目标对象和代理对象是否都存在于容器中

- Spring是什么时候动态代理生成代理对象的

###问题验证

通过ApplicationContext创建的对象会在创建时直接将所有的非懒加载的bean实例化

````java

public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				//扫描所有类 并将bean信息添加到beanDefinitionMap中 执行用户自定义和Spring自带的BeanFactoryProcessor
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// 实例化所有非懒加载的单例bean
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
	}

````

spring通过处理每个bean的信息生成一个键值对，保存beanName与其信息的一一对应关系。

````java

public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}
		List<String> beanNames;
		synchronized (this.beanDefinitionMap) {
			// Iterate over a copy to allow for init methods which in turn register new bean definitions.
			// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
			beanNames = new ArrayList<String>(this.beanDefinitionNames); 
		}
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName); //在此位置通过bean的名称去创建一个bean
				}
			}
		}
	}

````

接着我们在创建bean的方法中 看到spring会根据bean的不同作用域去实例bean 由于bean的默认作用域便是单例

````java


// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

````


紧接着便是实例此单例对象

````java

// Initialize the bean instance.
		Object exposedObject = bean;
		try {
		    //对此对象进行填充 对bookService的实现类来说 便是注入bookMapper
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
			    //在此方法中进行动态代理生成代理对象
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

我们注意到initializeBean这个方法 在这个方法中 包含了bean的生命周期相关的方法

- BeanPostProcessorsBeforeInitialization

- init-method

- BeanPostProcessorsAfterInitialization 此方法便是spring动态代理生成代理对象的位置，所有spring是通过BeanPostProcessor的After
Initialization方法生成代理对象的

````java

protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
		    //执行这个bean的所有init方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}

		if (mbd == null || !mbd.isSynthetic()) {
		    //在此方法中生成代理对象
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}

````

BeanPostProcessor为AnnotationAwareAspectJAutoProxyCreator

最后返回的对象为jdk动态代理生成的对象，这是因为此实例有接口，否则便是cglib动态代理。如果熟悉IOC，其实在IOC容器中，就是有
很多的map集合去储存bean。其中便有一个单例对象的容器，在我们返回代理对象后便将其add进容器

````java

protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
		    //将此代理对象put进map中
			this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}

````

到这里 我们明白了几个问题
- 第一，在ApplicationContext创建后，便生成了代理对象并将其放进了IOC容器中，但是目标对象却并没有装进去。

- 第二，代理对象是在bean的生命周期中被创建的，通过BeanPostProcessor


其实，问题已经得到解决，在接下来的getBean方法中，并不会再对bean进行修改，那么在容器中存在的，只会是代理对象，而不是目标对象

### 附  BeanWrapper解释

   在源码分析中，我们发现在spring实例化bean时，使用的并不是bean，而是bean的一个包装类，beanWrapper。通过官方文档可知
beanWrapper中包含setPropertyValue， setPropertyValues，getPropertyValue，和getPropertyValues方法，其目的就是对bean
属性进行get/set。如果我们使用bean进行上述操作，当然也可。但是却并不显得通用。在beanWrapper中，还可支持嵌套属性获取或设置

````java

BeanWrapper company = new BeanWrapperImpl(new Company());
// setting the company name..
company.setPropertyValue("name", "Some Company Inc.");
// ... can also be done like this:
PropertyValue value = new PropertyValue("name", "Some Company Inc.");
company.setPropertyValue(value);

// ok, let's create the director and tie it to the company:
BeanWrapper jim = new BeanWrapperImpl(new Employee());
jim.setPropertyValue("name", "Jim Stravinsky");
company.setPropertyValue("managingDirector", jim.getWrappedInstance());

// retrieving the salary of the managingDirector through the company
Float salary = (Float) company.getPropertyValue("managingDirector.salary");

````
