---
title: 日志初始化流程(2)-Log4j
date: 2020-06-06 19:30:00
tags:
---
<!-- toc -->

## 3 Log4j加载流程

首先需要简单讲下Log4j如何与SLF4J整合使用，这里在代码中并不是直接引用log4j的包，而是引入slf4j-log4j12。slf4j-log4j12可以理解为为了让log4j能够接入SLF4J体系而做的桥接。下面要进行分析的StaticLoggerBinder也实现在slf4j-log4j12中。

### 3.1 初始化入口

从slf4j-log4j12的StaticLoggerBinder开始入手过一遍流程

```java
public class StaticLoggerBinder implements LoggerFactoryBinder {

    // 静态变量首先初始化，触发执行构造函数
    private static final StaticLoggerBinder SINGLETON = new StaticLoggerBinder();

    public static final StaticLoggerBinder getSingleton() {
        return SINGLETON;
    }

    public static String REQUESTED_API_VERSION = "1.6.99"; // !final

    private static final String loggerFactoryClassStr = Log4jLoggerFactory.class.getName();

    private final ILoggerFactory loggerFactory;

    private StaticLoggerBinder() {
      // 创建一个Log4jLoggerFactory
        loggerFactory = new Log4jLoggerFactory();
        try {
            @SuppressWarnings("unused")
            Level level = Level.TRACE;
        } catch (NoSuchFieldError nsfe) {
            Util.report("This version of SLF4J requires log4j version 1.2.12 or later. See also http://www.slf4j.org/codes.html#log4j_version");
        }
    }

    public ILoggerFactory getLoggerFactory() {
        return loggerFactory;
    }

    public String getLoggerFactoryClassStr() {
        return loggerFactoryClassStr;
    }
}

public class Log4jLoggerFactory implements ILoggerFactory {
	......
    // 主要关注够构造函数
    public Log4jLoggerFactory() {
        loggerMap = new ConcurrentHashMap<String, Logger>();
        // force log4j to initialize
        org.apache.log4j.LogManager.getRootLogger();
    }
}

public class LogManager {
	// 主要关注静态方法
  static {
    // By default we use a DefaultRepositorySelector which always returns 'h'.
    Hierarchy h = new Hierarchy(new RootLogger((Level) Level.DEBUG));
    repositorySelector = new DefaultRepositorySelector(h);

    /** Search for the properties file log4j.properties in the CLASSPATH.  */
    String override =OptionConverter.getSystemProperty(DEFAULT_INIT_OVERRIDE_KEY,
                                                       null);

    // if there is no default init override, then get the resource
    // specified by the user or the default config file.
    if(override == null || "false".equalsIgnoreCase(override)) {
			
      // 尝试从配置项中获取配置文件路径
      String configurationOptionStr = OptionConverter.getSystemProperty(
        DEFAULT_CONFIGURATION_KEY, 
        null);

      String configuratorClassName = OptionConverter.getSystemProperty(
        CONFIGURATOR_CLASS_KEY, 
        null);

      URL url = null;

      // if the user has not specified the log4j.configuration
      // property, we search first for the file "log4j.xml" and then
      // "log4j.properties"
      if(configurationOptionStr == null) {	
        // 配置项没有配置的话尝试使用默认配置文件名 DEFAULT_XML_CONFIGURATION_FILE = log4j.xml
        url = Loader.getResource(DEFAULT_XML_CONFIGURATION_FILE);
        if(url == null) {
          // 第二个默认配置 DEFAULT_CONFIGURATION_FILE = log4j.properties
          // 这里就有个优先级问题了，同时存在的话看起来只有log4j.xml生效
          url = Loader.getResource(DEFAULT_CONFIGURATION_FILE);
        }
      } else {
        try {
          url = new URL(configurationOptionStr);
        } catch (MalformedURLException ex) {
          // so, resource is not a URL:
          // attempt to get the resource from the class path
          url = Loader.getResource(configurationOptionStr); 
        }	
      }

      // If we have a non-null url, then delegate the rest of the
      // configuration to the OptionConverter.selectAndConfigure
      // method.
      if(url != null) {
        LogLog.debug("Using URL ["+url+"] for automatic log4j configuration.");
        try {
          // 使用最终定位到配置文件的URL，进行配置加载
          OptionConverter.selectAndConfigure(url, configuratorClassName,
                                             LogManager.getLoggerRepository());
        } catch (NoClassDefFoundError e) {
          LogLog.warn("Error during default initialization", e);
        }
      } else {
        LogLog.debug("Could not find resource: ["+configurationOptionStr+"].");
      }
    } else {
      LogLog.debug("Default initialization of overridden by " + 
                   DEFAULT_INIT_OVERRIDE_KEY + "property."); 
    }  
  } 
}
```

可以很简单的分析出在被调用StaticLoggerBinder.getSingleton()方法后的执行顺序：

- 初始化静态变量SINGLETON = new StaticLoggerBinder()
- 执行StaticLoggerBinder构造函数，创建一个Log4jLoggerFactory实例
- 执行Log4jLoggerFactory的构造函数，这里加载了org.apache.log4j.LogManager类
- LogManager的静态代码块中尝试获取配置文件，并调用OptionConverter.selectAndConfigure进行加载

### 3.2 配置文件的加载

```java
org.apache.log4j.helpers.OptionConverter#selectAndConfigure(java.net.URL url, java.lang.String clazz, org.apache.log4j.spi.LoggerRepository LogManager.getLoggerRepository()) {
  Configurator configurator = null;
  String filename = url.getFile();

  if(clazz == null && filename != null && filename.endsWith(".xml")) {
    clazz = "org.apache.log4j.xml.DOMConfigurator";
  }

  if(clazz != null) {
    // 这里因为可能有配置项的clazz传入，所以不能直接断定是XML的DOMConfigurator进行new
    LogLog.debug("Preferred configurator class: " + clazz);
    configurator = (Configurator) instantiateByClassName(clazz,
                                                         Configurator.class,
                                                         null);
    if(configurator == null) {
      LogLog.error("Could not instantiate configurator ["+clazz+"].");
      return;
    }
  } else {
    configurator = new PropertyConfigurator();
  }

  // 前面确定了configurator，现在进行配置加载
  configurator.doConfigure(url, hierarchy);
}
```

在org.apache.log4j.helpers.OptionConverter#selectAndConfigure主要完成了配置项解析器的确定及实例化，并调用解析器对配置进行解析。

#### 3.2.1 XML解析流程

在configurator.doConfigure(url, hierarchy)之后，经过一些简单逻辑，流程走到了parse方法这，此时repostory已经被赋值为刚才传入的参数hierarchy。

```java
public class DOMConfigurator implements Configurator {
  
  LoggerRepository repository;
  protected void parse(Element element) {
    String rootElementName = element.getTagName();
    // 根据xml 检查版本
    ......
      // 根据xml 是否开启debug
      ......

      //
      //   reset repository before configuration if reset="true"
      //       on configuration element.
      // TODO 不知道干啥的，先忽略
      ......

      // 根据xml 设置日志级别
      ......

      //Hashtable appenderBag = new Hashtable(11);

      /* Building Appender objects, placing them in a local namespace
       for future reference */

      // First configure each category factory under the root element.
      // Category factories need to be configured before any of
      // categories they support.
      //
      String   tagName = null;
    Element  currentElement = null;
    Node     currentNode = null;
    NodeList children = element.getChildNodes();
    final int length = children.getLength();

    // 遍历xml 优先解析categoryFactory
    ......
      if (tagName.equals(CATEGORY_FACTORY_TAG) || tagName.equals(LOGGER_FACTORY_TAG)) {
        parseCategoryFactory(currentElement);
      }
    ......

      //  再次遍历xml 
      for (int loop = 0; loop < length; loop++) {
        currentNode = children.item(loop);
        if (currentNode.getNodeType() == Node.ELEMENT_NODE) {
          currentElement = (Element) currentNode;
          tagName = currentElement.getTagName();

          if (tagName.equals(CATEGORY) || tagName.equals(LOGGER)) {
            // 解析logger（category）
            parseCategory(currentElement);
          } else if (tagName.equals(ROOT_TAG)) {
            // 解析root
            parseRoot(currentElement);
          } else if(tagName.equals(RENDERER_TAG)) {
            // 解析对象渲染
            parseRenderer(currentElement);
          } else if(tagName.equals(THROWABLE_RENDERER_TAG)) {
            // 解析异常渲染
            if (repository instanceof ThrowableRendererSupport) {
              ThrowableRenderer tr = parseThrowableRenderer(currentElement);
              if (tr != null) {
                ((ThrowableRendererSupport) repository).setThrowableRenderer(tr);
              }
            }
          } else if (!(tagName.equals(APPENDER_TAG)
                       || tagName.equals(CATEGORY_FACTORY_TAG)
                       || tagName.equals(LOGGER_FACTORY_TAG))) {
            // 异常项
            quietParseUnrecognizedElement(repository, currentElement, props);
          }
        }
      }
  }
  // 主要看logger及root，及它们的children是如何初始化的

  // 解析logger（创建）
  protected void parseCategory(Element loggerElement) {
    // Create a new org.apache.log4j.Category object from the <category> element.
    String catName = subst(loggerElement.getAttribute(NAME_ATTR));
    Logger cat;
    // 获取logger class(默认不需要的)
    String className = subst(loggerElement.getAttribute(CLASS_ATTR));

    // 根据logger类（有的话）创建子类
    ......

      // Setting up a category needs to be an atomic operation, in order
      // to protect potential log operations while category
      // configuration is in progress.
      synchronized (cat) {
      boolean additivity = OptionConverter.toBoolean(
        subst(loggerElement.getAttribute(ADDITIVITY_ATTR)),
        true);

      LogLog.debug("Setting [" + cat.getName() + "] additivity to [" + additivity + "].");
      cat.setAdditivity(additivity);
      // 解析logger的关联的child，主要是appender
      parseChildrenOfLoggerElement(loggerElement, cat, false);
    }
  }

  // 解析root
  protected void parseRoot(Element rootElement) {
    Logger root = repository.getRootLogger();
    // category configuration needs to be atomic
    synchronized (root) {
      // 解析root关联的child
      parseChildrenOfLoggerElement(rootElement, root, true);
    }
  }
  
  // 解析二级对象（主要从关联上来说的，一级就是logger root，二级就是被logger、root关联的appender
  protected void parseChildrenOfLoggerElement(Element catElement,
                                              Logger cat, boolean isRoot) {

    ......
      // Remove all existing appenders from cat. They will be
      // reconstructed if need be.
      cat.removeAllAppenders();


    NodeList children = catElement.getChildNodes();
    final int length = children.getLength();

    for (int loop = 0; loop < length; loop++) {
      Node currentNode = children.item(loop);

      if (currentNode.getNodeType() == Node.ELEMENT_NODE) {
        Element currentElement = (Element) currentNode;
        String tagName = currentElement.getTagName();

        if (tagName.equals(APPENDER_REF_TAG)) {
          // 解析到是appender则创建
          Element appenderRef = (Element) currentNode;
          // 查找或创建 appender
          Appender appender = findAppenderByReference(appenderRef);
          ......
            // logger关联上appender
            cat.addAppender(appender);
          .............
        } else {
          quietParseUnrecognizedElement(cat, currentElement, props);
        }
      }
    }
    ......
  }

  // findAppenderByReference(appenderRef) 最终调用到 parseAppender(appenderElement)
  // appenderElement 为通过appender name 从 dom中找出来的element
  protected Appender parseAppender(Element appenderElement) {
    String className = subst(appenderElement.getAttribute(CLASS_ATTR));
    LogLog.debug("Class name: [" + className + ']');
    try {
      // 根据className创建一个appender实例
      Object instance = Loader.loadClass(className).newInstance();
      Appender appender = (Appender) instance;
      PropertySetter propSetter = new PropertySetter(appender);

      appender.setName(subst(appenderElement.getAttribute(NAME_ATTR)));

      NodeList children = appenderElement.getChildNodes();
      final int length = children.getLength();

      for (int loop = 0; loop < length; loop++) {
        Node currentNode = children.item(loop);
        /* We're only interested in Elements */
        if (currentNode.getNodeType() == Node.ELEMENT_NODE) {
          Element currentElement = (Element) currentNode;
          // Parse appender parameters
          if (currentElement.getTagName().equals(PARAM_TAG)) {
            setParameter(currentElement, propSetter);
          }
          // Set appender layout
          else if (currentElement.getTagName().equals(LAYOUT_TAG)) {
            appender.setLayout(parseLayout(currentElement));
          }
          // Add filters
          else if (currentElement.getTagName().equals(FILTER_TAG)) {
            parseFilters(currentElement, appender);
          } else if (currentElement.getTagName().equals(ERROR_HANDLER_TAG)) {
            parseErrorHandler(currentElement, appender);
          } else if (currentElement.getTagName().equals(APPENDER_REF_TAG)) {
            String refName = subst(currentElement.getAttribute(REF_ATTR));
            if (appender instanceof AppenderAttachable) {
              AppenderAttachable aa = (AppenderAttachable) appender;
              LogLog.debug("Attaching appender named [" + refName +
                           "] to appender named [" + appender.getName() + "].");
              aa.addAppender(findAppenderByReference(currentElement));
            } else {
              LogLog.error("Requesting attachment of appender named [" +
                           refName + "] to appender named [" + appender.getName() +
                           "] which does not implement org.apache.log4j.spi.AppenderAttachable.");
            }
          } else {
            parseUnrecognizedElement(instance, currentElement, props);
          }
        }
      }
      // 启动appender
      propSetter.activate();
      return appender;
    } catch (Exception oops) {
      if (oops instanceof InterruptedException || oops instanceof InterruptedIOException) {
        Thread.currentThread().interrupt();
      }
      LogLog.error("Could not create an Appender. Reported error follows.",
                   oops);
      return null;
    }
  }
}
```

总结：

- 在XML首先遍历Logger、Root等第一层级对象
- 根据Logger、Root关联的Appender-Ref，查找Appender配置，并创建Apender
- 关联Logger与Appender

#### 3.2.2 Properties解析流程

Properties解析流程与Xml类似，同样是第一层级对象优先解析生成，之后再创建第二层级对象并关联。

### 3.3 加载流程总结

从StaticLoggerBinder到解析配置文件

 ```mermaid
sequenceDiagram
	participant StaticLoggerBinder
	participant Log4jLoggerFactory
	participant LogManager
	participant OptionConverter
	participant Configurator 
	StaticLoggerBinder ->> StaticLoggerBinder: Constructor
	StaticLoggerBinder ->>+ Log4jLoggerFactory: Constructor
	Log4jLoggerFactory ->>+ LogManager: getRootLogger()
	LogManager ->> LogManager: run static code
	LogManager ->>+ OptionConverter: selectAndConfigure(url, clazz, repo)
	OptionConverter ->>+ Configurator: doConfigure(url, hierarchy);
 ```

配置文件解析流程：

- 遍历第一层级对象，如Logger、Root等
- 对Logger、Root中引用的child挨个初始化并关联至Logger或Root，如Appender等

### 3.4 生命周期

在Log4j的设计中，Logger不存在生命周期，而Appender存在一个Active-Close的过程。

在配置文件解析过程中，Appender被创建，并被调用activeOption完成启动步骤（如打开文件，创建输出流等）。
