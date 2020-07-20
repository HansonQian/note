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