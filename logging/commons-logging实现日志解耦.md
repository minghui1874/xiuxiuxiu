# 需要解耦

日志是实际应用中的一个重要部分，日志系统也有许多开源的实现，如java.util.logging, logback, log4j系列等。 
在使用日志系统时，如果与具体的日志实现耦合太深，如使用log4j作为日志的实现，在每一处需要打印日志的地方都会创建日志实例：

```java
logger = LogManager.getLogger("instanceName");
```

当由于性能或者其他方面的需求需要更换日志实现时，如log4j升级到log4j2，就不得不替换每一处日志实例得创建，将会是一个噩梦般的工作。

因此需要将应用与具体的日志实现解耦。commons-logging提供了日志实现解耦方案，当然commons-logging也提供了简单的日志实现。 
commons-logging的官网地址为：[http://commons.apache.org/proper/commons-logging/](http://commons.apache.org/proper/commons-logging/) 
其中对commons-logging的介绍：

`The Logging package is an ultra-thin bridge between different logging implementations. A library that uses the commons-logging API can be used with any logging implementation at runtime. Commons-logging comes with support for a number of popular logging implementations, and writing adapters for others is a reasonably simple task.`

`译文：日志记录包是不同日志记录实现之间的超薄桥接器。使用commons-logging API的库可以在运行时与任何日志记录实现一起使用。Commons-logging支持许多流行的日志记录实现，并为其他人写入适配器是一个相当简单的任务。`

#  commons-logging简单日志实现：

commons-logging除了提供解耦合方案外，也提供了`org.apache.commons.logging.impl.SimpleLog`作为简单的日志实现。 
使用SimpleLog，需要做相应配置：

- ## commons-logging除了提供解耦合方案外，也提供了org.apache.commons.logging.impl.SimpleLog作为简单的日志实现。 
使用SimpleLog，需要做相应配置：

在classpath根路径放置名为commons-logging.properties的文件（该路径不可改变），内容为：
`org.apache.commons.logging.Log=org.apache.commons.logging.impl.SimpleLog`


- ## 创建Log实例
commons-logging.properties配置好了之后就可以创建Log实例并直接使用：

```java
private  static  final Log logger = LogFactory.getLog(CommonsLogTest.class);
```
输出为，默认日志级别为INFO:

![1](https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/1.png)



- ## 输出配置

当然，SimpleLog也可以做一些简单的输出配置，打开SimpleLog的源码，可以看到在初始化时：

```java
static {
        // Add props from the resource simplelog.properties
        InputStream in = getResourceAsStream("simplelog.properties");
        if(null != in) {
            try {
                simpleLogProps.load(in);
                in.close();
            } catch(java.io.IOException e) {
                // ignored
            }
        }
![1]($res/1.png)

        showLogName = getBooleanProperty(systemPrefix + "showlogname", showLogName);
        showShortName = getBooleanProperty(systemPrefix + "showShortLogname", showShortName);
        showDateTime = getBooleanProperty(systemPrefix + "showdatetime", showDateTime);

        if(showDateTime) {
            dateTimeFormat = getStringProperty(systemPrefix + "dateTimeFormat",
                                               dateTimeFormat);
            try {
                dateFormatter = new SimpleDateFormat(dateTimeFormat);
            } catch(IllegalArgumentException e) {
                // If the format pattern is invalid - use the default format
                dateTimeFormat = DEFAULT_DATE_TIME_FORMAT;
                dateFormatter = new SimpleDateFormat(dateTimeFormat);
            }
        }
    }
```

可以看到最开始试图从simplelog.properties读取配置，若不存在后面指定了默认配置，因此只要在classpath根路径放置simplelog.properties文件即可控制日志输出行为。 

#  commons-logging解耦原理





commons-logging最核心有用的功能是解耦，它的SimpleLog实现性能比不上其他实现，如log4j等。 
首先，日志实例是通过LogFactory的getLog(String)方法创建的：

![2](https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/2.png)

LogFatory是一个抽象类，它负责加载具体的日志实现，分析其Factory getFactory()方法：

```java
public static org.apache.commons.logging.LogFactory getFactory() throws LogConfigurationException {
        // Identify the class loader we will be using
        ClassLoader contextClassLoader = getContextClassLoaderInternal();

        if (contextClassLoader == null) {
            // This is an odd enough situation to report about. This
            // output will be a nuisance on JDK1.1, as the system
            // classloader is null in that environment.
            if (isDiagnosticsEnabled()) {
                logDiagnostic("Context classloader is null.");
            }
        }

        // Return any previously registered factory for this class loader
        org.apache.commons.logging.LogFactory factory = getCachedFactory(contextClassLoader);
        if (factory != null) {
            return factory;
        }

        if (isDiagnosticsEnabled()) {
            logDiagnostic(
                    "[LOOKUP] LogFactory implementation requested for the first time for context classloader " +
                            objectId(contextClassLoader));
            logHierarchy("[LOOKUP] ", contextClassLoader);
        }

        // Load properties file.
        //
        // If the properties file exists, then its contents are used as
        // "attributes" on the LogFactory implementation class. One particular
        // property may also control which LogFactory concrete subclass is
        // used, but only if other discovery mechanisms fail..
        //
        // As the properties file (if it exists) will be used one way or
        // another in the end we may as well look for it first.
        // classpath根目录下寻找commons-logging.properties
        Properties props = getConfigurationFile(contextClassLoader, FACTORY_PROPERTIES);

        // Determine whether we will be using the thread context class loader to
        // load logging classes or not by checking the loaded properties file (if any).
        // classpath根目录下commons-logging.properties是否配置use_tccl
        ClassLoader baseClassLoader = contextClassLoader;
        if (props != null) {
            String useTCCLStr = props.getProperty(TCCL_KEY);
            if (useTCCLStr != null) {
                // The Boolean.valueOf(useTCCLStr).booleanValue() formulation
                // is required for Java 1.2 compatibility.
                if (Boolean.valueOf(useTCCLStr).booleanValue() == false) {
                    // Don't use current context classloader when locating any
                    // LogFactory or Log classes, just use the class that loaded
                    // this abstract class. When this class is deployed in a shared
                    // classpath of a container, it means webapps cannot deploy their
                    // own logging implementations. It also means that it is up to the
                    // implementation whether to load library-specific config files
                    // from the TCCL or not.
                    baseClassLoader = thisClassLoader;
                }
            }
        }

        // 这里真正开始决定使用哪个factory
        // 首先，尝试查找vm系统属性org.apache.commons.logging.LogFactory，其是否指定factory
        // Determine which concrete LogFactory subclass to use.
        // First, try a global system property
        if (isDiagnosticsEnabled()) {
            logDiagnostic("[LOOKUP] Looking for system property [" + FACTORY_PROPERTY +
                    "] to define the LogFactory subclass to use...");
        }

        try {
            String factoryClass = getSystemProperty(FACTORY_PROPERTY, null);
            if (factoryClass != null) {
                if (isDiagnosticsEnabled()) {
                    logDiagnostic("[LOOKUP] Creating an instance of LogFactory class '" + factoryClass +
                            "' as specified by system property " + FACTORY_PROPERTY);
                }
                factory = newFactory(factoryClass, baseClassLoader, contextClassLoader);
            } else {
                if (isDiagnosticsEnabled()) {
                    logDiagnostic("[LOOKUP] No system property [" + FACTORY_PROPERTY + "] defined.");
                }
            }
        } catch (SecurityException e) {
            if (isDiagnosticsEnabled()) {
                logDiagnostic("[LOOKUP] A security exception occurred while trying to create an" +
                        " instance of the custom factory class" + ": [" + trim(e.getMessage()) +
                        "]. Trying alternative implementations...");
            }
            // ignore
        } catch (RuntimeException e) {
            // This is not consistent with the behaviour when a bad LogFactory class is
            // specified in a services file.
            //
            // One possible exception that can occur here is a ClassCastException when
            // the specified class wasn't castable to this LogFactory type.
            if (isDiagnosticsEnabled()) {
                logDiagnostic("[LOOKUP] An exception occurred while trying to create an" +
                        " instance of the custom factory class" + ": [" +
                        trim(e.getMessage()) +
                        "] as specified by a system property.");
            }
            throw e;
        }

        // 第二，尝试使用java spi服务发现机制，载META-INF/services下寻找org.apache.commons.logging.LogFactory实现
        // Second, try to find a service by using the JDK1.3 class
        // discovery mechanism, which involves putting a file with the name
        // of an interface class in the META-INF/services directory, where the
        // contents of the file is a single line specifying a concrete class
        // that implements the desired interface.

        if (factory == null) {
            if (isDiagnosticsEnabled()) {
                logDiagnostic("[LOOKUP] Looking for a resource file of name [" + SERVICE_ID +
                        "] to define the LogFactory subclass to use...");
            }
            try {
                // META-INF/services/org.apache.commons.logging.LogFactory, SERVICE_ID
                final InputStream is = getResourceAsStream(contextClassLoader, SERVICE_ID);

                if (is != null) {
                    // This code is needed by EBCDIC and other strange systems.
                    // It's a fix for bugs reported in xerces
                    BufferedReader rd;
                    try {
                        rd = new BufferedReader(new InputStreamReader(is, "UTF-8"));
                    } catch (java.io.UnsupportedEncodingException e) {
                        rd = new BufferedReader(new InputStreamReader(is));
                    }

                    String factoryClassName = rd.readLine();
                    rd.close();

                    if (factoryClassName != null && !"".equals(factoryClassName)) {
                        if (isDiagnosticsEnabled()) {
                            logDiagnostic("[LOOKUP]  Creating an instance of LogFactory class " +
                                    factoryClassName +
                                    " as specified by file '" + SERVICE_ID +
                                    "' which was present in the path of the context classloader.");
                        }
                        factory = newFactory(factoryClassName, baseClassLoader, contextClassLoader);
                    }
                } else {
                    // is == null
                    if (isDiagnosticsEnabled()) {
                        logDiagnostic("[LOOKUP] No resource file with name '" + SERVICE_ID + "' found.");
                    }
                }
            } catch (Exception ex) {
                // note: if the specified LogFactory class wasn't compatible with LogFactory
                // for some reason, a ClassCastException will be caught here, and attempts will
                // continue to find a compatible class.
                if (isDiagnosticsEnabled()) {
                    logDiagnostic(
                            "[LOOKUP] A security exception occurred while trying to create an" +
                                    " instance of the custom factory class" +
                                    ": [" + trim(ex.getMessage()) +
                                    "]. Trying alternative implementations...");
                }
                // ignore
            }
        }

        // 第三，尝试从classpath根目录下的commons-logging.properties中查找org.apache.commons.logging.LogFactory属性指定的factory
        // Third try looking into the properties file read earlier (if found)

        if (factory == null) {
            if (props != null) {
                if (isDiagnosticsEnabled()) {
                    logDiagnostic(
                            "[LOOKUP] Looking in properties file for entry with key '" + FACTORY_PROPERTY +
                                    "' to define the LogFactory subclass to use...");
                }
                String factoryClass = props.getProperty(FACTORY_PROPERTY);
                if (factoryClass != null) {
                    if (isDiagnosticsEnabled()) {
                        logDiagnostic(
                                "[LOOKUP] Properties file specifies LogFactory subclass '" + factoryClass + "'");
                    }
                    factory = newFactory(factoryClass, baseClassLoader, contextClassLoader);

                    // TODO: think about whether we need to handle exceptions from newFactory
                } else {
                    if (isDiagnosticsEnabled()) {
                        logDiagnostic("[LOOKUP] Properties file has no entry specifying LogFactory subclass.");
                    }
                }
            } else {
                if (isDiagnosticsEnabled()) {
                    logDiagnostic("[LOOKUP] No properties file available to determine" + " LogFactory subclass from..");
                }
            }
        }

        // 最后，使用后备factory实现，org.apache.commons.logging.impl.LogFactoryImpl
        // Fourth, try the fallback implementation class

        if (factory == null) {
            if (isDiagnosticsEnabled()) {
                logDiagnostic(
                        "[LOOKUP] Loading the default LogFactory implementation '" + FACTORY_DEFAULT +
                                "' via the same classloader that loaded this LogFactory" +
                                " class (ie not looking in the context classloader).");
            }

            // Note: unlike the above code which can try to load custom LogFactory
            // implementations via the TCCL, we don't try to load the default LogFactory
            // implementation via the context classloader because:
            // * that can cause problems (see comments in newFactory method)
            // * no-one should be customising the code of the default class
            // Yes, we do give up the ability for the child to ship a newer
            // version of the LogFactoryImpl class and have it used dynamically
            // by an old LogFactory class in the parent, but that isn't
            // necessarily a good idea anyway.
            factory = newFactory(FACTORY_DEFAULT, thisClassLoader, contextClassLoader);
        }

        if (factory != null) {
            /**
             * Always cache using context class loader.
             */
            cacheFactory(contextClassLoader, factory);

            if (props != null) {
                Enumeration names = props.propertyNames();
                while (names.hasMoreElements()) {
                    String name = (String) names.nextElement();
                    String value = props.getProperty(name);
                    factory.setAttribute(name, value);
                }
            }
        }

        return factory;
    }
```

可以看出，抽象类LogFactory加载具体实现的步骤如下：

```
1.  从vm系统属性org.apache.commons.logging.LogFactory
2.  使用SPI服务发现机制，发现org.apache.commons.logging.LogFactory的实现
3.  查找classpath根目录commons-logging.properties的org.apache.commons.logging.LogFactory属性是否指定factory实现
4.  使用默认factory实现，org.apache.commons.logging.impl.LogFactoryImpl
```

LogFactory的getLog()方法返回类型是org.apache.commons.logging.Log接口，提供了从trace到fatal方法。可以确定，如果日志实现提供者只要实现该接口，并且使用继承自org.apache.commons.logging.LogFactory的子类创建Log，必然可以构建一个松耦合的日志系统。

# log4j+commons-logging解耦

使用commons-logging解耦log4j，可按照下列步骤：

### 1\. 将log4j和commons-logging依赖放入classpath：

```xml
 <!--commons-logging-->
<dependency>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
    <version>1.2</version>
</dependency>
<!-- log4j -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```
### 2\. 配置log4j.xml或者log4j.properties，放入类路径根目录

### 3\. 使用commons-logging得LogFactory获取日志实例
log4j提供了LogManager.getLogger(String) 方法创建日志实例，但是为了日志实现的解耦，必须使用commons-logging提供的抽象工厂LogFactory创建日志实例：
```java
Log logger = LogFactory.getLog(String)
```
接下来就可以使用logger输出日志信息了。

**log4j如何被commons-logging加载？** 
根据上述commons-logging解耦的原理，如果log4j是通过继承抽象工厂org.apache.commons.logging.LogFactory来创建实现org.apache.commons.logging.Log接口的 
日志实例的，那么就可以通过3种方法让commons-logging加载log4j，分别是：

```
1.  从vm系统属性org.apache.commons.logging.LogFactory
2.  使用SPI服务发现机制，发现org.apache.commons.logging.LogFactory的实现
3.  查找classpath根目录commons-logging.properties的org.apache.commons.logging.LogFactory属性是否指定factory实现
```


但是，查看log4j的jar包，发现在META-INF下面并没有名为services得文件夹，更别说 
org.apache.commons.logging.LogFactory文件了：

![3](https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/3.png)

而且，Logger类也没有实现commons-logging提供的Log接口：

![4]($res/https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/4.png

log4j中也没有找到继承org.apache.commons.logging.LogFactory的类。 
**那么，log4j是如何被commons-logging加载的呢？** 
按照上述LogFactory寻找factory实现的流程，LogFactory在找不到`org.apache.commons.logging.LogFactory`实现时，会使用默认实现`org.apache.commons.logging.impl.LogFactoryImpl`。


![5](https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/5.png)

通过分析LogFactoryImpl的getInstance()方法，其调用以下方法获得logger实例：

```java
protected Log newInstance(String name) throws LogConfigurationException {
        Log instance;
        try {
            if (logConstructor == null) {
                instance = discoverLogImplementation(name);
            }
            else {
                Object params[] = { name };
                instance = (Log) logConstructor.newInstance(params);
            }

            if (logMethod != null) {
                Object params[] = { this };
                logMethod.invoke(instance, params);
            }

            return instance;

        } catch (LogConfigurationException lce) {

            // this type of exception means there was a problem in discovery
            // and we've already output diagnostics about the issue, etc.;
            // just pass it on
            throw lce;

        } catch (InvocationTargetException e) {
            // A problem occurred invoking the Constructor or Method
            // previously discovered
            Throwable c = e.getTargetException();
            throw new LogConfigurationException(c == null ? e : c);
        } catch (Throwable t) {
            handleThrowable(t); // may re-throw t
            // A problem occurred invoking the Constructor or Method
            // previously discovered
            throw new LogConfigurationException(t);
        }
    }
```

**discoverLogImplementation方法如下：**

```
1.  该方法首先查找用户是否指定了日志实现
2.  classesToDiscover定义如下： 
    private static final String[] classesToDiscover = { 
    LOGGING_IMPL_LOG4J_LOGGER, // org.apache.commons.logging.impl.Log4JLogger 
    “org.apache.commons.logging.impl.Jdk14Logger”, 
    “org.apache.commons.logging.impl.Jdk13LumberjackLogger”, 
    “org.apache.commons.logging.impl.SimpleLog” 
    }; 
    用户没有指定日志实现的情况下，按照classesToDiscover数组元素的顺序，依次创建对应Log实例，直到返回成功创建的Log实例:
```
```java
private Log discoverLogImplementation(String logCategory) throws LogConfigurationException {
        if (isDiagnosticsEnabled()) {
            logDiagnostic("Discovering a Log implementation...");
        }

        initConfiguration();

        Log result = null;

        // See if the user specified the Log implementation to use
        // 查找用户是否指定了日志实现?
        String specifiedLogClassName = findUserSpecifiedLogClassName();

        if (specifiedLogClassName != null) {
            if (isDiagnosticsEnabled()) {
                logDiagnostic("Attempting to load user-specified log class '" +
                    specifiedLogClassName + "'...");
            }

            result = createLogFromClass(specifiedLogClassName,
                                        logCategory,
                                        true);
            if (result == null) {
                StringBuffer messageBuffer =  new StringBuffer("User-specified log class '");
                messageBuffer.append(specifiedLogClassName);
                messageBuffer.append("' cannot be found or is not useable.");

                // Mistyping or misspelling names is a common fault.
                // Construct a good error message, if we can
                informUponSimilarName(messageBuffer, specifiedLogClassName, LOGGING_IMPL_LOG4J_LOGGER);
                informUponSimilarName(messageBuffer, specifiedLogClassName, LOGGING_IMPL_JDK14_LOGGER);
                informUponSimilarName(messageBuffer, specifiedLogClassName, LOGGING_IMPL_LUMBERJACK_LOGGER);
                informUponSimilarName(messageBuffer, specifiedLogClassName, LOGGING_IMPL_SIMPLE_LOGGER);
                throw new LogConfigurationException(messageBuffer.toString());
            }

            return result;
        }

        // No user specified log; try to discover what's on the classpath
        //
        // Note that we deliberately loop here over classesToDiscover and
        // expect method createLogFromClass to loop over the possible source
        // classloaders. The effect is:
        //   for each discoverable log adapter
        //      for each possible classloader
        //          see if it works
        //
        // It appears reasonable at first glance to do the opposite:
        //   for each possible classloader
        //     for each discoverable log adapter
        //        see if it works
        //
        // The latter certainly has advantages for user-installable logging
        // libraries such as log4j; in a webapp for example this code should
        // first check whether the user has provided any of the possible
        // logging libraries before looking in the parent classloader.
        // Unfortunately, however, Jdk14Logger will always work in jvm>=1.4,
        // and SimpleLog will always work in any JVM. So the loop would never
        // ever look for logging libraries in the parent classpath. Yet many
        // users would expect that putting log4j there would cause it to be
        // detected (and this is the historical JCL behaviour). So we go with
        // the first approach. A user that has bundled a specific logging lib
        // in a webapp should use a commons-logging.properties file or a
        // service file in META-INF to force use of that logging lib anyway,
        // rather than relying on discovery.

        if (isDiagnosticsEnabled()) {
            logDiagnostic(
                "No user-specified Log implementation; performing discovery" +
                " using the standard supported logging implementations...");
        }
        // 用户没有指定日志实现的情况下，按照classesToDiscover数组元素的顺序，依次创建对应Log实例，直到返回成功创建的Log实例
        for(int i=0; i<classesToDiscover.length && result == null; ++i) {
            result = createLogFromClass(classesToDiscover[i], logCategory, true);
        }

        if (result == null) {
            throw new LogConfigurationException
                        ("No suitable Log implementation");
        }

        return result;
    } 
```

**findUserSpecifiedLogClassName方法如下：**

```
1.  该方法首先试图获取commons-logging.properties 中的org.apache.commons.logging.Log指定的日志实现
2.  试图获取commons-logging.properties 中的org.apache.commons.logging.log指定的日志实现
3.  试图获取vm系统属性org.apache.commons.logging.Log指定的日志实现
4.  试图获取vm系统属性org.apache.commons.logging.log指定的日志实现
5.  都没有返回null
```

```java
private String findUserSpecifiedLogClassName() {
        if (isDiagnosticsEnabled()) {
            logDiagnostic("Trying to get log class from attribute '" + LOG_PROPERTY + "'");
        }
        // 试图获取commons-logging.properties 中的org.apache.commons.logging.Log指定的日志实现
        String specifiedClass = (String) getAttribute(LOG_PROPERTY);

        if (specifiedClass == null) { // @deprecated
            if (isDiagnosticsEnabled()) {
                logDiagnostic("Trying to get log class from attribute '" +
                              LOG_PROPERTY_OLD + "'");
            }
            // 试图获取commons-logging.properties 中的org.apache.commons.logging.log指定的日志实现
            specifiedClass = (String) getAttribute(LOG_PROPERTY_OLD);
        }

        if (specifiedClass == null) {
            if (isDiagnosticsEnabled()) {
                logDiagnostic("Trying to get log class from system property '" +
                          LOG_PROPERTY + "'");
            }
            try {
                // 试图获取vm系统属性org.apache.commons.logging.Log指定的日志实现
                specifiedClass = getSystemProperty(LOG_PROPERTY, null);
            } catch (SecurityException e) {
                if (isDiagnosticsEnabled()) {
                    logDiagnostic("No access allowed to system property '" +
                        LOG_PROPERTY + "' - " + e.getMessage());
                }
            }
        }

        if (specifiedClass == null) { // @deprecated
            if (isDiagnosticsEnabled()) {
                logDiagnostic("Trying to get log class from system property '" +
                          LOG_PROPERTY_OLD + "'");
            }
            try {
                // 试图获取vm系统属性org.apache.commons.logging.log指定的日志实现
                specifiedClass = getSystemProperty(LOG_PROPERTY_OLD, null);
            } catch (SecurityException e) {
                if (isDiagnosticsEnabled()) {
                    logDiagnostic("No access allowed to system property '" +
                        LOG_PROPERTY_OLD + "' - " + e.getMessage());
                }
            }
        }

        // Remove any whitespace; it's never valid in a classname so its
        // presence just means a user mistake. As we know what they meant,
        // we may as well strip the spaces.
        if (specifiedClass != null) {
            specifiedClass = specifiedClass.trim();
        }

        return specifiedClass;
    }
```

如此一来，如果像上述一样没有指定任何`org.apache.commons.logging.Log`的实现，那么commons-logging首先使用的是 
`org.apache.commons.logging.impl.Log4JLogger`作为Log实现，可以看到Log4jLogger是实现了Log接口的：
```java
public  class Log4JLogger implements Log, Serializable；
```
Log4jLogger则是通过组合log4j提供的org.apache.log4j.Logger，来提供trace到fatal功能。
![6](https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/6.png)

在log4j的jar包没有导入到classpath之前，这个类是无法通过编译的。

到现在应该知道了commons-logging是如何与log4j实现解耦的了，不是通过commons-logging提供的抽象工厂，而是通过 
`org.apache.commons.logging.impl.LogFactoryImpl`加载具体的日志实现。不得不说，在我看来这种设计极其不好，完全可以用抽象工厂提供的日志实现加载！

#  log4j2+commons-logging解耦

log4j2是Apache推出的log4j的升级版本，在性能等各方面都做了提升。 
要想使用commons-logging解耦log4j2，可按照下列步骤：

### 1\. 导入log4j2, commons-logging, log4j-jcl依赖

其中，log4j-jcl是用来作为log4j和commons-logging的桥接的

```xml
<!--log4j2-->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.7</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.7</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-jcl</artifactId>
    <version>2.7</version>
</dependency>
```

### 2\. 配置好log4j2依赖的配置

log4j2支持了更多的配置方式，包括xml和json格式；若不指定配置文件位置，将其放置在classpath根路径

### 3\. 创建日志实例使用

log4j2也提供了自己的日志实例创建的方法： 
logger = LogManager.getLogger(Log4jTwoTest.class); 


为了实现解耦，需要使用commons-logging提供的抽象共产LogFactory.getLog(String)方法获取日志实例即可；

**解耦原理：** 
log4j-jcl提供了解耦的实现，观察其jar包：

![7](https://github.com/minghui1874/xiuxiuxiu/blob/master/logging/7.png)

该文件内容为：`org.apache.logging.log4j.jcl.LogFactoryImpl `
根据commons-logging抽象日志工厂的第二步骤spi服务发现规则，抽象工厂使用`org.apache.logging.log4j.jcl.LogFactoryImpl`作为实际产生日志实现的工厂。 
分析LogFactoryImpl的代码：

```java
private final LoggerAdapter<Log> adapter = new LogAdapter();

private final ConcurrentMap<String, Object> attributes = new ConcurrentHashMap<>();

@Override
public Log getInstance(final String name) throws LogConfigurationException {
    return adapter.getLogger(name);
}
```

Log实例是由LogAdapter类的getLogger(String)方法创建的，其继承了抽象适配器AbstractLoggerAdapter的getLogger方法：
```java
public L getLogger(String name) {
    LoggerContext context = this.getContext();
    ConcurrentMap loggers = this.getLoggersInContext(context);
    Object logger = loggers.get(name);
    if(logger != null) {
        return logger;
    } else {
        loggers.putIfAbsent(name, this.newLogger(name, context));
        return loggers.get(name);
    }
}
```

最终调用LogAdapter的newLogger()方法产生日志实例：
```java
@Override
protected Log newLogger(final String name, final LoggerContext context) {
    return new Log4jLog(context.getLogger(name));
}
```

Log4jLog组合了log4j2中的ExtendedLogger提供日志打印功能。 
可以看出，log4j2和commons-logging的解耦，完全使用了commons-logging提供的spi服务发现机制。

