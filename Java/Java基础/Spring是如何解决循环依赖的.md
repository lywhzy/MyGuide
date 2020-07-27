## Spring循环依赖


循环依赖其实就是bean之间的依赖形成一个环，比如a依赖于b，而b又依赖于a。


### 源码解析

Spring创建单例bean的代码如下,首先第一次调用getSingleton方法，尝试从单例池中获取bean


````java

// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

````

这里我们可以发现Spring提供了一个earlySingletonObjects池去解决循环依赖的问题，它会存储正在被循环依赖的对象的引用

````java

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		
		//如果单例池中没有此对象，且该对象正在被其他对象所创建 那么先返回一个对象引用供其使用
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		
		//如果对象不存在且该对象没有在其他对象的生命周期中，那么返回该对象
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}

````

紧接着到达第二次getSingleton方法中，此方法是真正的创建单例对象的方法，在其中会存在beforeSingletonCreation(beanName)方法，此方法的作用便是
将该对象的状态进行改变，表明此对象正在被创建。


### 原理

Spring三级缓存 SingletonFactories bean执行构造器后 便将此bean加入到三级缓存中，全部实例化后，在加入到SingletonObjects中，
所以通过b在发现依赖于a后，会先从缓存里找，而不是直接创建。