# 1、Spring IOC 源码分析

## 1.1、Spring IoC 容器初始化主体流程

IoC 容器是Spring的核心模块，是抽象了对象管理、依赖关系管理的框架解决方案。Spring 提供了很多的容器，其中 `BeanFactory` 是顶层容器（根容器），不能被实例化，它定义了所有 IoC 容器必须遵从的一套原则，具体的容器实现可以增加额外的功能，比如我们常用到的 `ApplicationContext`，其下更具体的实现如 `ClassPathXmlApplicationContext` 包含了解析 xml 等一系列的内容；`AnnotationConfigApplicationContext` 则是包含了注解解析等一系列的内容。

### 1.1.1、BeanFactory 容器源码

```java
public interface BeanFactory {
	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);
}
```

### 1.1.2、BeanFactory 容器继承体系

![BeanFacotry 容器继承体系](05-Spring%20IOC%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20200720104131004.png)

1. ApplicationContext 继承了 ListableBeanFactory，这个 Listable 的意思就是，通过这个接口，我们可以获取多个 Bean，最顶层 BeanFactory 接口的方法都是获取单个 Bean 的
2. ApplicationContext 继承了 HierarchicalBeanFactory，Hierarchical 单词本身已经能说明问题了，也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为父子关系
3. AutowireCapableBeanFactory 这个名字中的 Autowire 大家都非常熟悉，它就是用来自动装配 Bean 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory() 就知道了

## 1.2、BeanFactory 创建流程

### 1.2.1、调试程序

- java

```java
public class AnalysisBean {
    public String hi() {
        return "hello 我是Analysis Bean";
    }
}
```

- xml 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="analysisBean" class="com.github.bean.AnalysisBean"/>
</beans>
```

- 测试程序

```java
public class ExampleApplication {
    public static void main(String[] args) throws Exception {
        System.out.println("准备初始化容器");
        ApplicationContext act = new ClassPathXmlApplicationContext("applicationContext.xml");
        System.out.println("容器初始化完成");
        AnalysisBean bean = act.getBean(AnalysisBean.class);
        System.out.println(bean.hi());
        System.out.println("现在开始关闭容器");
        Method method = act.getClass().getMethod("close", null);
        method.invoke(act, null);
    }
}
```

### 1.2.2、启动过程分析

以 `ClassPathXmlApplicationContext` 进行分析

#### 1.2.2.1、构造函数

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    private Resource[] configResources;
    
    // 如果已经有 ApplicationContext 并需要配置成父子关系，那么调用这个构造方法
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, (ApplicationContext)null);
    }
    //...
    public ClassPathXmlApplicationContext(String[] configLocations,boolean refresh, 
                                          ApplicationContext parent) throws BeansException {
        super(parent);
        // 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
        this.setConfigLocations(configLocations);
        if (refresh) {
            this.refresh(); //核心方法
        }

    }    
}
```

#### 1.2.2.2、refresh 方法

该方法在 `AbstractApplicationContext` 中

```java
public void refresh() throws BeansException, IllegalStateException {
    // 1、加锁，保证在同一时刻只能有一个线程处理
    synchronized(this.startupShutdownMonitor) {
        // 2、准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
        this.prepareRefresh();
        /* 3、
         * 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
         * 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
         * 注册也只是将这些信息都保存到了注册中心(类似是一个Map<beanName,beanDefinition>)
         */
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        // 4、设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
        this.prepareBeanFactory(beanFactory);
        try {
            /* 5、
             * 这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，那么
             * 在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法
             * 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，
             * 但是都还没有初始化 具体的子类可以在这步的时候添加一些特殊的 
             * BeanFactoryPostProcessor 的实现类或做点什么事
             */
            this.postProcessBeanFactory(beanFactory);
            // 6、调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory()方法
            this.invokeBeanFactoryPostProcessors(beanFactory);
            /* 7、
             * 注册 BeanPostProcessor 的实现类 ,此接口两个方法: 
             * postProcessBeforeInitialization 和postProcessAfterInitialization
             * 两个方法分别在 Bean 初始化之前和初始化之后得到执行
             * 到这里 Bean 还没初始化
             */
            this.registerBeanPostProcessors(beanFactory);
            // 8、初始化当前 ApplicationContext 的 MessageSource
            this.initMessageSource();
            // 9、初始化当前 ApplicationContext 的事件广播器
            this.initApplicationEventMulticaster();
            // 10、钩子方法，子类以在这里初始化一些特殊的Bean(在初始化 singleton beans 之前)
            this.onRefresh();
            // 11、注册事件监听器，监听器需要实现 ApplicationListener 接口
            this.registerListeners();
            // 12、初始化所有的 singleton beans(lazy-init 的除外)
            this.finishBeanFactoryInitialization(beanFactory);
            // 13、最后，广播事件，ApplicationContext 初始化完成
            this.finishRefresh();
        } catch (BeansException var9) {
            // 14、销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
            this.destroyBeans();
            // 15、重置 ‘active’ 标记
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            this.resetCommonCaches();
        }
    }
}
```

#### 1.2.2.3、创建 Bean 容器前准备工作

```java
protected void prepareRefresh() {
    // 记录时间
    this.startupDate = System.currentTimeMillis();
    // 将 closed 属性设置为 false
    this.closed.set(false);
    // 将 active 属性设置为 true
    this.active.set(true);
    initPropertySources();
    // 校验 xml 配置文件
    getEnvironment().validateRequiredProperties();.
    this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

#### 1.2.2.4、创建 Bean容器，加载并注册Bean

##### 1.2.2.4.1、创建Bean 容器

这一步将会初始化 BeanFactory、加载 Bean、注册 Bean 等等

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
    refreshBeanFactory();
    // 返回刚刚创建的 BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    return beanFactory;
}
```

查看 `AbstractRefreshableApplicationContext` 的 `refreshBeanFactory()` 方法

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    /*
     * 如果 ApplicationContext 中已经加载过 BeanFactory 了，销毁所有 Bean，关闭 BeanFactory
     * 注意，应用中 BeanFactory 本来就是可以多个的，这里可不是说应用全局是否有 BeanFactory，而是当前
     * ApplicationContext 是否有 BeanFactory
     */
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 初始化一个 DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 用于 BeanFactory 的序列化
        beanFactory.setSerializationId(getId());
        // 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
        customizeBeanFactory(beanFactory);
        // 加载 Bean 到 BeanFactory 中
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

继续查看`AbstractRefreshableApplicationContext` 的 `customizeBeanFactory()` 方法

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    // 是否允许 Bean 定义覆盖
    if (this.allowBeanDefinitionOverriding != null) {
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // 是否允许 Bean 间的循环依赖
    if (this.allowCircularReferences != null) {
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

BeanDefinition 的覆盖问题大家也许会碰到，就是在配置文件中定义 bean 时使用了相同的 id 或 name，默认情况下，allowBeanDefinitionOverriding 属性为 null，如果在同一配置文件中重复了，会抛错，但是如果不是同一配置文件中，会发生覆盖。

循环引用也很好理解：A 依赖 B，而 B 依赖 A。或 A 依赖 B，B 依赖 C，而 C 依赖 A。

默认情况下，Spring 允许循环依赖，当然如果你在 A 的构造方法中依赖 B，在 B 的构造方法中依赖 A 是不行的。

##### 1.2.2.4.2、Bean 加载

 `loadBeanDefinitions(beanFactory)` 方法，这个方法将根据配置，加载各个 Bean，然后放到 BeanFactory 中。

查看`AbstractXmlApplicationContext` 的 `loadBeanDefinitions(beanFactory)` 方法

```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 给这个 beanFactory 实例化一个 XmlBeanDefinitionReader
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    // 核心方法
    loadBeanDefinitions(beanDefinitionReader);
}
```

查看 `loadBeanDefinitions` 方法

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```

两个分支都会进入 `AbstractBeanDefinitionReader` 的 `loadBeanDefinitions(resources)` 方法

```java
@Override
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int counter = 0;
    // 每一个文件都是一个 resource
    for (Resource resource : resources) {
        counter += loadBeanDefinitions(resource); // 核心方法
    }
    return counter;
}
```

查看 `XmlBeanDefinitionReader` 的 `loadBeanDefinitions(resource)` 方法

```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}
```

继续查看 `XmlBeanDefinitionReader` 的 `loadBeanDefinitions(encodedResource)` 方法

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    // 用一个 ThreadLocal 来存放所有的配置文件资源
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<EncodedResource>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 核心部分
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}
```

继续查看 `XmlBeanDefinitionReader` 的 `doLoadBeanDefinitions(inputSource,resource)` 方法

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
    try {
        // 转换为 Document 对象
        Document doc = doLoadDocument(inputSource, resource);
        // 核心方法
        return registerBeanDefinitions(doc, resource); 
    }
	// 省略异常信息
}
```

继续查看 `XmlBeanDefinitionReader` 的 `registerBeanDefinitions(doc,resource)` 方法

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 核心方法
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

查看`DefaultBeanDefinitionDocumentReader` 的 `registerBeanDefinitions(doc,readerContext)` 方法

```java
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    // 转为dom 树
    Element root = doc.getDocumentElement();
    // 核心方法
    doRegisterBeanDefinitions(root);
}
```

至此，已经将 resource 转为一颗完整的 DOM 树，注意，这里指的是其中一个配置文件，不是所有的，前面有个 for 循环的。

继续查看`DefaultBeanDefinitionDocumentReader` 的 `doRegisterBeanDefinitions(root)` 方法

```java
protected void doRegisterBeanDefinitions(Element root) {
    // BeanDefinitionParserDelegate 核心的解析器类 
    BeanDefinitionParserDelegate parent = this.delegate;
    
    this.delegate = createDelegate(getReaderContext(), root, parent);

    if (this.delegate.isDefaultNamespace(root)) {
        // 获取根节点属性 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        // 如果当前环境配置的 profile 不包含此 profile，那就直接 return 了，不对此 <beans /> 解析
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                return;
            }
        }
    }
    preProcessXml(root); // 钩子方法
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);// 钩子方法
    this.delegate = parent;
}
```

继续查看 `DefaultBeanDefinitionDocumentReader` 的 `parseBeanDefinitions(root, this.delegate)` 方法

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    // 默认解析四个标签:<import />、<alias />、<bean /> 和 <beans />
                    parseDefaultElement(ele, delegate);
                }else {
                    // 解析自定义标签
                    delegate.parseCustomElement(ele);
                }
            }
        }
    } else {
        // 解析自定义标签
        delegate.parseCustomElement(root);
    }
}
```

继续查看 `DefaultBeanDefinitionDocumentReader`  的`parseDefaultElement(ele, delegate)` 方法

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);//处理 import 标签
    }
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);// 处理 alias 标签
    }
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);// 处理 bean 标签
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        doRegisterBeanDefinitions(ele); // 处理嵌套的 <beans /> 标签
    }
}
```

<a name="divtop">解析Bean核心处理</a>：以处理`bean` 标签为例，继续查看 `DefaultBeanDefinitionDocumentReader` 的 ` processBeanDefinition(ele, delegate)`方法。

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 核心方法，将 <bean> 标签内容 封装成 一个 BeanDefinitionHolder
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 如果有自定义属性的话，进行相应的解析
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 注册Bean
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                                     bdHolder.getBeanName() + "'", ele, ex);
        }
        // 注册完成后，发送事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

查看`BeanDefinitionParserDelegate` 的 `parseBeanDefinitionElement(ele)`方法

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, 
                                                       BeanDefinition containingBean) {
    String id = ele.getAttribute(ID_ATTRIBUTE);
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    List<String> aliases = new ArrayList<String>();
    // 将 name 属性的定义按照 ”逗号、分号、空格“ 切分，形成一个别名列表数组
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }
    String beanName = id;
    // 如果没有指定id, 那么用别名列表的第一个名字作为beanName
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
    }
    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }
    // 核心方法,根据 <bean ...>...</bean> 中的配置创建 BeanDefinition，然后把配置中的信息都设置到实例中
    AbstractBeanDefinition beanDefinition = 
        parseBeanDefinitionElement(ele, beanName, containingBean);
    // <bean /> 标签就算解析结束，一个 BeanDefinition 就形成了
    if (beanDefinition != null) {
        // 如果都没有设置 id 和 name，那么此时的 beanName 就会为 null，进入下面这块代码产生
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                        beanDefinition, this.readerContext.getRegistry(), true);
                }else {
                    // 当id 和 name 没有定义时 进行处理
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null && 
                        beanName.startsWith(beanClassName) && 
                        beanName.length() > beanClassName.length() &&
                        !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        // 把 beanClassName 设置为 Bean 的别名
                        aliases.add(beanClassName);
                    }
                }
            } catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        // 返回 BeanDefinitionHolder
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    return null;
}
```

继续查看  `DefaultBeanDefinitionDocumentReader`  的 `parseBeanDefinitionElement(ele, beanName, containingBean)` 方法

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {
    this.parseState.push(new BeanEntry(beanName));
    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    try {
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }
        // 创建 BeanDefinition，然后设置类信息
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);
		// 设置 BeanDefinition 的一堆属性，这些属性定义在 AbstractBeanDefinition 中
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

       /*
        * 下面的一堆是解析 <bean>......</bean> 内部的子元素，
        * 解析出来以后的信息都放到 bd 的属性中
        */
        parseMetaElements(ele, bd);// 解析 <meta />
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());// 解析 <lookup-method />
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());// 解析 <replaced-method />

        parseConstructorArgElements(ele, bd);// 解析 <constructor-arg />
        parsePropertyElements(ele, bd);// 解析 <property />
        parseQualifierElements(ele, bd);// 解析 <qualifier />

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));
		// 返回 BeanDefinition
        return bd;
    }
    catch (...
           // 异常信息省略
    finally {
        this.parseState.pop();
    }
    return null;
}
```

至此 BeanDefinition 创建完成，然后注册到 IOC容器中。查看[解析Bean入口](#divtop)

##### 1.2.2.4.3、注册Bean

查看 `DefaultBeanDefinitionDocumentReader` 的 `processBeanDefinition` 方法

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 注册Bean 实例
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                                     bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

跟进 `BeanDefinitionReaderUtils` 的 `registerBeanDefinition` 方法

```java
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

    // 注册这个 Bean
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 如果还有别名的话，也要根据别名全部注册一遍，不然根据别名就会找不到 Bean 了
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            // alias -> beanName 保存它们的别名信息，这个很简单，用一个 map 保存一下就可以了，
            // 获取的时候，会先将 alias 转换为 beanName，然后再查找
            registry.registerAlias(beanName, alias);
        }
    }
}
```

跟进 `DefaultListableBeanFactory` 的 `registerBeanDefinition` 方法

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }catch (BeanDefinitionValidationException ex) {
            //"Validation of bean definition failed"
        }
    }

    //  处理是否允许 覆盖 BeanDefinition 配置
    BeanDefinition oldBeanDefinition;

    // 所有的 Bean 注册后会放入这个 beanDefinitionMap 中
    oldBeanDefinition = this.beanDefinitionMap.get(beanName);

    // 处理重复名称的 Bean 定义的情况
    if (oldBeanDefinition != null) {
        if (!isAllowBeanDefinitionOverriding()) {
            // 如果不允许覆盖的话，抛异常
        } else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
			// 用框架定义的 Bean 覆盖用户自定义的 Bean
        }else if (!beanDefinition.equals(oldBeanDefinition)) {
			// 用新的 Bean 覆盖旧的 Bean
        }else {
			// 用同等的 Bean 覆盖旧的 Bean，这里指的是 equals 方法返回 true 的 Bean
        }
        // 覆盖
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }else {
      // 判断是否已经有其他的 Bean 开始初始化了.
      // 注意，"注册Bean" 这个动作结束，Bean 依然还没有初始化，我们后面会有大篇幅说初始化过程，
      // 在 Spring 容器启动的最后，会 预初始化 所有的 singleton beans
        if (hasBeanCreationStarted()) {
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
             // 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
            this.beanDefinitionMap.put(beanName, beanDefinition);
             // 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
            this.beanDefinitionNames.add(beanName);
             // 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
             // 注意这里是 remove 方法，到这里的 Bean 当然不是手动注册的
             // 手动指的是通过调用以下方法注册的 bean ：
             //   registerSingleton(String beanName, Object singletonObject)
             // Spring 会在后面"手动"注册一些 Bean，如 "environment"、"systemProperties" 等 bean
             // 也可以在运行时注册 Bean 到容器中的            
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (oldBeanDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

### 1.2.3、BeanFactory 容器实例化完成

时序图

```sequence
title: 获取BeanFactory子流程
participant AbstractApplicationContext as A
participant AbstractRefreshableApplicationContext as B

A->A:1、obtainFreshBeanFactory()
A->B:2、refreshBeanFactory()
B->B:3、createBeanFactory()
A->B:4、getBeanFactory()
B-->>A:5、DefaultListableBeanFactory
```

## 1.3、BeanFacroty 预处理

查看`AbstractApplicationContext#prepareBeanFactory`

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置 BeanFactory 的类加载器,设置为加载当前 ApplicationContext 类的类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    // 设置 Bean 表达式解析器
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 设置 设置属性编辑器注册器
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加一个 BeanPostProcessor，这个 processor 比较简单：
    // 实现了 Aware 接口的 beans 在初始化的时候，这个 processor 负责回调，
    // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
    // 注意：它不仅仅回调 ApplicationContextAware，
    // 还会负责回调 EnvironmentAware、ResourceLoaderAware 等，看下源码就清楚了
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    
    // 如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，
    // Spring 会通过其他方式来处理这些依赖。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   /**
    * 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
    * 之前我们说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行。
    * ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
    * 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
    * 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean
    */
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 这个 BeanPostProcessor 也很简单，在 bean 实例化后，如果是 ApplicationListener 的子类，
    // 那么将其添加到 listener 列表中，可以理解成：注册 事件监听器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 这里涉及到特殊的 bean，名为：loadTimeWeaver，是 AspectJ 的概念，指的是在运行期进行织入
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 如果没有定义 "environment" 这个 bean，那么 Spring 会 "手动" 注册一个
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    // 如果没有定义 "systemProperties" 这个 bean，那么 Spring 会 "手动" 注册一个
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    // 如果没有定义 "systemEnvironment" 这个 bean，那么 Spring 会 "手动" 注册一个
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

## 1.4、初始化所有的单例 Bean

查看`AbstractApplicationContext#finishBeanFactoryInitialization`，这里会负责初始化所有的 singleton beans。

到目前为止，应该说 BeanFactory 已经创建完成，并且所有的实现了 BeanFactoryPostProcessor 接口的 Bean 都已经初始化并且其中的 postProcessBeanFactory(factory) 方法已经得到回调执行了。而且 Spring 已经“手动”注册了一些特殊的 Bean，如 `environment`、`systemProperties` 等。

剩下的就是初始化 singleton beans 了，我们知道它们是单例的，如果没有设置懒加载，那么 Spring 会在接下来初始化所有的 singleton beans。

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化名字为 conversionService 的 Bean，用于参数绑定与转换
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }
    // 处理内置的值解析器，默认为String类型的值解析器
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
            @Override
            public String resolveStringValue(String strVal) {
                return getEnvironment().resolvePlaceholders(strVal);
            }
        });
    }

    // 初始化 LoadTimeWeaverAware 类型的 Bean
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // 开始初始化
    beanFactory.preInstantiateSingletons();
}
```

跟进 `DefaultListableBeanFactory#preInstantiateSingletons` 方法

```java
public void preInstantiateSingletons() throws BeansException {

    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // 触发 所有非懒加载的单例Bean 的实例化
    for (String beanName : beanNames) {
        // 合并父 Bean 中的配置，bean 继承概念
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 非抽象、非懒加载的 singletons。如果配置了 'abstract = true'，那是不需要初始化的
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 处理 FactoryBean
            if (isFactoryBean(beanName)) {
                // FactoryBean 的话，在 beanName 前面加上 ‘&’ 符号。再调用 getBean 方法
                final FactoryBean<?> factory = 
                    (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit;
                // 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现
                if (System.getSecurityManager() != null && 
                    factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(
                        new PrivilegedAction<Boolean>() {
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
                // 对于普通的 Bean，只要调用 getBean(beanName) 这个方法就可以进行初始化了
                getBean(beanName);
            }
        }
    }
	// 到这里说明所有的非懒加载的 singleton beans 已经完成了初始化
    // 如果我们定义的 bean 是实现了 SmartInitializingSingleton 接口的，那么在这里得到回调.
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

接下来，我们就进入到 getBean(beanName) 方法了，这个方法我们经常用来从 BeanFactory 中获取一个 Bean，而初始化的过程也封装到了这个方法里。

跟进`AbstractBeanFactory#getBean(java.lang.String)` 方法

```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
protected <T> T doGetBean(
    final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
    throws BeansException {

    final String beanName = transformedBeanName(name);
    Object bean;

    // 检查下是不是已经创建过了
    Object sharedInstance = getSingleton(beanName);
    
    if (sharedInstance != null && args == null) {
        // 如果是普通 Bean 的话，直接返回 sharedInstance，
        // 如果是 FactoryBean 的话，返回它创建的那个实例对象
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        if (isPrototypeCurrentlyInCreation(beanName)) {
         // 创建过了此 beanName 的 prototype 类型的 bean，那么抛异常，
         // 往往是因为陷入了循环引用
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 检查一下这个 BeanDefinition 在容器中是否存在
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 如果当前容器不存在这个 BeanDefinition，试试父容器中有没有
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // 返回父容器的查询结果.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            } else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        if (!typeCheckOnly) {
            // typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中。
            markBeanAsCreated(beanName);
        }
        /*
         * 稍稍总结一下：
         * 到这里的话，要准备创建 Bean 了，对于 singleton 的 Bean 来说，容器中还没创建过此 Bean；
         * 对于 prototype 的 Bean 来说，本来就是要创建一个新的 Bean。
         */
        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

           // 先初始化依赖的所有 Bean，这个很好理解。
           // 注意，这里的依赖指的是 depends-on 中定义的依赖
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 检查是不是有循环依赖，这里的循环依赖和我们前面说的循环依赖又不一样，
                    // 这里肯定是不允许出现的，不然要乱套了
                    if (isDependent(beanName, dep)) {
                        // 循环依赖异常信息
                    }
                    // 注册一下依赖关系
                    registerDependentBean(dep, beanName);
                    // 先初始化被依赖项
                    getBean(dep);
                }
            }

            // 如果是 singleton scope 的，创建 singleton 的实例
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            // 执行创建 Bean
                            return createBean(beanName, mbd, args);
                        } catch (BeansException ex) {
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
			// 如果是 prototype scope 的，创建 prototype 的实例
            else if (mbd.isPrototype()) {
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    // 执行创建 Bean
                    prototypeInstance = createBean(beanName, mbd, args);
                } finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
			// 如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    // 异常信息
                }
                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                // 执行创建 Bean
                                return createBean(beanName, mbd, args);
                            } finally {
                                afterPrototypeCreation(beanName);
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }catch (IllegalStateException ex) {
					// 异常信息;
                }
            }
        } catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // 最后，检查一下类型对不对，不对的话就抛异常，对的话就返回了
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }catch (TypeMismatchException ex) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

查看`createBean()`方法定义

```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException;
```

第三个参数 args 数组代表创建实例需要的参数，不就是给构造方法用的参数，或者是工厂 Bean 的参数嘛，不过要注意，在我们的初始化阶段，args 是 null。

抽象类，交由子类完成，跟进 `AbstractAutowireCapableBeanFactory#createBean(...))`方法，`AbstractAutowireCapableBeanFactory` 用于处理 `@Autowired` 注解

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    RootBeanDefinition mbdToUse = mbd;
    // 确保 BeanDefinition 中的 Class 被加载
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // 准备方法覆写，这里又涉及到一个概念：MethodOverrides，
    // 它来自于 bean 定义中的 <lookup-method /> 和 <replaced-method />
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
       // 异常信息
    }

    try {
        // 让 InstantiationAwareBeanPostProcessor 在这一步有机会返回代理
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }catch (Throwable ex) {
        // 异常信息
    }
	// 创建 bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    
    return beanInstance;
}
```

跟进`doCreateBean(beanName, mbdToUse, args)`方法

```java
protected Object doCreateBean(final String beanName, 
                              final RootBeanDefinition mbd, final Object[] args)
    throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 核心方法
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
               // 异常信息
            }
            mbd.postProcessed = true;
        }
    }

    // 解决循环依赖的问题
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 负责属性装配，核心方法
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 处理 bean 初始化完成后的各种回调 核心方法
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    catch (Throwable ex) {
       // 异常信息
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    // 异常信息
                }
            }
        }
    }

    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
       // 异常信息
    }

    return exposedObject;
}
```

