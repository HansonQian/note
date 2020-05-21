## Spring IoC源码分析

### Spring IoC容器初始化主体流程

#### Spring IoC容器体系

​		IoC容器是Spring的核心模块，是抽象了对象管理、依赖关系管理的框架解决方案。Spring提供了很多容器，其中

`BeanFactory`是顶级容器，不能被实例化，他定义了所有IOC容器必须遵从的一套原则，具体的容器实现可以增加额外的功能，例如常用的`ApplicationContext`，其下更具体的实现如`ClasspathXmlApplication`包含了解析xml等一系列的内容，`AnnotationConfigApplication`则是包含了注解解析等一系列的内容。Spring IoC容器继承体系，可供使用者根据需求灵活选取，不必使用功能大而全的。

​		`BeanFactory`顶级接口方法栈如下

![image-20200509144002738](Spring%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20200509144002738.png)

​		`BeanFactory`继承体系

![image-20200509144348375](Spring%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20200509144348375.png)

​		通过其接口设计，可以看出平时使用的`ApplicationContext`接口除了继承`BeanFactory`接口，还继承了`ResourceLoader`、`MessageSource`等接口，这样就能够提供更丰富的功能了。

#### Spring Bean生命周期分析



#### Spring Bean初始化主流程

​		由Bean的生命周期分析可知，Spring Ioc容器初始化的关键环节就在` AbstractApplicationContext`的`refresh()`方法中，

查看`refresh`方法来分析IOC容器创建的主体流程。

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 第一步： 刷新前预处理
        prepareRefresh();
        /*
		 * 第二步：
		 * 获取BeanFactory,默认实现是DefaultListableBeanFactory
		 * 加载BeanDefition 并注册到 BeanDefitionRegistry
         */ 
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 第三步：BeanFactory预准备工作（BeanFactory进行一些设置,例如设置context的类加载器）
        prepareBeanFactory(beanFactory);
        try {
            // 第四步：BeanFactory准备工作完成后进行的后置处理工作
            postProcessBeanFactory(beanFactory);
            // 第五步：实例化并调用实现了BeanFactoryPostProcessor的Bean
            invokeBeanFactoryPostProcessors(beanFactory);
            // 第六步：注册BeanPostProcessor,在创建Bean的前后执行
            registerBeanPostProcessors(beanFactory);
            // 第七步：初始化MessageSource组件（做国际化功能;消息绑定;消息解析）
            initMessageSource();
            // 第八步：初始化事件派发器
            initApplicationEventMulticaster();
            // 第九步：⼦类重写这个⽅法，在容器刷新的时候可以⾃定义逻辑
            onRefresh();
            // 第十步：注册应用监听器即注册实现了ApplicationListener接口的Bean
            registerListeners();
            /*
			 * 第十一步：
			 * 初始化创建非懒加载方式的单例Bean示例（未设置属性）
			 * 填充属性
			 * 初始化方法调用（例如afterPropertiesSet方法，init-method方法）
			 * 调用BeanPostProcessor对实例Bean进行后置处理
			 */
            finishBeanFactoryInitialization(beanFactory);
            /*
			 * 第十二步：
			 * 完成context的刷新。主要是调用LifecycleProcessor的onRefresh方法,并且
			 * 发布事件（ContextRefreshEvent）
			 */
            finishRefresh();
        }catch (BeansException ex) {
            ...
        }
    }
```

### BeanFactory创建流程

#### 获取BeanFactory子流程

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

#### BeanDefinition加载解析及注册子流程

- 该流程涉及到如下几个关键步骤
  - Resource定位
    - 指对BeanDefinition的资源定位过程。简单的说就是怎么找到定义了JavaBean的XML文件（applicationContext.xml）
  - BeanDefinition载入
    - 指将用户定义的JavaBean转为IoC容器需要的数据结构，这个数据结构就是BeanDefinition
  - 注册BeanDefinition到IoC容器

- 过程分析

  - 子流程入口在`AbstractRefreshableApplicationContext`的`refreshBeanFactory`方法中

  ```java
  protected final void refreshBeanFactory() throws BeansException {
      // 1、判断使用已经存在 Bean Factory
      if (hasBeanFactory()) {
          // 1.1、销毁beans
          destroyBeans();
          // 1.2、关闭BeanFactory
          closeBeanFactory();
      }
      try {
          // 2、实例化 DefaultListableBeanFactory
          DefaultListableBeanFactory beanFactory = createBeanFactory();
          // 3、设置序列化id
          beanFactory.setSerializationId(getId());
          // 4、自定义bean工厂的一些属性（是否覆盖、是否允许循环依赖）
          customizeBeanFactory(beanFactory);
          // 5、加载应用中的BeanDefinition
          loadBeanDefinitions(beanFactory);
          synchronized (this.beanFactoryMonitor) {
              // 6、赋值当前的 bean factory
              this.beanFactory = beanFactory;
          }
      }catch (IOException ex) {
          ...
      }
  }
  ```

  - 依次调用多个类的`loadBeanDefinitions`方法，`AbstractXmlApplicationContext`->`AbstractBeanDefinitionReader`->`XmlBeanDefinitionReader`一直执行到``XmlBeanDefinitionReader``的`loadBeanDefinitions`方法

  ```java
  protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
      throws BeanDefinitionStoreException {
      try {
          // 1、读取XML中的信息，将信息保存为Dcoument对象
          Document doc = doLoadDocument(inputSource, resource);
          // 2、解析Document对象，封装成BeanDefinition对象并进行注册
          return registerBeanDefinitions(doc, resource);
      }catch (BeanDefinitionStoreException ex) {
          ...
      }
  }
  ```

  - 分析`registerBeanDefinitions`方法

  ```java
  public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
      BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
      // 1、获取已有BeanDefinition数量
      int countBefore = getRegistry().getBeanDefinitionCount();
      // 2、注册BeanDefinition
      documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
      // 3、返回新注册的BeanDefinition数量
      return getRegistry().getBeanDefinitionCount() - countBefore;
  }
  ```

  - 分析`createReaderContext`与`registerBeanDefinitions`方法

    - `createReaderContext`方法，该方法主要完成了`DefaultNamespaceHandlerResolver`的初始化工作

      ```java
      public XmlReaderContext createReaderContext(Resource resource) {
          return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
                                      this.sourceExtractor, this, getNamespaceHandlerResolver());
      }
      public NamespaceHandlerResolver getNamespaceHandlerResolver() {
          if (this.namespaceHandlerResolver == null) {
              this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
          }
          return this.namespaceHandlerResolver;
      }
      protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
          return new DefaultNamespaceHandlerResolver(getResourceLoader().getClassLoader());
      }
      ```

    - `registerBeanDefinitions`方法，该方法位于`DefaultBeanDefinitionDocumentReader`类中

      ```java
      public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
          this.readerContext = readerContext;
          Element root = doc.getDocumentElement();
          doRegisterBeanDefinitions(root);
      }
      ```

      进入`doRegisterBeanDefinitions`方法

      ```java
  protected void doRegisterBeanDefinitions(Element root) {
          BeanDefinitionParserDelegate parent = this.delegate;
          this.delegate = createDelegate(getReaderContext(), root, parent);
      
          if (this.delegate.isDefaultNamespace(root)) {
              String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
              if (StringUtils.hasText(profileSpec)) {
                  String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                      profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                  if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                      return;
                  }
              }
          }
          // 钩子方法
          preProcessXml(root);
          // 解析BeanDefinition
          parseBeanDefinitions(root, this.delegate);
          // 钩子方法
          postProcessXml(root);
          this.delegate = parent;
      }
      ```
    
      进入`parseBeanDefinitions`方法，该方法用于解析文档中根级别的元素：**import、alias、bean**
    
      ```java
      protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
          if (delegate.isDefaultNamespace(root)) {
              NodeList nl = root.getChildNodes();
              for (int i = 0; i < nl.getLength(); i++) {
                  Node node = nl.item(i);
                  if (node instanceof Element) {
                      Element ele = (Element) node;
                      if (delegate.isDefaultNamespace(ele)) {
                          // 解析默认的标签元素
                          parseDefaultElement(ele, delegate);
                      }else {
                          // 解析自定义标签元素
                          delegate.parseCustomElement(ele);
                      }
                  }
              }
          }else {
              delegate.parseCustomElement(root);
          }
      }
      ```
    
      进入`parseDefaultElement`方法
    
      ```java
      private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
          // import元素处理
          if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
              importBeanDefinitionResource(ele);
          }
          // alias元素处理
          else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
              processAliasRegistration(ele);
          } 
          // bean元素处理
          else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
              processBeanDefinition(ele, delegate);// 处理Bean元素
          }
          // 嵌套beans元素处理
          else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
              doRegisterBeanDefinitions(ele);
          }
      }
      ```
    
      进入`processBeanDefinition`方法
    
      ```java
      protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
          // 将Bean元素解析成BeanDefinition，但此时使用BeanDefinitionHolder对象进行了包装
          BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
          if (bdHolder != null) {
              // 如果有自定义标签，则处理自定义标签
              bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
              try {
                  // 完成BeanDefinition的注册
                  BeanDefinitionReaderUtils.registerBeanDefinition(
                      bdHolder, getReaderContext().getRegistry());
              } catch (BeanDefinitionStoreException ex) {
                ....
              }
              // 发出响应事件，通知相关的监听器，这个bean已经加载完成
              getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
          }
      }
      ```
    
      到这边BeanDefinition加载解析以及注册就结束了，其实继续跟踪可以发现所谓的注册，其实就是把在XML定义的Bean信息转为BeanDefinition对象之后存入Map中，BeanFactory是以Map的数据结构组织管理这些BeanDefinition的
    
      ```java
      // Still in startup registration phase
      this.beanDefinitionMap.put(beanName, beanDefinition);
      this.beanDefinitionNames.add(beanName);
      this.manualSingletonNames.remove(beanName);
      ```
    
      可以在`DefaultListableBeanFactory`查看**beanDefinitionMap**的定义
    
      ```java
      private final Map<String, BeanDefinition> beanDefinitionMap =
          					new ConcurrentHashMap<String, BeanDefinition>(256);
      ```
    
    - 时序图
    
    ```sequence
    title: BeanDefinition加载解析及注册流程
    participant AbstractApplicationContext as A
    participant AbstractRefreshableApplicationContext as B
    participant AbstractXmlApplicationContext as C
    participant AbstractBeanDefinitionReader as D
    participant BeanDefinitionReader as E
    participant XmlBeanDefinitionReader as F
    participant BeanDefinitionDocumentReader as H
    
    A->A:1、obtainFreshBeanFactory()
    A->B:2、refreshBeanFactory()
    B->B:3、createBeanFactory()
    A->B:4、getBeanFactory()
    A->B:5、loadBeanDefinitions()
    B->C:6、loadBeanDefinitions()
    C->C:7
    
    
    ```
    
    