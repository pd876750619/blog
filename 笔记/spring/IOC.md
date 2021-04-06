# Ioc

### Resource定位

将BeanDefinition的资源定位由ResourceLoader通过统一对的Resource接口完成

1. loadBeanDefintion方法采用了模板模式，将实现委托给子类完成，而算法逻辑框架已经由父类定制好，调用的形式大多为子类->父类初始化方法->子类特殊定制处理

### Refersh方法，初始化上下文的提纲



1. 初始化上下文（todo 修改）

   1. 设置有效性
   2. 加载特殊占位符的property
   3. 校验是否获取了所有必需property

2. 重启BeanFactory

3. ```java
   @Override
   protected final void refreshBeanFactory() throws BeansException {
      if (hasBeanFactory()) {
         //如果已经有容器存在，需要把现有容器销毁和关闭，保证在refresh后使用的是新建立的ioc容器
         destroyBeans();
         closeBeanFactory();
      }
      try {
         //创建IOC容器
         DefaultListableBeanFactory beanFactory = createBeanFactory();
         beanFactory.setSerializationId(getId());
         customizeBeanFactory(beanFactory);
         //启动对BeanDefinition的载入，由子类实现
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

   1. 如果有容器存在，销毁和关闭容器

   2. 创建一个新的IOC容器

   3. 启动对BeanDefinition的载入和注册 详见下文

4. 初始化BeanFactory

5. 设置BeanFactory的后置处理

6. 调用BeanFactory的后处理器

7. 注册Bean的后处理器

8. 初始化上下文的事件（传播）机制

9. 初始化特殊的Bean

10. 检查监听Bean并向容器注册他们

11. 实例化所有的非懒启动（non-lazy-init）单例

12. 发布容器事件

### 载入和注册

#### 载入（读取配置文件）

AbstractXmlApplicationContext的实现：

1. 创建一个XmlBeanDefinitionReader读取器，并设置到BeanFactory
2. 为读取器Reader设置ResourceLoader（加载器？）
3. 获取xml文件位置/Resource
4. 使用读取器载入，文件位置的先读取Resource，最终的实现都是通过Resource载入
5. 具体的载入通过子类实现

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   //创建一个XmlBeanDefinitionReader，并且回调设置到BeanFactory中
   // Create a new XmlBeanDefinitionReader for the given BeanFactory.
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment.
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   //为Reader设置ResourceLoader
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // Allow a subclass to provide custom initialization of the reader,
   // then proceed with actually loading the bean definitions.
   initBeanDefinitionReader(beanDefinitionReader);
   //使用该Reader读取
   loadBeanDefinitions(beanDefinitionReader);
}

//XmlBeanDefinitionReader的载入实现
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

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

//具体的实现在doLoadBeanDefinitions方法，通过Document操作，然后注册
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			//取得XML文件的Document对象，通过DefaultDocumentLoader实现
			Document doc = doLoadDocument(inputSource, resource);
			//对BeanDefinition解析的详细过程
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

此处以xml配置为例，最终找到对应的配置文件，并转成java XML的Element集合（多个配置文件）

#### 解析和注册

将配置转换成BeanDefinition，并注册到容器中，此时依然没有初始化bean

BeanDefinitionParserDelegate，具体xml文件解析的实现类，整个解析注册的过程都依赖此类

1. 创建一个BeanDefinitionParserDelegate

2. 依次解析每个子标签，会按照默认标签和其他标签区分， default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />

3. <bean>标签的解析

   1. 提取id和别名

   2. 提取各种属性配置

      ```java
      public AbstractBeanDefinition parseBeanDefinitionElement(
            Element ele, String beanName, BeanDefinition containingBean) {
      
         // todo 为什么这里要用一个栈
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
            //创建一个带有父类名的BD
            AbstractBeanDefinition bd = createBeanDefinition(className, parent);
      
            //从xml中读取各种属性
            parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
      
            // <meta />
            parseMetaElements(ele, bd);
            // <lookup-method />
            parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            // <replaced-method />
            parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
      
            // <constructor-arg />
            parseConstructorArgElements(ele, bd);
            // <property />
            parsePropertyElements(ele, bd);
            // <qualifier />
            parseQualifierElements(ele, bd);
      
            bd.setResource(this.readerContext.getResource());
            bd.setSource(extractSource(ele));
      
            return bd;
         }
         catch (ClassNotFoundException ex) {
            error("Bean class [" + className + "] not found", ele, ex);
         }
         catch (NoClassDefFoundError err) {
            error("Class that bean class [" + className + "] depends on not found", ele, err);
         }
         catch (Throwable ex) {
            error("Unexpected failure during bean definition parsing", ele, ex);
         }
         finally {
            this.parseState.pop();
         }
      
         return null;
      }
      
      
      public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
      			BeanDefinition containingBean, AbstractBeanDefinition bd) {
      
      		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
      			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
      		}
      		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
      			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
      		}
      		else if (containingBean != null) {
      			// Take default from containing bean in case of an inner bean definition.
      			bd.setScope(containingBean.getScope());
      		}
      
      		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
      			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
      		}
      
      		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
      		if (DEFAULT_VALUE.equals(lazyInit)) {
      			lazyInit = this.defaults.getLazyInit();
      		}
      		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));
      
      		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
      		bd.setAutowireMode(getAutowireMode(autowire));
      
      		String dependencyCheck = ele.getAttribute(DEPENDENCY_CHECK_ATTRIBUTE);
      		bd.setDependencyCheck(getDependencyCheck(dependencyCheck));
      
      		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
      			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
      			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
      		}
      
      		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
      		if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
      			String candidatePattern = this.defaults.getAutowireCandidates();
      			if (candidatePattern != null) {
      				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
      				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
      			}
      		}
      		else {
      			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
      		}
      
      		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
      			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
      		}
      
      		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
      			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
      			if (!"".equals(initMethodName)) {
      				bd.setInitMethodName(initMethodName);
      			}
      		}
      		else {
      			if (this.defaults.getInitMethod() != null) {
      				bd.setInitMethodName(this.defaults.getInitMethod());
      				bd.setEnforceInitMethod(false);
      			}
      		}
      
      		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
      			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
      			bd.setDestroyMethodName(destroyMethodName);
      		}
      		else {
      			if (this.defaults.getDestroyMethod() != null) {
      				bd.setDestroyMethodName(this.defaults.getDestroyMethod());
      				bd.setEnforceDestroyMethod(false);
      			}
      		}
      
      		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
      			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
      		}
      		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
      			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
      		}
      
      		return bd;
      	}
      ```

   3. 自定义属性解析

4. 注册

   1. 