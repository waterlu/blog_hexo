---
title: Mybatis动态代理原理
date: 2018-05-31 15:41:27
categories: 数据库
tags: [Mybatis, 动态代理]
toc: true
description: 本文解密为什么我们只需要定义一个Mapper接口就可以使用@Autowired注解注入DAO对象了。为了容易理解，我们先从单纯使用Mybatis开始，然后是Spring+Mybatis，最后才是SpringBoot+Mybatis。
comments: true
---

## Mybatis动态代理原理

我们将一步一步揭开Mybtis的实现原理。

### Mybatis

第一步，抛开Spring直接使用Mybatis。我们平时都是Spring+Mybatis一起使用的，由于Spring本身就比较复杂，再和Mybtis掺和到一块，不容易看清事情的真相。抛开String让我们更接近Mybatis。

#### 使用

揭秘之前先看看怎么使用。

首先，我们定义一个mybatis-config.xml，非常干净，里面只有数据源定义和Mapper文件定义

```java
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC" />
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://" />
                <property name="username" value="" />
                <property name="password" value="" />
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="mybatis/mapper/UserMapper.xml" />
    </mappers>
</configuration>
```

实体类User和UserMapper.xml我这里就不罗列了，UserMapper接口定义如下

```java
public interface UserMapper {
    User queryById(Long userId);
}
```

准备好以后，看看如何使用

```java
@RunWith(BlockJUnit4ClassRunner.class)
public class MybatisTests {
  @Test
  public void testMybatisConfig() {
    // 读取配置文件
    InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
    // 根据配置文件构造SessionFactory
    SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
    // 获取SqlSession
    SqlSession sqlSession = factory.openSession();
    // 从SqlSession中获取UserMaper类实例对象
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    User user = userMapper.queryById(100001L);
  }
}
```

虽然代码很简单，但其实很神奇。我们只声明了UserMapper接口，并没有提供它的实现，但我们可以从Session中获取UserMapper的实例对象并执行。

其核心是`sqlSession.getMapper()` ，看一下返回值是`org.apache.ibatis.binding.MapperProxy@e19bb76` ，说明返回了一个动态代理对象，这就是我们没有实现UserMapper接口就可以执行的原因。

#### 源码分析

下面从源码角度仔细分析Mybatis是如何实现上面效果的，以下以mybatis-3.4.5.jar源码为例。

##### MapperProxyFactory

我们从getMapper()方法开始入手，它调用了Configuration的getMapper()方法。

```java
package org.apache.ibatis.session.defaults;
public class DefaultSqlSession implements SqlSession {
  private final Configuration configuration;  
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }  
}
```

Configuration类如下，调用了MapperRegistry的getMapper()方法。

```java
package org.apache.ibatis.session;
public class Configuration {
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
}
```

MapperRegistry类如下，从名字中也能知道这是一个Mapper的注册表，getMapper()方法从Map中读取到工厂类，然后调用工厂类来创建代理类对象。

```java
package org.apache.ibatis.binding;
public class MapperRegistry {
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, 
  		MapperProxyFactory<?>>();
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 从Map中读取
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) 
      knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 创建并返回代理对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
}
```

如果调试一下可以发现Map中的确存在Key为`UserMapper.class` 的MapperProxyFactory对象，下面我们先不往下看MapperProxyFactory是如何创建MapperProxy对象的，先看看knownMappers是如何初始化的。

这就需要倒着找了，首先我们发现MapperRegistry中有addMapper()方法，那肯定是调用这个方法添加的。

```java
package org.apache.ibatis.binding;
public class MapperRegistry {  
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      boolean loadCompleted = false;
      // 主键是类对象，值是MapperProxyFactory对象
      knownMappers.put(type, new MapperProxyFactory<T>(type));
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      loadCompleted = true;
    }
  }
}
```

其实很容易想到，应该是Configuration类调用的addMapper()

```java
package org.apache.ibatis.session;
public class Configuration {
  public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
  }
}
```

再往上就不那么好找了，我们在Configuration::addMapper()方法上加个断点看一下调用堆栈，发现是XMLMapperBuilder类调用了Configuration的addMapper()方法

```java
package org.apache.ibatis.builder.xml;
public class XMLMapperBuilder extends BaseBuilder {
  // 继承自BaseBuilder
  protected final Configuration configuration;
  // ="resources/mybatis/mapper/UserMapper.xml"
  private final String resource;
  public void parse() {
    bindMapperForNamespace();
  }
  private void bindMapperForNamespace() {
    // ="cn.lu.mybatis.mapper.UserMapper"
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      // cn.lu.mybatis.mapper.UserMapper.class
      Class<?> boundType = Resources.classForName(namespace);
      if (boundType != null) {
        if (!configuration.hasMapper(boundType)) {
          configuration.addLoadedResource("namespace:" + namespace);
          configuration.addMapper(boundType);
        }
      }
    }
  }  
}
```

从名字就能看出XMLMapperBuilder应该是用来解析UserMapper.xml文件的，resource属性就对应文件名。我们再看`bindMapperForNamespace()` 方法，第一行显然在解析XML文件中的`<mapper namespace="">` ，接下来通过反射得到类对象boundType，最后调用configuration.addMapper()把它加入到MapperRegistry注册表中。我们再来回顾一下最后addMapper()时做了什么：

```java
public <T> void addMapper(Class<T> type) {
  knownMappers.put(type, new MapperProxyFactory<T>(type));
}
```

再看看谁调用了XMLMapperBuilder的parse()方法，一直往上找，最后找到build()方法。

```java
public SqlSessionFactory build(InputStream inputStream) {
  return build(inputStream, null, null);
}
```

还记得这句吗，这里调用了build()方法，并返回SqlSessionFactory。

```java
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);
```

这样就串起来，以上重点是根据Mapper类型获取MapperProxyFactory，我们再来正向过一遍。

- 构造SqlSessionFactory时，读取mybatis-config.xml文件，解析其中的`<mappers>` 标签，为每个`<mapper>` 创建XMLMapperBuilder对象并调用parse()方法进行解析；
- XMLMapperBuilder读取Mapper.xml文件，解析namespace得到对应的接口类；
- 最后以接口类型为主键，创建MapperProxyFactory对象为值，放入Mapper注册表中；
- getMapper()时从Mapper注册表中根据接口类型获取MapperProxyFactory，创建代理对象并返回。

##### Configuration

还有一个问题，那就是XMLMapperBuilder和DefaultSqlSession必须使用相同的Configuration，我们来验证一下。

SqlSessionFactoryBuilder类的build()方法创建了一个XMLConfigBuilder，我们知道这对应mybatis-config.xml文件；然后它调用XMLConfigBuilder的parse()方法，并用返回的Configuration创建SqlSessionFactory。

```java
package org.apache.ibatis.session;
public class SqlSessionFactoryBuilder {
  public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
  }
  public SqlSessionFactory build(InputStream inputStream, String environment, 
                                 Properties properties) {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());
  }
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
}
```

DefaultSqlSessionFactory类代码如下，构造函数中保存了Configuration，并在openSession()时使用相同的Configuration来创建DefaultSqlSession。

```java
package org.apache.ibatis.session.defaults;
public class DefaultSqlSessionFactory implements SqlSessionFactory {
  private final Configuration configuration;
  public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
  }
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }  
  private SqlSession openSessionFromDataSource(ExecutorType execType, 
					TransactionIsolationLevel level, boolean autoCommit) {
    // 这里省去了无关代码
    return new DefaultSqlSession(configuration, executor, autoCommit);
  }
}
```

DefaultSqlSession类代码如下，getMapper()时使用的Configuration就是构造时传入的Configuration。

```java
package org.apache.ibatis.session.defaults;
public class DefaultSqlSession implements SqlSession {
  private final Configuration configuration;
  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }  
}
```

由此我们确认SqlSession使用的Configuration不是自己创建的，是构造时SqlSessionFactoryBuilder传入的，那么Configuration到底是谁创建的呢？

```java
package org.apache.ibatis.builder.xml;
public class XMLConfigBuilder extends BaseBuilder {
  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
  }
  public Configuration parse() {
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }  
  private void parseConfiguration(XNode root) {
    mapperElement(root.evalNode("mappers"));
  }
  private void mapperElement(XNode parent) throws Exception {  
    // 这里简化了代码，更方便看懂
    XMLMapperBuilder mapperParser = new XMLMapperBuilder(configuration);
    mapperParser.parse();
  }
}
```

SqlSessionFactoryBuilder在构造XMLConfigBuilder时，XMLConfigBuilder类创建了Configuration对象，并在parse()方法中使用这个Configuration创建了XMLMapperBuilder，最后返回Configuration给SqlSessionFactoryBuilder，SqlSessionFactoryBuilder再用它创建SqlSession，所以XMLMapperBuilder和SqlSession使用的是一个Configuration。

我们再来复习一下：

- SqlSessionFactoryBuilder - 先解析mybatis-config.xml，然后创建并返回SqlSessionFactory
- XMLConfigBuilder - 解析mybatis-config.xml文件；
- XMLMapperBuilder - 解析UserMapper.xml文件；
- Configuration - 一个XMLConfigBuilder只有一个Configuration ，多个XMLMapperBuilder 共享XMLConfigBuilder的Configuration ；SqlSessionFactory也使用这个Configuration；
- SqlSessionFactory - 用来创建SqlSession；
- MapperRegistry - 以Map形式存储XMLMapperBuilder解析出来的Mapper接口
- MapperProxyFactory - 创建代理类对象的工厂

##### MapperProxy

前面弄清楚了MapperProxyFactory的来由，下面看看`mapperProxyFactory.newInstance(sqlSession);` 具体做了什么？

```java
package org.apache.ibatis.binding;
public class MapperProxyFactory<T> {
  private final Class<T> mapperInterface;
  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }  
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, 
                                                          methodCache);
    return newInstance(mapperProxy);
  }
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { 
      mapperInterface }, mapperProxy);
  }  
}
```

从前面我们知道MapperProxyFactory对象是这么创建的`new MapperProxyFactory<T>(type)` ，所以mapperInterface实际上就对应着UserMapper.class。我们再来复习一下JDK动态代理，Proxy.newInstance()方法将创建一个动态代理对象，这个动态代理对象实现了mapperInterface接口，也就是UserMapper定义的接口，对接口的访问将触发mapperProxy.invoke()方法的调用。

下面看MapperProxy的代码

```java
package org.apache.ibatis.binding;
public class MapperProxy<T> implements InvocationHandler, Serializable {
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;  
  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, 
                     Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }  
  private boolean isDefaultMethod(Method method) {
    return ((method.getModifiers()
        & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC)) == Modifier.PUBLIC)
        && method.getDeclaringClass().isInterface();
  }  
}
```

从前面我们分析出，Proxy.newInstance()方法帮我们创建了动态代理对象并返回，这个动态代理对象实现了UserMapper接口，当执行动态代理对象的方法时触发MapperProxy的invoke()方法。

我们知道，invoke()方法中，proxy是动态代理类对象，method是UserMapper接口反射出来的方法，args是参数，那么`method.invoke(this, args);` 将调用MapperProxy类的UserMapper接口方法，显然MapperProxy没有UserMapper接口，所以如果执行`method.invoke(this, args);` 一定会抛出异常。

所以，`if (Object.class.equals(method.getDeclaringClass()))` 这行代码的意思是如果我们声明的接口是Object类就可以执行，由于所有类都继承Object类，所以即使真的走到这行代码，也不会出错。当然，实际上我们不可能声明Mapper接口是Object类，所以`return method.invoke(this, args);` 永远不会被执行。

我们再看isDefaultMethod()方法，如果Mapper接口声明为`public abstract static xxx();` 就返回true，显然正常情况下都是返回false的。

综上，invoke()方法实际起作用的是最后两行代码。

看看cachedMapperMethod()干了啥？第一次执行时创建MapperMethod并缓存到Map中，以后从Map中读取出MapperMethod直接执行。

```java
package org.apache.ibatis.binding;
public class MapperProxy<T> implements InvocationHandler, Serializable {  
  private final Map<Method, MapperMethod> methodCache;
  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
}
```

##### MapperMethod

好吧，我们距离真相越来越近，再看MapperMethod代码。

```java
package org.apache.ibatis.binding;
public class MapperMethod {
  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    // type = "SELECT"
    // name = "cn.lu.mybatis.mapper.UserMapper.queryById"
    this.command = new SqlCommand(config, mapperInterface, method);
    // public abstract cn.lu.mybatis.entity.User 
    // cn.lu.mybatis.mapper.UserMapper.queryById(java.lang.Long);
    this.method = new MethodSignature(config, mapperInterface, method);
  }  
  public Object execute(SqlSession sqlSession, Object[] args) {
    // 这里涉及到很多种SQL语句，我们简化只看SELECT
    Object result;
    switch (command.getType()) {
      case SELECT:
        // 得到参数值
        Object param = method.convertArgsToSqlCommandParam(args);
        // 执行SQL语句
        result = sqlSession.selectOne(command.getName(), param);
        break;
        return result;
    }
  }
}
```

MapperMethod在构造时解析SQL的ID和类型，在execute()时调用SqlSession去执行SQL语句。

```java
package org.apache.ibatis.session.defaults;
public class DefaultSqlSession implements SqlSession {
  @Override
  public <T> T selectOne(String statement, Object parameter) {
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException();
    } else {
      return null;
    }
  }
  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  }
}
```

DefaultSqlSession最后根据SQL的ID得到SQL语句并执行，这里和动态代理无关，不是我们考虑的重点。

我们再来总结一下得到MapperProxyFatory后都做了什么：

- 每个Mapper接口都对应一个MapperProxyFactory，当需要执行Mapper接口中的方法时，每次由MapperProxyFactory创建一个MapperProxy的动态代理对象，每执行一次就创建一个；
- 所以最后执行的是MapperProxy的动态代理对象实现的接口方法，重定向到MapperProxy类的invoke()方法；
- invoke()方法的作用是根据方法名从Map找到需要执行的SQL语句的ID，最后交给SqlSession执行。

最后看范围

- MapperProxyFactory - 和UserMapper.class一一对应
- MapperProxy - 和getMapper()操作一一对应，每getMapper()一次，就生成一个
- methodCache - MapperProxyFactory 缓存了methodCache ，在各个MapperProxy 之间共享，所以methodCache 是和UserMapper.class一一对应的
- MapperMethod - 和UserMapper.class中的接口方法一一对应，由于使用了methodCache 缓存，所以一个方法只会创建一次MapperMethod对象；
- 也就是说：虽然每次getMapper()后返回的动态代理对象是不同的，但是它们最后都会执行MapperProxy的invoke()方法，各个MapperProxy都从methodCache中读取MapperMethod，而methodCache是在MapperProxyFactory 中创建的，所以实际上每个MapperMethod 只会在第一次被执行时创建。

##### 过程总结

- 首先创建SqlSessionFactory工厂，工厂加载并解析UserMapper.xml文件，UserMapper.xml文件中指定了对应的接口类UserMapper，创建一个Mapper注册表，存储到Configuration中；
- 执行UserMapper接口方法前先从SqlSessionFactory获取SqlSession，创建SqlSession时传入Configuration作为参数，这样SqlSession中就能找到Mapper注册表；
- 调用SqlSession的getMapper()方法，得到UserMapper接口的动态代理类对象（这个类是动态生成的）；
- 生成动态代理类对象的过程为：从Mapper注册表中找到MapperProxy工厂类，工厂类中使用JDK动态代理生成实现了UserMapper接口的动态代理类，InvocationHandler为MapperProxy；
- 执行UserMapper接口方法时，实际执行动态代理类实现的接口，转到MapperProxy的invoke()方法；
- MapperProxy的invoke()方法负责拼接SQL语句的ID=UserMapper接口类名+方法名，最后转交给SqlSession执行。

##### 原理总结

首先，我们认为MySQL最核心最基础的方法是这样的：

```java
InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession sqlSession = sqlSessionFactory.openSession();
User user = sqlSession.selectOne("cn.lu.mybatis.mapper.UserMapper.queryById", 100001L);
```

- openSession()创建一个数据库连接Connection，然后selectOne()在这个Connection上执行SQL语句；
- 构造SqlSessionFactory时读取Mapper.xml文件，创建一个保存SQL语句的Map，Map中的key是接口类名+xml文件中的id（例如：`cn.lu.mybatis.mapper.UserMapper.queryById`），value是SQL语句；
- selectOne()时传入key，得到SQL语句后执行。

以上是很好理解的，和动态代理没有关系。

后来，在使用过程中，大家觉得这样写太蛮烦，所以出现了下面这样的写法，实际上和上面的写法是等价的，最后执行的还是`sqlSession.selectOne("cn.lu.mybatis.mapper.UserMapper.queryById", 100001L);`

```java
User user = sqlSession.getMapper(UserMapper.class).queryById(100001L);
```

实际上整个动态代理的过程就是把接口类定义

```java
package cn.lu.mybatis.mapper;
public interface UserMapper {
  User queryById(Long userId);
}
```

翻译成下面字符串的过程。

```java
"cn.lu.mybatis.mapper.UserMapper.queryById"
```

### Spring + Mybatis

了解了Mybatis动态代理原理以后，我们加入Spring，同样先从使用开始。

#### 使用

第一种，定义每一个Mapper接口，每个Mapper接口都定义一个bean，对应MapperFactoryBean类，mapperInterface属性对应接口类

```xml
<beans>
  <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" />
  <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mybatis/mapper/*.xml"></property>
  </bean>  
  <bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" >
    <property name="mapperInterface" value="cn.lu.mybatis.mapper.UserMapper" />
    <property name="sqlSessionFactory" ref="sqlSessionFactory" />
  </bean>
</beans>
```

使用时直接注入

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring-mybatis.xml"})
public class SpringMybatisTests {
  @Autowired
  UserMapper userMapper;
  @Test
  public void test() {
    User user = userMapper.queryById(100001L);
  }
}
```

第二种，每个Mapper接口都定义一个Bean对象太麻烦了，更改为注解扫描的方式

- 使用MapperScannerConfigurer类自动扫描注解生成Mapper接口的Bean对象
- basePackage属性指定扫描哪些包
- annotationClass属性指定扫描哪个注解

```xml
<beans>
  <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" />
  <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mybatis/mapper/*.xml"></property>
  </bean>  
  <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!-- 自动扫描的位置 -->
    <property name="basePackage" value="cn.lu.mybatis.mapper"/>
    <!-- 自动扫描的注解 -->
    <property name="annotationClass" value="org.springframework.stereotype.Repository" />
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
  </bean>
</beans>
```

使用时还是直接注入，代码不变

#### 源码分析

第一种，注入userMapper时调用MapperFactoryBean的getObject()方法，在这个方法中执行getMapper()，Class信息从mapperInterface属性传入。

- Spring要做的是需要注入UserMapper接口的Bean对象时，调用getMapper()方法，只需要配置Bean的类型是FactoryBean，统一调用getMapper()方法返回动态代理对象即可。

```java
package org.mybatis.spring.mapper;
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }  
}
```

第二种，从入口MapperScannerConfigurer开始，它实现了BeanDefinitionRegistryPostProcessor接口，当容器加载完Bean以后开始处理，这里创建了Scanner并调用scan()方法

```java
package org.mybatis.spring.mapper;
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor {
  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage);
  }  
}
```

ClassPathBeanDefinitionScanner是ClassPathMapperScanner的基类，scan()里调用doScan()

```java
package org.springframework.context.annotation;
public class ClassPathBeanDefinitionScanner {
  public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    doScan(basePackages);
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
  }
}
```

子类ClassPathMapperScanner先执行基类的doScan()方法扫描Mapper接口，创建Bean对象并加入到Spring容器总，然后再通过processBeanDefinitions()修改Bean的class定义，这里是核心。

```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
    processBeanDefinitions(beanDefinitions);
    return beanDefinitions;
  }
}
```

基类doScan()时创建Mapper接口的Bean对象并加入到容器中

```java
package org.springframework.context.annotation;
public class ClassPathBeanDefinitionScanner {	
  protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
    // 扫描包可以定义多个目录
    for (String basePackage : basePackages) {
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      // 遍历每一个类
      for (BeanDefinition candidate : candidates) {
        ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(candidate);
        candidate.setScope(scopeMetadata.getScopeName());
        // beanName是Mapper接口类名，beanClass也是Mapper接口类
        // 这里只是扫描Mapper接口把Bean对象创建出来，后面再修改beanClass
        String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
        if (checkCandidate(beanName, candidate)) {
          BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
          beanDefinitions.add(definitionHolder);
          // 将新创建的Bean注册到Spring容器中
          registerBeanDefinition(definitionHolder, this.registry);
        }
      }
    }
    return beanDefinitions;
  }
  protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder,
                                        BeanDefinitionRegistry registry) {
    // 注册Bean
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
  }  
}
```

再回到ClassPathMapperScanner看后续处理：setBeanClass()方法设置MapperFactoryBean作为工厂Bean，这样注入Mapper接口时将调用MapperFactoryBean的getObject()方法，MapperFactoryBean里面再调用getMapper()就串联在一起了。

```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
  private MapperFactoryBean<?> mapperFactoryBean = new MapperFactoryBean<Object>();
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      // mapperInterface设置为UserMapper
      definition.getPropertyValues().add("mapperInterface", definition.getBeanClassName()); 
      // 将beanClass从UserMapper替换为MapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBean.getClass());
      definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
      definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
      definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    }
  }
}
```

总结一下：MapperScannerConfigurer对象实现BeanDefinitionRegistryPostProcessor接口，扫描basePackage包下的所有类文件，创建Bean对象加入到容器中，并且设置class属性为MapperFactoryBean。

### SpringBoot + Mybatis

#### 使用

我们先来回顾一下：

不使用Spring，我们直接硬编码从SqlSession获取Mapper接口的动态代理对象。

```java
UserMapper userMapper = sqlSessionFactory.openSession().getMapper(UserMapper.class);
```

使用了Spring以后，我们直接注入UserMapper

```java
@Autowired
UserMapper userMapper;
```

注入的前提是在XML文件中配置了MapperScannerConfigurer

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer" />
```

MapperScannerConfigurer负责扫描到每一个UserMapper对象，并用代码方式手工创建Bean对象加入到容器中，起到和如下xml配置相同的作用

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean" />
```

MapperFactoryBean最终调用getSqlSession().getMapper()，返回动态代理对象。

那么，在SpringBoot中如何使用Mybatis呢？

```java
@SpringBootApplication
@MapperScan("cn.lu.mybatis.mapper")
public class UserApplication {
}
```

对，只加一行@MapperScan注解就行了，这就是SpringBoot，约定大于配置。

#### 源码分析

从@MapperScan注解开始看

```java
package org.mybatis.spring.annotation;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Import({MapperScannerRegistrar.class})
public @interface MapperScan {
  String[] value() default {};
  String[] basePackages() default {};
  Class<?>[] basePackageClasses() default {};
  Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;
  Class<? extends Annotation> annotationClass() default Annotation.class;
  Class<?> markerInterface() default Class.class;
  Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;
}
```

这里Import了MapperScannerRegistrar配置类，重点一定在这里

```java
package org.mybatis.spring.annotation;
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, 
										   ResourceLoaderAware {
  private ResourceLoader resourceLoader;
  @Override
  public void setResourceLoader(ResourceLoader resourceLoader) {
    this.resourceLoader = resourceLoader;
  }
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
                                      BeanDefinitionRegistry registry) {
    // 读取MapperScan注解的属性
    String name = MapperScan.class.getName();
    Map<String, Object> map = importingClassMetadata.getAnnotationAttributes(name);
    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(map);
    // 创建scanner
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    if (resourceLoader != null) {
      scanner.setResourceLoader(resourceLoader);
    }
    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }    
    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs
      										.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }
    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));
    List<String> basePackages = new ArrayList<String>();
    for (String pkg : annoAttrs.getStringArray("value")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (String pkg : annoAttrs.getStringArray("basePackages")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (Class<?> clazz : annoAttrs.getClassArray("basePackageClasses")) {
      basePackages.add(ClassUtils.getPackageName(clazz));
    }
    scanner.registerFilters();
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }          
}
```

MapperScannerRegistrar和MapperScannerConfigurer一样使用ClassPathMapperScanner扫描注解，区别在于MapperScannerConfigurer根据xml配置创建scanner，MapperScannerRegistrar根据注解属性创建scanner，调用 scanner.doScan()方法之后的逻辑是一样的了。

虽然我们只在@MapperScan注解中设置了value一个属性，但其实它有很多属性，默认值起了作用。其中最重要的就是factoryBean了，默认值是MapperFactoryBean，看到这个一定觉得很亲切吧，终于绕回来了。

#### 总结

为什么SpringBoot中只需要一行@MapperScan就实现了自动注入Mapper接口的动态代理对象？

- @MapperScan注解定义了basePackages、markerInterface和factoryBean属性；
- @MapperScan引入MapperScannerRegistrar
- MapperScannerRegistrar创建ClassPathMapperScanner
- ClassPathMapperScanner扫描basePackages包下的所有markerInterface接口（默认值Class.class也就是所有类），生成Bean对象放入Spring容器，Bean的name为类名，class为factoryBean（默认值为MapperFactoryBean.class，也就是调用MapperFactoryBean.getObject()返回对象）
- 这样Spring容器中就有了接口类对象
- 自动注入时调用MapperFactoryBean.getObject()方法
- MapperFactoryBean调用SqlSession的getMapper()方法
- 通过Configuration、MapperRegistry ，最后调用MapperProxyFactory的newInstance()方法，执行Proxy.newInstance()，接口是Mapper接口，InvocationHandler是MapperProxy
- MapperProxy.invoke()方法拼接SQL语句ID，最终调用SqlSession的方法执行SQL语句。























