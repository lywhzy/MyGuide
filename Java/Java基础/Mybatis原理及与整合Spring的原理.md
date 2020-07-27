## Mybatis原理

首先，mapper在项目中是以接口的方式存在的。这里我们先不讨论整合spring的情况。在只使用mybatis的情况下，我们需要先得到sqlSession对象，
紧接着获取mapper对象。

````java

SqlSession session = sqlSessionFactory.openSession();

BookMapper bookMapper = session.getMapper(BookMapper.class);
````

问题来了，既然我们创建的mapper是接口，而并没有指定实现过这个接口，那这里获得的bookMapper如何去访问其方法呢。换言之，既然能够访问其方法，那么
一定是一个实体类。经过调试，我们发现bookMapper实际上是一个代理类，那么其中就一定用到了动态代理。


### 源码分析

深入getMapper的源码发现，实际上他仅仅是通过jdk动态代理返回了一个代理类,jdk动态代理需要三个参数，类加载器，目标类实现的接口，以及InvocationHandler
其实，前两个参数只要知道class便可以得到，所以我们发现getMapper方法传递的参数是class

````java

protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

````

接下来我们注意到的是mapperProxy这个对象，其实现了InvocationHandler接口，并给其传递了sqlSession参数,下面便是mapper代理类的具体实现

````java

@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
       //如果是Object的方法或者是默认的方法，直接执行
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //通过method对象将其sql进行解析
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //将方法参数传入其中，执行sql语句
    return mapperMethod.execute(sqlSession, args);
  }

````


## Mybatis整合Spring原理

在Spring中，我们发现只需要一个mapperScan注解便可扫描到该路径下的所有mapper，并将其注入到Spring容器中。
那么，我们首先来看看spring是如何通过mapperScan注解扫描到mapper的.在这个注解中，我们发现了其Import了一个类MapperScannerRegistrar
这个类实现了ImportBeanDefinitionRegistrar，其作用是，可以通过代码向Bean定义的容器中加入Bean定义。这里解释一下，bean定义其实就是生成bean
时所用的数据

````java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(tk.mybatis.spring.annotation.MapperScannerRegistrar.class)
public @interface MapperScan {

````



````java

scanner.registerFilters();

//通过mapperScan上的指定的mapper位置进行扫描
scanner.doScan(StringUtils.toStringArray(basePackages));

````


````java

public Set<BeanDefinitionHolder> doScan(String... basePackages) {
        //扫描路径下的mapper接口，并将其转化为bean的定义信息
        Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

        if (beanDefinitions.isEmpty()) {
            logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
        } else {
            //执行这些bean定义
            processBeanDefinitions(beanDefinitions);
        }

        return beanDefinitions;
    }

````


到这里，如果是Spring扫描的bean，那么通过这些bean定义，可以轻易的将其放入到Spring容器中，但是从mybatis的原理上看，每个mapper接口的实现都是
通过动态代理生成的，那么，如何将已经创建的对象放入Spring容器中。这里 我们将用到factoryBean。在下面的代码中，getObject将会返回已经创建的
对象，也就是mapper的代理类

````java

public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

    //用来保存生成mapper的类型
    private Class<T> mapperInterface;

    private boolean addToConfig = true;

    private MapperHelper mapperHelper;

    public MapperFactoryBean() {
        //intentionally empty
    }

    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected void checkDaoConfig() {
        super.checkDaoConfig();

        notNull(this.mapperInterface, "Property 'mapperInterface' is required");

        Configuration configuration = getSqlSession().getConfiguration();
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Exception e) {
                logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
                throw new IllegalArgumentException(e);
            } finally {
                ErrorContext.instance().reset();
            }
        }
        //直接针对接口处理通用接口方法对应的 MappedStatement 是安全的，通用方法不会出现 IncompleteElementException 的情况
        if (configuration.hasMapper(this.mapperInterface) && mapperHelper != null && mapperHelper.isExtendCommonMapper(this.mapperInterface)) {
            mapperHelper.processConfiguration(getSqlSession().getConfiguration(), this.mapperInterface);
        }
    }

    /**
     * Return the mapper interface of the MyBatis mapper
     *
     * @return class of the interface
     */
    public Class<T> getMapperInterface() {
        return mapperInterface;
    }

    /**
     * Sets the mapper interface of the MyBatis mapper
     *
     * @param mapperInterface class of the interface
     */
    public void setMapperInterface(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }

    //------------- mutators --------------

    /**
     * {@inheritDoc}
     */
    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    /**
     * Return the flag for addition into MyBatis config.
     *
     * @return true if the mapper will be added to MyBatis in the case it is not already
     * registered.
     */
    public boolean isAddToConfig() {
        return addToConfig;
    }

    /**
     * If addToConfig is false the mapper will not be added to MyBatis. This means
     * it must have been included in mybatis-config.xml.
     * <p/>
     * If it is true, the mapper will be added to MyBatis in the case it is not already
     * registered.
     * <p/>
     * By default addToCofig is true.
     *
     * @param addToConfig
     */
    public void setAddToConfig(boolean addToConfig) {
        this.addToConfig = addToConfig;
    }

    /**
     * 设置通用 Mapper 配置
     *
     * @param mapperHelper
     */
    public void setMapperHelper(MapperHelper mapperHelper) {
        this.mapperHelper = mapperHelper;
    }
    /**
     * {@inheritDoc}
     */
    @Override
    public boolean isSingleton() {
        return true;
    }
}

````

那么到这里，这个factoryBean是如何注入到Spring容器中的呢，其实，就是通过之前的MapperScannerRegistrar，他将factoryBean这个bean的定义import到
bean定义的容器中。这样factoryBean便可以注入到容器中。


````java

 // the mapper interface is the original class of the bean
            // but, the actual class of the bean is MapperFactoryBean
            
            //将beanFactory的构造器参数新增到bean定义中
            definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
            definition.setBeanClass(this.mapperFactoryBean.getClass());

````    

此时，我们mapper的代理对象就放入到ioc容器中，从而可以进行依赖注入



### 附

    factoryBean和bean的区别，factoryBean其实是两个bean 一个是他本身，另一个则是他getObject得到的一个bean

