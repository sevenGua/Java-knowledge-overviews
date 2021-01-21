# Web服务器深度解析及调优

## Tomcat

[Apache Tomcat® - Welcome!](https://tomcat.apache.org/)

[tomcat中文网 – tomcat使用、tomcat安装、tomcat下载](http://www.tomcat.org.cn/)

[一念成佛_LHY的博客_CSDN博客-Tomcat,Mysql,java进阶领域博主](https://blog.csdn.net/qq_36807862/article/list/4)

[Tomcat的使用（详细流程）_崔满汉的段全席的博客-CSDN博客_tomcat](https://blog.csdn.net/weixin_40396459/article/details/81706543)

 Tomcat 服务器是一个开源的轻量级Web应用服务器，在中小型系统和并发量小的场合下被普遍使用，是开发和调试Servlet、JSP 程序的首选。如果把Tomcat的内核高度的抽象，则它可以看成连接器（Connector）组件和容器（Container）组件组成，其中Connector组件负责在服务端处理客户端的连接，包括接收客户端连接、接收客户端的消息报文以及消息报文的解析等工作；Container组件则负责对客户端的请求进行逻辑处理，并把结果返回给客户端。

从Tomcat服务器配置文件server.xmlde内容格式也可以看出Tomcat的大致结构：

<?xml version=’1.0’ encoding=’utf-8’?>

<Server>

<Listener/>

<GlobalNamingResources>

<Resource/>

</ GlobalNamingResources >

<Service>

<Executor/>

<Connector/>

<Engine>

<Cluster/>

<Realm/>

<Host>

<Context/>

</Host>

</Engine>

</Service>

</Server>

### Server组件与Service组件

#### Server组件

作为Tomcat最外层的核心组件，Server组件的作用主要有以下几个。

- 提供了监听器机制，用于在Tomcat整个生命周期中对不同事件进行处理；
- 提供了Tomcat容器全局的命名资源实现；
- 监听某个端口以接收SHUTDOWN命令；

##### 生命周期监听器

为了在Server组件的某阶段执行某些逻辑，于是提供了监听器机制。在Tomcat中实现一个生命周期监听器很简单，只要实现LifecycleListener接口即可，在lifecycleEvent方法中对感兴趣的生命周期事件进行处理。

##### 全局命名资源

在Tomcat启动的时候，通过Digester框架将server.xml的描述映射到对象，在Server组件中创建NamingResource和NamingContextListener两个对象。监听器将在启动初始化的时候利用ContextResource里面的属性创建命名上下文，并组织成树状；

在Web应用中是无法访问全局命名资源的。因为要访问全局命名资源，所以这些资源都必须放置在Server组件中；

##### 监听SHUTDWON命令

Server会另外开方一个端口用于监听关闭命令，这个端口默认是8005，此端口与接收客户端请求的端口并不是同一个。客户端传输的第一行如果能够匹配关闭命令（默认是SHUTDOWN）,则整个Server将关闭；

实现这个功能：

​          Tomcat中有两类线程，一类是主线程，另外一类是daemon线程。当Tomcat启动的时候，Server将被主线程执行，其实就是完成所有的启动工作，包括启动接收客户端和处理客户端报文的线程，这些线程都是daemon线程；。所有的启动工作完成之后，主线程将进入等待SHUTDOWN命令，它将不断尝试读取客户端发送过来的消息，一旦匹配SHUTDWON命令则跳出循环。主线程继续往下执行Tomcat的关闭工作。最后主线程结束，整个tomcat停止；

#### Service组件

Service组件是一个简单地组件，Service组件是若干Connector组件和Executor组件组合而成的概念。

Connector组件负责监听某个端口的客户端请求，不同的端口对应不同的Connector。

Executor组件在Service抽象层面提供了线程池，让Service下的组件可以通用线程池；

使用池的技术：就是为了尽量的减少创建和销毁的连接操作；

### Connector组件

Connector（连接器）组件是Tomcat最核心的两个组件之一，主要的职责就是负责接收客户端连接和客户端请求的处理加工。每个Connector都将指定一个端口进行监听，分别负责对请求报文的解析和响应报文组装，解析过程生成Request对象，而组装过程涉及Response对象；

如果将Tomcat整体比作一个巨大的城堡，那么Connector组件就是城堡的城门，每个人要进入城门就必须通过城门，它为人们进出城堡提供了通道。同时，一个城堡还可能有两个或者多个城门，每个城门代表了不同的通道；

Connector组件其中包含Protocol组件、Mapper组件和CoyoteAdapter组件；

Protocol组件：协议的抽象，对不同的通信协议进行了封装，比如HTTP协议和AJP协议；

Endpoint是接收端的抽象，由于使用了不同的I/O模式，因此存在很多类型的Endpoint，如BIO模式的JionPoint、NIO模式的NioEndPoint和本地库I/O模式的APREndpoint。

Acceptor是专门用于接收客户端连接的接收器组件；

Executor则是处理客户端请求的连接池，Connector可能使用了Service组件的共享线程池，也可能是Connector自己私有的线程池。

Processor组件是处理客户端请求的处理器，不同的协议和不同的I/O模式都有不同的处理方式，所以存在不同类型的Processor；

Mapper组件可以称为路由器，它提供了对客户端请求URL的映射功能，即可以通过它将请求转发到对应的Host组件、Context组件、Wrapper组件以进行处理并响应客户端，也就是我们说的将某客户端请求发送到某虚拟机上的某个Web应用的某个Servlet。

CoyoteAdapter组件是一个适配器，它负责将Connector组件和Engine容器适配连接起来。把接收到的客户端请求报文解析生成的请求对象Request和响应对象Response传递到Engine容器，交由容器处理；

HTTP Connector所支持的协议版本为HTTP/1.1和HTTP/1.0，无须显式适配HTTP的版本，Connector会自动适配版本。每个Connector实例对应一个端口，在同个service实例内可以配置若干个Connector实例；

AJP Connector组件用于支持AJP协议通信，当我们想将Web应用中包含的静态内容交给Apache处理的时候，Apache与Tomcat之间的通信协议则使用AJP协议；

Connector也在服务器端提供了SSL安全通道的支持，用于客户端以HTTPS方式访问，可以通过配置server.xml的<Connector>节点SSLEnabled属性开启；

### Engine容器

Engine即为全局引擎容器，包含以下主要组件：

- 虚拟主机——Host组件

Host组件是Engine容器的一个子容器，它表示一个虚拟主机。Host组件也包含了很多其他的组件

- 访问日志——AccessLog组件

因为Engine是一个全局的Servlet容器，所以这里的访问日志作用的范围是所有客户端的请求访问，不管访问哪个虚拟主机都会被该日志组件记录。

- 管道——Pipeline组件

Pipeline其实属于一种设计模式，在Tomcat中可以认为它是将不同容器级别串联起来的通道，当请求进来之后就可以通过管道进行流转处理。Tomcat中有4个级别的容器，每个容器都会有一个属于自己的Pipeline。

- Engine集群——Cluster组件

Tomcat中有Engine和Host两个级别的集群，而这里的集群组件正是属于全局引擎容器。它主要是把不同JVM的全局引擎容器内的所有应用都抽象成集群，让它们能在不同的JVM之间互相通信，是会话同步，集群部署得以实现。

- Engine域——Realm组件

Realm对象其实就是一个存储了用户、密码以及权限等的数据对象，它的存储方式可能是内存、xml文件或者数据库等。它的作用是配合Tomcat实现资源认证模块。

Tomcat中有多个级别的Realm域，这里的Realm域是Engine容器级别，在<Engine>节点下配置Realm，则在启动的时候对应的域会添加到Realm容器中。

- 生命周期监听器——LifecycleListener组件

Engine容器内的生命周期监听器是为了监听Tomcat从启动到关闭整个过程的某些事件，然后根据这些事件做不同的逻辑处理。

- 日志——Log组件

日志组件负责的事情就是不同级别的日志输出，几乎所有的系统都有日志组件

### Host容器

一个Servlet引擎可以包含若干个Host容器，它是根据URL地址中的主机部分抽象的，一个Servlet引擎可以包含若干个Host容器，而一个Host容器可以包含若干个Context容器、AccessLog组件、Pipeline组件、Cluster组件、Realm组件、HostConfig组件和Log组件。

#### Web应用——Context

每个Host容器保安若干个Web应用（Context），对于Web项目来讲，其结构相对比较复杂，而且包含很多机制，Tomcat需要对它的结构进行解析，同时还需要具体实现各种功能和机制，这些复杂的工作就交给了Context容器。Context容器对应实现了Web应用包含的语义，实现了Servlet和JSP规范；

#### 访问日志——AccessLog

Host容器里面的AccessLog组件负责客户端请求访问日志的记录。Host容器的访问日志作用范围是该虚拟主机的所有客户端的请求访问，不管是哪个应用都会被该日志组件记录。

#### 管道——Pipeline

在Tomcat中，它是将不同级别的容器串联起来的通道，当请求进来的时候就通过管道进行流转处理。

#### Host集群——Cluster

它提供了Host级别的集群会话和集群部署；

#### Host域——Realm

Realm对象其实就是一个存储了用户、密码以及权限等的数据对象，它的存储方式可能是内存、xml文件或者数据库等。它的作用是配合Tomcat实现资源认证模块。

Tomcat中有多个级别的Realm域，这里的Realm域是Host容器级别，在<Host>节点中配置；

#### 生命周期监听器——HostConfig

在Tomcat启动的时候有两个阶段将Context添加到Host中。

**第一种方式：用Digester****框架解析server.xml****；**

需要先将Context节点配置到server.xml的Host节点下，缺点就是将应用配置与Web服务器耦合在一起，而且对server.xml配置的修改不会立即生效，除非重启Tomcat。

**第二种方式：在server.xml****加载解析完之后再在特定的时刻寻找指定的Context****配置文件。**这时候已经将应用配置解耦出Web服务器。

HostConfig监听器当START_EVENT事件发生的时候则会执行Web应用部署加载动作。

Web应用有三种部署类型：

- Descriptor描述符类型
- WAR包类型
- 目录类型

##### Descriptor描述符类型

Descriptor描述符的部署是通过对指定部署文件解析之后进行部署的。部署文件会按照一定的规则存放，一般为%CATALINA_HOME%/conf[EngineName]/[HostName]/Web项目名.xml，此文件的大致内容为<Context docBase=”web应用的绝对路径”reloadable=”true”> reloadable=”true”表示/WEB-INF/classes/和/WEB-INF/lib改变的时候会自动的重新加载，如果一个Host包含多个Context则可以配置多个XML描述文件。为了优化多个应用项目部署时间，HostConfig监听器引入了使用线程池和Future优化部署耗时。；

##### WAR包类型

WAR包类型部署方式是直接读取%CATALINA_HOME%/webapps目录下所有以war包形式打包的web项目，然后根据war包的内容生成Tomcat内部需要的各种对象。为了优化多个应用项目部署时间，HostConfig监听器引入了使用线程池和Future优化部署耗时。；

##### 目录类型

目录类型部署方式是直接读取%CATALINA_HOME%/webapps目录下所有目录形式的Web项目，与前面两种一样，HostConfig监听器引入了使用线程池和Future优化部署耗时。

### Context容器

Container容器是子容器的父接口，所有的子容器都必须实现这个接口，在Tomcat中Container容器的设计是典型的责任链设计模式，其有四个子容器：Engine、Host、Context和Wrapper。这四个容器之间是父子关系，Engine容器包含Host，Host包含Context，Context包含Wrapper。 

我们在web项目中的一个Servlet类对应一个Wrapper，多个Servlet就对应多个Wrapper，当有多个Wrapper的时候就需要一个容器来管理这些Wrapper了，这就是Context容器了，Context容器对应一个工程，所以我们新部署一个工程到Tomcat中就会新创建一个Context容器。

#### Context容器的配置文件

1. Tomcat的server.xml配置文件中的<Context>节点用于配置Context，它直接在Tomcat解析server.xml的时候，就完成Context对象的创建，而不用交给HostConfig监听器创建；
2. Web应用的/META-INF/context.xml文件可用于配置Context，此配置文件用于配置Web应用对应的Context属性。
3. 可用%CATALINA_HOME%/conf[EngineName]/[HostName]/Web项目名.xml文件声明创建一个Context。
4. Tomcat全局配置为conf/context.xml，此文件配置的属性会设置到所有的Context中
5. Tomcat的Host级别配置文件为/ conf[EngineName]/[HostName]/context.xml.default文件，它配置的属性会设置到某Host下面所有的Context中。

#### 包装器——Wrapper

在web项目中的一个Servlet类对应一个Wrapper，多个Servlet就对应多个Wrapper。它不能再包含其他的子容器了，它的父容器必须是Context容器。

#### Context域——Realm

#### 访问日志——AccessLog

每个Context容器可以有自己的访问日志组件。

#### 错误页面——ErrorPage

每个Context容器都拥有各自的错误处理页面，它用于定义在Web容器处理过程中出现问题之后向客户端展示错误信息的页面。它可以在Web部署描述文件中配置；

Web应用启动之后，会将web.xml中配置的这些error-page元素读取到ConText容器中，并以ErrorPage对象形式存在。ErrorPage类包含三个属性:errorCode、exceptionType和location，刚好对应web.xml中的error-page元素；

#### 会话管理器——Manager

每个Context都会有自己的会话管理器；

#### 目录上下文——DirContext

DirContext接口其实属于JNDI的标准接口，实现此接口即可实现目录对象相关属性的操作。

对于Tomcat来讲，就是Context容器需要支持一种便捷的方式去访问整个Web应用包含的文件。例如，通过一个字符串路径就可以找到对应的文件资源； 

DirContext接口要完成的事情就是通过某些字符串便捷的获取对应的内容，于是Context容器需要依赖这个接口，为后面的访问提供便捷的访问方式。

war包和Web目录分别对应DirContext的实现类为WARDirContext和FileDirContext。

#### 安全认证

web.xml中涉及安全认证的元素由<security-constraint>元素和<login-config>元素，Tomcat启动的时候将这些元素转换为Java形态，即SecurityConstraint和LoginConfig对象。

#### Jar扫描器——JarScanner

它一般在Context容器中，专门用于扫描Context对应的Web应用的JAR包。

#### 过滤器

过滤器提供了为某个Web应用的所有请求和响应做统一逻辑处理的功能。

每个Context可以包含若干个过滤器。

Tomcat 解析<filter>的时候将该元素转换为FilterDef实体对象；

ContextFilterMaps对应<filter-mapping>元素，用于保存过滤器映射关系；

ApplicationFilterConfig是FilterConfig接口的具体实现，每当初始化一个Filter的时候，ApplicationFilterConfig对象会作为Filter的init方法的参数。

调用这些过滤器进行过滤的工作则由Wrapper容器中的管道负责；

#### 命令资源——NamingResource

NamingResource将配置文件中声明的不同的资源及其属性映射到内存中，这些映射统一由NamingResource对象封装；

#### Servlet映射器——Mapper

Context容器包含了一个请求路由映射器Mapper组件，它属于局部路由映射器，它只能负责本Context容器内额路由导航，即每个Web应用包含若干个Servlet，而当对请求使用分发器RequestDispatcher以分发到不同的Servlet上处理的时，就用到了此映射器；

#### 管道——Pipeline

Context容器的管道负责对请求进行Context级别的处理，管道中包含了若干个不同逻辑处理的阀门，其中有个基础阀门，主要逻辑就是找到对应的servlet并将请求传递给它进行处理；

Tomcat中的4种容器级别都包含了各自的管道对象。

每个容器的管道完成的工作都不同，每个管道都需要搭配阀门才能工作；

#### Web应用载入器——WebappLoader

每个Web应用对应一个WebappLoader，每个WebappLoader互相隔离，各自包含的类互相不可见；

WebappLoader的核心工作其实是由内部的WebappClassLoader完成的。

#### ServletContext的实现——ApplicationContext

在Servlet规范中规定了一个ServletContext接口，它提供了Web应用所有Servlet的视图，通过它可以对某个Web应用的各种资源和功能进行访问；

对应Tomcat容器，为了满足Servlet规范，必须包含一个ServletContext接口的实现，这个实现类就是ApplicationContext。，每个Tomcat的Context容器中都会包含一个ApplicationContext。

#### 实例管理器——InstanceManager

Context容器中包含一个实例管理器，主要实现对Context容器中监听器、过滤器以及Servlet等实例进行管理。

#### ServletContainerInitializer初始化

在Web 容器启动的时候为了让第三方组件做一些初始化工作，Servlet规范中通过ServletContainerInitializer实现此功能；

#### Context容器的监听器

Context容器的生命周期伴随着Tomcat的整个生命周期，在Tomcat生命周期的不同阶段Context需要做很多不同的操作。为了更好的将这些操作从Tomcat中解耦出来，提供一种类似可拔插可扩展的能力，这里使用了监听器模式，把不同类型的工作交给不同的监听器，监听器对Tomcat声明周期不同阶段的事件作出响应；

### Wrapper容器

Wrapper容器是Tomcat中最小级别的容器，可能对应一个Servlet对象，也可能对应一个Servlet对象池；

#### Servlet工作机制

Servlet在初始化的时候调用init方法，在销毁时调用destroy方法，而对客户端的请求则调用service方法。对于这些机制，都必须由Tomcat在内部提供支持，具体则由Wrapper容器提供支持；

#### Servlet对象池

Servlet在不实现SingleThreadModel的情况下以单个实例运行，对应此Servlet的所有客户端请求都会使用此Servlet对象。

为了支持一个Servlet对应对应一个线程，Servlet规范提出了一个SingleThreadModel接口，对于这种模式，Tomcat的Wrapper容器使用了对象池策略；

Servlet对象池就是一个栈结构，需要的时候pop出一个对象，使用完就push回去；

#### 过滤器链

#### Servlet种类

- 普通Servlet；
- JSP页面路由到JspServlet；

JSP页面最终也会被Tomcat编译成为Servlet

- 静态资源路由到DefaultServlet，它是Tomcat专门用于处理静态资源的Servlet

#### Comet模式的支持

Comet模式是一种服务端推技术，核心思想是一种能够让服务器往客户端发送数据的方式。

该模式可以大大减少发送到服务器端的请求，从而避免了开销，而且它还具备更好的实时性；

一般Comet模式需要NIO配合，而在BIO中无法使用Comet模式。

#### WebSocket协议的支持

WebSocket协议属于HTML5标准，它能够让客户端和浏览器实现双向通信。另外WebSocket协议摒弃了HTTP协议繁琐的请求头部，而是以数据帧的方式进行传输，效率更高；

WebSocketServlet

#### 异步Servlet

Servlet中等待阻塞会导致Web容器整体的处理能力低下，因为对于比较耗时的操作，可以将它放置在一个单独的线程中处理，此过程保留了连接的请求和响应对象，在处理完成之后，可以把处理结果通知到客户端；

### 生命周期管理	

Tomcat是以容器的方式来组织整个系统架构的，就像数据结构的树。

这样只需要启动根容器，就可以将其他所所有的容器都启动，达到统一启动、停止和关闭的效果；

作为统一的入口，Lifecycle把所有的启动、停止和关闭、生命周期相关的方法都组织到一起，就可以很方便的管理Tomcat各个容器组件的生命周期；

### 日志框架及其国际化

Tomcat底层使用JDK自带的日志工具，没有使用第三方日志工具。以减少包的引用，没有采用JDK日志工具的默认配置，而是通过系统变量和重写某些类达到特定的效果；

使用到了JDK里面的三个类：MessageFormat、Locale、ResourceBundle，Tomcat中利用StringManamger将这三个类封装起来，方便操作，每个Java包对应一个StringManager对象，折中的考虑使得性能与资源得以同时兼顾；

### 公共与隔离的类加载器

类加载器：就是用于加载Java类到Java虚拟机中的组件，它负责读取Java字节码，并转换为java.lang.Class类的一个实例，是字节码.class文件得以运行。一般类加载器负责根据一个指定的类找到对应的字节码，然后根据这些字节码定义一个java类，它还可以加载资源，包括图像文件和配置文件；

类加载器的好处是：可以使Java类动态的加载到JVM中并运行，即可以在程序运行时候再加载类，提供了很灵活的动态加载方式；

#### 类加载器

在Java体系中，可以将系统分为三种类加载器：

- 启动类加载器（Bootstrap ClassLoader）：

该加载器使用C/C++实现，加载对象是Java核心库，负责加载JAVA_HOME/jre/lib目录下的JVM指定的类库

- 扩展类加载器（Extension ClassLoader）：

加载的对象为Java的扩展库，即加载JAVA_HOME/jre/lib/ext目录中的类，这个类由启动类加载器加载，但因为启动类加载器并非用Java实现，已经脱离了Java体系，它的父类加载器是启动类加载器；

- 应用程序类加载器（Application ClassLoader）：

负责加载用户类路径（CLASSPATH）指定的类库，如果程序没有指定类加载器，就默认使用应用程序类加载器，它也由启动类加载器加载，如果使用这个加载器，通过ClassLoader.getSystemClassLoader()即可；

在JVM中，一个类由完全匹配类名（包名+类名）和一个类加载器的实例ID作为唯一标识。也就是说，同一个虚拟机中可以有两个包名、类名都相同的类。只要它们是由不同的类加载器加载。这种特征为我们提供了隔离机制；

#### 自定义类加载器

继承ClassLoader类：

- 沿用双亲委派机制自定义类加载器：重写findClass()
- 打破双亲委派机制自定义类加载器：重写loadClass()和findClass()

#### Tomcat的类加载器

![img](https://img-blog.csdn.net/20180730170149981?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### 类加载器工厂——ClassLoaderFactory

#### 遭遇ClassNotFoundException

前面提到Tomcat会创建Common类加载器，CatAlina类加载器和共享类加载器，这三个其实是同一个类加载器对象。Tomcat在创建类加载器之后就马上将其设置为当前线程类加载器，即Thread.currentThread().setContextClassLoader，这里主要是为了避免加载类的时候加载不成功的问题；

### 请求URI映射器Mapper

Mapper组件主要职责就是负责Tomcat的请求路由，每个客户端的请求到达Tomcat之后，都将由Mapper路由到对应的处理逻辑上（Servlet）上，在Tomcat结构中有两种部分包含Mapper组件，一个是Connector组件，称为全局路由Mapper；另一个是Context组件，称为局部路由Mapper；

#### 请求的映射类型

每个完整的请求都会对应服务端Tomcat内部的Host、Context、Wrapper层次与之对应。而具体的路由工作则由Mapper组件负责。

#### Mapper的实现

Mapper组件会包含N个Host容器的引用，然后每个Host会包含N个Context容器引用，最后每个Context容器会包含N个Wrapper容器的引用；

#### 局部路由Mapper

局部路由Mapper提供了Context容器内部路由导航功能的组件。它只存在于Context容器中，用于记录访问资源与Wrapper之间的映射，每个Web应用都存在自己的局部路由Wrapper组件；

局部路由Mapper只能在同一个Web应用内进行转发路由，而不能实现跨Web应用的路由，如果要实现跨Web应用，需要使用重定向功能，让客户端重新定向到其他主机或者其他的Web应用上。

而对于从客户端到服务端的请求，则需要全局路由Mapper组件的参与；

#### 全局路由Mapper

除了局部路由Mapper之外，另外一种Mapper就是全局路由Mapper，它是提供了完整的路由导航功能的组件。位于Tomcat中的Connector组件中，提供对Host、Context、Wrapper容器的路由。

所以全局路由Mapper拥有Tomcat容器完整的路由映射，负责完整的请求地址路由功能；

### Tomcat的JNDI

一般来讲，要使用JNDI需要完成以下三个步骤：

1. 驱动器jar包放置；
2. 配置文件的配置；
3. 在程序中调用；

根据范围层次，可分为两种配置方案。一种是Web应用层次上的局部配置方式，它只可以在自己的Web项目中使用。另一个是全局配置方式，通过资源连接，它可以供该Tomcat下的所有Web应用使用；

#### Web应用的局部配置方式

找到Tomcat的server.xml找到工程的Context节点,添加一个私有数据源

<Context docBase="WebApp" path="/WebApp" reloadable="true" source="org.eclipse.jst.jee.server:WebApp"> 

<Resource 

  name="jdbc/mysql" 

  scope="Shareable" 

  type="javax.sql.DataSource" 

  factory="org.apache.tomcat.dbcp.dbcp.BasicDataSourceFactory" 

  url="jdbc:mysql://localhost:3306/test" 

  driverClassName ="com.mysql.jdbc.Driver" 

  username="root" 

  password="root" 

/> 

</Context> 

#### 服务器的全局配置方式

- 配置全局JNDI数据源,应用到单个应用

找到Tomcat的server.xml中GlobalNamingResources节点,在节点下加一个全局数据源

<GlobalNamingResources>

<Resource 

  name="jdbc/mysql" 

  scope="Shareable" 

  type="javax.sql.DataSource" 

  factory="org.apache.tomcat.dbcp.dbcp.BasicDataSourceFactory" 

  url="jdbc:mysql://localhost:3306/test" 

  driverClassName ="com.mysql.jdbc.Driver" 

  username="root" 

  password="root" 

/> 

</GlobalNamingResources>

找到要应用此JNDI数据源的工程Context节点,增加对全局数据源的引用ResourceLink 

<Context docBase="WebApp" path="/WebApp" reloadable="true"> 

  <ResourceLink global="jdbc/mysql" name="jdbc/mysql" type="javax.sql.DataSource" /> 

</Context> 

【Spring对JNDI数据源的引用】

在applicationContext.xml中加一个bean,替代原来的dataSource

1. **<jee:jndi-lookup** id="dataSource" jndi-name="jdbc/mysql" **/>** 

对Tomcat内部来讲，全局资源和局部命名资源都有各自的命令上下文。全局命名资源对Web应用是不可见的。只能通过ResourceLink从全局命名资源中查找对应的资源。局部部署只能有对应的web 应用使用，而全局部署可供所有的Web 应用使用。

#### Tomcat的标准资源

Tomcat标准资源包括如下几类：

1. 普通JavaBean资源：

主要用于创建某个Java类对象供Web应用使用。

1. UserDatabase资源：

它一般会配置成为一个全局资源，作为具有认证功能的数据源使用，一般该数据源通过XML（conf/tomcat-user.xml）文件存储

   2.JavaMail会话资源：

Tomcat提供JavaMail服务，可以使用发送Email功能；

   3.JDBC数据源资源

默认的JDBC数据源基于DBCP连接池；

### JSP编译器Jasper

Jasper模块是Tomcat的JSP核心引擎，我们知道JSP本质上是一个Servlet。

Tomcat使用Jasper对JSP语法进行解析，生成Servlet并生成Class字节码。另外，在运行的时候，Jasper还会检测JSP文件是否修改，如果修改，则会重新编译JSP文件。

Jasper自动检测机制

![img](https://img-blog.csdn.net/20180730170511347?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 运行、通信以及访问的安全管理

#### 运行安全管理

Tomcat在启动的时候开启了安全管理器，它采用的是默认的安全管理器——SecurityManager。

#### 安全的通信

SSL/TLS层提供三方面的服务：

1. 证书：认证客户端和服务器，确保数据发送到正确的客户端和服务器；
2. 加密，加密数据以防止数据中途被窃取；
3. 签名，维护数据的完整性，确保数据在传输过程中不被篡改；

客户单在接收到证书之后，会读取证书中发布机构，匹配浏览器内置的受信任的发布机构列表，如果找不到，则说明这个证书发布结构不可信任，证书自然也就不可信任；程序会抛出一个错误信息；如果是受信任机构发布的，浏览器将使用受信任机构证书的公钥对服务器证书里面的指纹和指纹算法进行解密，再利用这个指纹算法计算服务器证书的指纹，把计算出来的证书与证书里面的指纹进行对比，进而判断证书是否被修改过以及证书是否是受信任结构发布的；

为了利用HTTPS通信，tomcat需要利用JSSE把SSL/TLS协议集成到自身系统上；Tomcat默认使用JDK中的包实现此功能；

#### 客户端访问认证机制

##### 认证模式

认证模式是由浏览器与Web服务器之间约定；不同模式有不同的特点；

###### Basic模式

  客户端对于每一个realm，通过提供用户名和密码来进行认证的方式。

  ※　包含密码的明文传递

  基本认证步骤：

   \1. 客户端访问一个受http基本认证保护的资源。

   \2. 服务器返回401状态，要求客户端提供用户名和密码进行认证。

​      401 Unauthorized

​      WWW-Authenticate： Basic realm="WallyWorld"

   \3. 客户端将输入的用户名密码用Base64进行编码后，采用非加密的明文方式传送给服务器。

​      Authorization: Basic xxxxxxxxxx.

   \4. 如果认证成功，则返回相应的资源。如果认证失败，则仍返回401状态，要求重新进行认证。

  特记事项：

   \1. Http是无状态的，同一个客户端对同一个realm内资源的每一个访问会被要求进行认证。

   \2. 客户端通常会缓存用户名和密码，并和authentication realm一起保存，所以，一般不需要你重新输入用户名和密码。

   \3. 以非加密的明文方式传输，虽然转换成了不易被人直接识别的字符串，但是无法防止用户名密码被恶意盗用。

###### Digest模式

Digest模式避免了模式避免了密码在网络中传输，提高了安全性，但是认证报文被攻击者拦截，攻击者仍然可以获取到资源；      

###### Form模式

使用表单进行身份认证，认证之后对于同一个会话期间都不再进行认证，因为认证的结果已经保存在服务器端的会话中；

由于每种语言都可以自定义自己的Form模式，因此没有一个通用的标准，而且它也存在密码明文传输安全问题；

###### Spnego模式

![img](https://img-blog.csdn.net/20180730170726312?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

①客户端浏览器向web服务器发送http请求。

②服务器返回401状态码，响应头部加上 WWW-Authenticate:Negotiate。

③用户通过浏览器输入用户名向AS请求TGS票证。

④AS生成TGS票证，然后查询用户密码并用此密码加密TGS票证，返回浏览器。

⑤浏览器使用用户密码解密出TGS票证，并向TGS服务发起请求。

⑥TGS服务生成服务票证响应给浏览器。

⑦浏览器将服务票证封装到SPNEGO token中，并发送给web服务器。

⑧服务器解密出用户名及服务票证，将票证发往TGS服务验证。

⑨通过验证，开始通信。
它将认证模块独立出来，结构变得复杂了，但是这样可以为所有应用提供认证功能，例如它可以很容易的实现多个系统之间的单点登录；

###### SSL模式

客户端和服务器端通过SSL协议建立起SSL通道，完成整个SSL通道的建立之后，才是认证的核心步骤；

###### NonLogin模式

该模式不要求用户登录，对资源进行配置，对没有配置任何角色的资源，用户不用登录即可访问，反之，不能访问；

##### Realm域

Realm域是为了方便统一提供一种权限和资源的映射关系；

有三个级别的Realm域：

1. Engine级别

所有Web应用共享

1. Host级别

该虚拟机上的Web应用共享

1. Context级别

专属于某个Web应用

### 处理请求和响应的管道

划分出的每个小模块之间互相独立并且各自负责一段处理逻辑，这些逻辑处理小模块根据顺序连接起来，前一模块的输出作为后一个模块的输入，最后一个模块的输出为最终的处理结果。如此依一来就是管道模式，在管道中连接一个或多个阀门，每个阀门负责一部分逻辑处理，数据按规定的顺序往下流。此模式分解了逻辑处理任务，可方便对某任务单元进行安装拆卸，提高了流程的可扩展性、可重用性、机动性、灵活性。

### 多样化的会话管理器

![img](https://img-blog.csdn.net/20180730171235543?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

客户端是把jsessionid传递到服务端一般会有三种方式：

①cookie方式，即通过浏览器读取小文本cookie，读取jsessionid值后附加到http协议的cookie头部，http协议报文传输到服务端后解析cookie头部便可以获取，但如果你把浏览器的cookie给禁止了则这种方式会失效。

②重写url方式，即把jsessionid附加到请求的url中，例如http://www.tomcat.com/index.jsp?jsessionid=326257DA6DB76F8D2E38F2C4540D1DEA。

③表单隐藏方式，这种方式其实类似重写url方式，我们把jsessionid及其值存放在html表单中，提交时就会一起被提交，服务端只要根据post或get方法分别解析便可获取到。

Web容器的会话机制补充了http协议的无状态性，使web在应用功能方面更加强大，满足了更多更复杂的需求。不管是web应用层开发人员还是中间件开发人员深入理解session机制在软件设计时都会有很大的帮助。

#### 标准会话对象——StandardSession

Tomcat使用了一个StandardSession对象用来表示标准的会话结构，用来封装需要存储的状态信息。标准会话对象StandardSession实现了Session、Serializable、HttpSession等几个接口，为什么需要实现这几个接口呢？Session接口定义了tomcat内部用来操作会话的一些方法；Serializable则是序列化接口，实现它是为了方便传输及持久化；HTTPSession是Servlet规范中为会话操作而定义的一些方法，作为一个标准web容器实现它是必然的。另外还会存在一个StandardSessionFacade的外观类，外观设计模式相信大家都很熟悉了，前面的Request及Response也使用了同样的模式，都是出于安全考虑引入一个外观类，它可以把一些tomcat内部使用的方法屏蔽了，只暴露web应用层允许调用的一些方法。![img](https://img-blog.csdn.net/20180730171315226?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

一个最简单的标准会话应该包括id和Map<String, Object>结构的attribute，id用于表示会话编号，它必须是全局唯一的，attribute用于存储会话相关信息，以kv结构存储。另外还应该包括会话创建时间、事件监听器、提供web层面访问的外观类等等。

#### 增量会话对象——DeltaSession

在集群环境中为了使集群中各个节点的会话状态都同步，同步操作是集群重点解决的问题，一般来说有两种同步策略:

其一是每次同步都把整个会话对象传给集群中其他节点，其他节点更新整个会话对象；

其二是对会话中增量修改的属性进行同步。这两种同步方案各有优缺点，整个会话对象同步策略实现过程比较简单方便，但会造成大量无效信息的传输。增量同步方式则不会传递无效的信息，但在实现上会比较复杂因为涉及到对会话属性操作过程的管理。

这节讨论的正是增量同步方式中涉及的会话对象DeltaSession，这个对象其实是对标准会话对象的扩展使之具备在整个请求过程记录会话所有的增量更改。DeltaSession的类图如下，除了继承StandardSession类外还实现了Externalizable、ClusterSession、ReplicatedMapEntry三个接口，Externalizable接口主要提供对外部的对象读写操作，ClusterSession接口主要提供判断集群会话是否为原始的会话操作，只有原始会话才有资格使会话过期，ReplicatedMapEntry接口提供差异复制的操作。对于DeltaSession其实就是除了继承StandardSession特性外还要额外实现这三个接口。

![img](https://img-blog.csdn.net/20180730171354334?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

当客户端发起一个请求时，服务端对请求的处理可能涉及会话相关的操作，例如获取客户端某些属性再根据属性值进行逻辑处理，而且在整个请求过程中可能涉及多次的会话操作，为了将这些改变能同步到集群的其他节点上，必须要有一个机制来实现，实际上同步的颗粒度大小是很重要，颗粒度太大会导致同步不及时，而颗粒度太小则可能导致传输及性能问题，考虑到性能及可行性，tomcat同步的颗粒度是以一个完整的请求为单位的，即从客户端发起请求到服务器完成逻辑处理返回结果之前这段时间为同步颗粒度。这个过程中对某会话的所有操作（对同一个属性的操作只记录最新的操作）都会被记录下来，如下图，绿色箭头表示一个完整的请求过程，期间包括了四个修改属性操作，分别修改了属性a、b、c、d，这四个操作会被抽象成四个动作放进一个列表中，集群其他节点获取列表后根据这些动作就可以对自己本地对应的会话进行同步。

![img](https://img-blog.csdn.net/2018073017141167?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

集群成员接收到某节点发送过来的同步消息后，将会逐一执行动作集里面的每个动作，下图大箭头表示同步的整个过程，最下面的为动作集列表，一共有4个动作，按顺序首先取出第一个update1动作，动作对象里面包含了指定修改哪个会话的会话id，根据此id去修改会话集对应的会话的属性。接着把剩下的其余3个动作执行完毕，于是完成了会话同步。 

![img](https://img-blog.csdn.net/2018073017143272?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在tomcat中会话增量的具体由DeltaSession类实现，DeltaSession继承了StandardSession标准会话的所有特性且增加了会话增量记录的功能，增量记录功能即通过动作集实现，动作集被封装在DeltaRequest类，所以DeltaSession主要通过DeltaRequest实现动作集的管理，动作集由一个LinkedList<AttributeInfo>结构保存，AttributeInfo描述了动作的一些消息，所以一个动作就被抽象成了一个AttributeInfo对象，它主要包含四个属性 name(String)、value(Object)、action(int)、type(int)，name表示会话的属性名，即哪个属性被改；value表示会话属性名对应的值；action表示动作类型，可能是设置属性也可能是删除属性；type表示会话哪种类别的属性将被修改。

![img](https://img-blog.csdn.net/20180730171453623?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

整个增量会话的实现机制就是上面所说的，会话的增量拷贝比起全量拷贝有很多好处，即使实现相对比较复杂。

#### 增量会话管理器——StandardManager

用于保存状态的会话对象已经有了，现在就需要一个管理器来管理所有会话，例如会话id生成、根据会话id找出对应的会话、对于过期的会话进行销毁等等操作。

用一句话描述标准会话管理器：**提供一个专门管理某个****web****应用所有会话的容器，并且会在web****应用启动停止时刻进行会话重加载和持久化。**

会话管理主要提供的功能包括会话ID生成器、后台处理（处理过期会话）、持久化模块及会话集的维护。

![img](https://img-blog.csdn.net/20180730171532733?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

首先看会话ID生成器，它负责为每个会话生成分配一个唯一标识，例如最终会生成类似“326257DA6DB76F8D2E38F2C4540D1DEA”字符串的会话标识，具体的默认生成算法主要依靠jdk提供的SHA1PRNG算法，如果在集群环境中，为了方便识别会话归属，它最终生成的会话标识类似于“326257DA6DB76F8D2E38F2C4540D1DEA.tomcat1”，后面会加上tomcat集群标识jvmRoute变量值，这里假设其中一个集群标识配置为“tomcat1”。如果你想置换随机数生成算法，可以通过配置server.xml的manager节点secureRandomAlgorithm及secureRandomClass属性达到修改算法的效果。

其次看下如何对过期会话进行处理。负责对会话是否过期的逻辑判断主要在backgroundProcess模块，在tomcat容器中会有一条线程专门用于执行后台处理，当然也包括标准会话管理器的backgroundProcess，不断循环判断所有的会话中是否有过期的，一旦过期则从会话集中删除此会话。

最后是关于持久化模块和会话集的维护，由于标准会话定位于提供一个简单便捷的管理器，所以持久化和重加载操作并不会太灵活且扩展性弱，tomcat会在每个StandardContext（web应用）停止时调用管理器将属于此web应用的所有会话持久化到磁盘，文件名为SESSIONS.ser，而目录路径则由server.xml的Manager节点pathname指定或Javax.servlet.context.tempdir变量指定，默认存放路径为%CATALINA_HOME%/work/Catalina/localhost/web’name/SESSIONS.ser。当web应用启动时又会加载这些被持久化的会话，加载完成后SESSIONS.ser文件将会被删除，所以每次启动成功后就不会看到此文件的存在。另外会话集的维护是指提供创建新会话对象、删除指定会话对象及更新会话对象的功能。

标准会话管理器是我们常用的会话管理器，也是tomcat默认的一个会话管理器，对它进行深入了解有助于对tomcat会话功能的把握，同时对后面其他会话管理器的理解也更容易。

#### 持久化会话管理器——PersistentManager

前面提到的标准会话管理器已经提供了基础的会话管理功能，但在持久化方面做得还是不够，或者说在某些情景下无法满足要求，例如把会话以文件或数据库形式存储到存储介质中，这些都是标准会话管理器无法做到的，于是另外一种会话管理器被设计出来——持久化会话管理器。

在分析持久化会话管理器之前不妨先了解另外一个抽象概念会话存储设备Store，引入这个概念是为了更清晰方便地实现各种会话存储方式。作为存储设备最重要的操作无非就是读写操作，读即是将会话从存储设备加载到内存中，而写则将会话写入存储设备中，所以定义了两个重要的方法load和save与之相对应。FileStore和JDBCStore只要扩展Store接口各自实现load和save方法即可分别实现以文件或数据库形式存储会话。UML类图如下所示：

![img](https://img-blog.csdn.net/20180730171632851?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### 集群增量会话管理器——DeltaManager

DeltaManager会话管理器是tomcat默认的集群会话管理器，它主要用于集群中各个节点之间会话状态的同步维护。

**集群增量会话管理器的职责是将某节点的会话该变同步到集群内其他成员节点上，它属于全节点复制模式，所谓全节点复制是指集群中某个节点的状态变化后需要同步到集群中剩余的节点，非全节点方式可能只是同步到其中某个或若干节点。**在集群中全节点会话复制的一个大致步骤如下图所示，客户端发起一个请求，假设通过一定的负载均衡设备分发策略分到其中一个结点node1，如果还未存在session对象的话web容器将会创建一个会话对象，接着执行一些逻辑处理，在对客户端响应之前有个重要的事情是要把session对象同步到集群中其他节点上，最后再响应客户端。当客户端第二次发起请求时，假如分发到node3节点上，由于同步了node1的session会话，所以在执行逻辑时并不会取不到session的值。如果删除某个会话对象则要同时通知其他节点把相应会话删除，如果修改了某个会话的某些属性也同样要更新到其他节点的会话中。

 

![img](https://img-blog.csdn.net/20180730171719805?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

**DeltaManager****其实就是一个会话同步通信解决方案，除了具备上面提到的全节点复制外，它还有具有只复制会话增量的特性，增量是以一个完整请求为周期，即会将一个请求过程中所有会话修改量在响应前进行集群同步。**往下看Tomcat具体实现方案。

为区分不同的动作必须要先定义好各种事件，例如会话创建事件、会话访问事件、会话失效事件、获取所有会话事件、会话增量事件、会话ID改变事件等等，实际上tomcat集群会有9种事件，集群根据这些不同的事件就可以彼此进行通信，接收方对不同事件做不同的操作。如下图，例如node1节点创建完一个会话后，即向其他三个节点发送EVT_SESSION_CREATED事件，其他三个节点接收到此事件后则各自在自己本地创建一个会话，会话包含了两个很重要的属性——会话ID和创建时间，这两个属性都必须由node1节点跟着EVT_SESSION_CREATED一起发送出去，本地会话创建成功后即完成了会话创建同步工作，此时你通过会话ID查找集群中任意一个节点都可以找到对应的会话。同样对于会话访问事件，node1向其他节点发送EVT_SESSION_ACCESSED事件及会话ID，其他节点根据会话ID找到对应会话并更新会话最后访问时间，以免被认为是过期会话而被清理。类似的还有会话失效事件（同步集群销毁某会话）、会话ID改变事件（同步集群更改会话ID）等等操作。

 

 

![img](https://img-blog.csdn.net/2018073017172088?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

Tomcat使用SessionMessageImpl类定义了各种集群通信事件及操作方法，在整个集群通信过程中就是按照此类定义好的事件进行通信，SessionMessageImpl包含的事件如下{ EVT_SESSION_CREATED、EVT_SESSION_EXPIRED、EVT_SESSION_ACCESSED、EVT_GET_ALL_SESSIONS、EVT_SESSION_DELTA、EVT_ALL_SESSION_DATA、EVT_ALL_SESSION_TRANSFERCOMPLETE、EVT_CHANGE_SESSION_ID、EVT_ALL_SESSION_NOCONTEXTMANAGER }，除此之外它继承了序列化接口（方便序列化）、集群消息接口（集群的操作）、会话消息接口（事件定义及会话操作）。

 

![img](https://img-blog.csdn.net/2018073017172078?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

集群增量会话管理器DeltaManager可以说是通过SessionMessageImpl消息来管理DeltaSession，即根据SessionMessageImpl里面的事件响应不同的操作。Tomcat的集群通信使用的是tribes组件（相关章节会对tribes组件详细分析），网络IO都交由tribes后应用可以更专注逻辑处理，DeltaManager存在一个messageDataReceived(ClusterMessage cmsg)方法，此方法会在本节点接收到其他节点发送过来的消息后被调用，且传入的参数为ClusterMessage类型，可转化为SessionMessage类型，然后根据SessionMessage定义的9种事件做不同处理。其中有一个事件需要关注的是EVT_SESSION_DELTA，它是对会话增量同步处理的事件，某个节点在一个完整的请求过程中对某会话相关属性的所有操作被抽象到了DeltaRequest对象中，而DeltaRequest被序列化后会放到SessionMessage中，所以EVT_SESSION_DELTA事件处理逻辑就是从SessionMessage获取并反序列化出DeltaRequest对象，再将DeltaRequest包含的对某个会话的所有操作同步到本地该会话中，至此完成会话增量同步。

 

![img](https://img-blog.csdn.net/20180730171720399?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

DeltaManager就是DeltaSession的管理器，它提供了会话增量的同步方式而不是全量同步，极大提高了同步效率。

#### 集群备份会话管理器——BackupManager

全节点复制的网络流量随节点数量增加呈平方趋势增长，也正是因为这个因素导致无法构建较大规模的集群，为了使集群节点能更加大，首要解决的就是数据复制时流量增长的问题，于是tomcat提出了另外一种会话管理方式，每个会话只会有一个备份，它使会话备份的网络流量随节点数量的增加呈线性趋势增长，大大减少了网络流量和逻辑操作，可构建较大的集群。

#### Tomcat会话管理器的集成

Tomcat中所有的会话管理器包括标准的会话管理器、持久化会话管理器、集群增量会话管理器、集群备份会话管理器。它们为用户提供了各种功能的会话管理器，有非持久化模式的，也有持久化模式的，有集群全量复制模式的，也有集群备份模式。

​     在不同场景中，用户根据实际情况可以选择不同的会话管理器。为了方便使用，需要提供一种简易的方法，Tomcat提供的是配置方式，只需要通过对配置文件进行配置即可完成对会话管理器的选择。

在程序层面上，为了让会话管理器实现可配置化，它需要定义一个统计的管理接口——Manager接口。

在Tomcat的server.xml中可以实现对这四种会话管理器的配置；

### 高可用的集群实现

对于Web容器来说，在请求是无状态的情况下，如果实现做集群功能其实是非常简单的，只需要把机器连接到具备一定的分发策略的分发器上即可实现集群功能，同时也要保证这个分发器必须具备故障转移的能力。后面需要多少机器的集群直接添加即可，不但达到了负载均衡的效果，而且达到了高可用的效果；

实际情况中，很多请求都是有状态的，简单的讲，就是请求与会话有关。

Tomcat中使用了自己的Tribes组件实现集群之间的通信，集群通信都是重复性的工作并且独立性比较强，因此这块逻辑被Tomcat单独抽离了出来。对于Tomcat，它需要一个更高层次的抽象，而不是直接使用Tribes组件。所以这里就有了Cluster组件，Cluster组件将集群相关的所有组件都封装起来，以实现集群的功能。

#### 从单机到集群的会话管理

##### 单机模式

单机时代对会话的管理主要有两种方式——非持久化方式和持久化方式。非持久化方式指会话直接由tomcat管理并保存在机器内存上，它是最简单的方式，如下图，所有的会话集合都保存在内存上，客户端访问时根据自己的会话id直接在服务器内存中寻找，查找简单且速度快，但同时也存在两个缺点：一是容量比较小，当数据量大时容易导致内存不足；一是机器意外停止会导致会话数据丢失缺点。

 

![img](https://img-blog.csdn.net/20180730171911839?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

为了解决上面非持久化方式存在的缺陷，我们需要引入持久化机制，即持久化方式。可以将会话数据以文件形式持久化到硬盘中，也可以通过数据库持久化会话数据。首先看硬盘持久化，如下图，会话数据会以文件形式保存在硬盘中，由于硬盘比存储空间比内存大且机器意外关机都不会使数据丢失，所以硬盘存储解决了上面两个缺点，但是硬盘读取的速度比较慢，可能会影响整体的响应时间，硬盘持久化方式在实际中基本不会使用。

![img](https://img-blog.csdn.net/20180730171912108?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

Tomcat提供的另外一种默认的持久化方式就是将会话数据持久化到数据库上，所有会话数据交由数据库存储，tomcat通过jdbc数据库驱动并使用连接池技术去数据库指定表读取会话信息，此种方式解决了非持久化方式的所有缺点同时也对以文件方式存储方式的IO进行了优化，用数据库存储会话其实是一种集中管理模式，现在实际中更多是使用一个分布式缓存替代数据库，例如memcached、redis集群等，因为缓存的查询读取速度快，且集群解决了高可用的问题，但Tomcat官方版本是不提供会话保存到memcached或redis的支持，如要使用可自己编写一个会话管理器及一个阀门valve，或使用第三方jar包。需要说明的是集中管理模式不管是tomcat单机还是集群模式都可以使用。

 

![img](https://img-blog.csdn.net/20180730171912146?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

 

 

##### 集群模式

为什么要使用集群？主要有两方面原因：一是对于一些核心系统要求长期不能中断服务，为了提供高可用性我们需要由多台机器组成的集群；另外一方面，随着访问量越来越大且业务逻辑越来越复杂，单台机器的处理能力已经不足以处理如此多且复杂的逻辑，于是需要增加若干台机器使整个服务处理能力得到提升。

如果说一个web应用不涉及会话的话，那么做集群是相当简单的，因为节点都是无状态的，集群内各个节点无需互相通信，只需要将各个请求均匀分配到集群节点即可。但基本所有web应用都会使用会话机制，所以做web应用集群时整个难点在于会话数据的同步，当然你可以通过一些策略规避复杂的额数据同步操作，例如前面说到的把会话信息保存在分布式缓存或数据库中统一集中管理，如下图，每个tomcat实例只需去写入或读取数据库即可，避免了tomcat集群之间的通信。但这种方式也有不足，要额外引入数据库或缓存服务，同时也要保证它们的高可用性，增加了机器和维护成本。

 

![img](https://img-blog.csdn.net/20180730171912244?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**【使用缓存或者数据库统一保存会话信息】**

 

鉴于以上存在的不足，提供另一种解决思路就是tomcat集群节点自身完成各自的数据同步，不管访问到哪个节点都能找到对应的会话，如下图，客户端第一次访问生成会话，tomcat自身会将会话信息同步到其他节点上，而且是每次请求完成都会同步此次请求过程中对session的所有操作，这样一来下一次请求到集群中任意节点都能找到响应的会话信息，且能保证信息的及时性。细看很容易发现集群的节点之间的会话是两两互相复制的，一旦集群节点数量及访问量大起来，将导致大量的会话信息需要互相复制同步，很容易导致网络阻塞，而且这些同步操作很可能会成为整体性能的瓶颈，根据经验，此种方案在实际生产上推荐的集群节点个数为3-6个，无法组建更大的集群，而且冗余了大量的数据，利用率不高。

 

![img](https://img-blog.csdn.net/20180730171912272?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**【全节点复制集群】**

全节点复制的网络流量随节点数量增加呈平方趋势增长，也正是因为这个因素导致无法构建较大规模的集群，为了使集群节点能更加大，首要解决的就是数据复制时流量增长的问题，下面将介绍另外一种会话管理方式，**每个会话只会有一个备份**，它使会话备份的网络流量随节点数量的增加呈线性趋势增长，大大减少了网络流量和逻辑操作，可构建较大的集群。

下面看看这种方式具体的工作机制，集群一般是通过负载均衡对外提供整体服务，所有节点被隐藏在后端组成一个整体。**前面各种模式的实现都无需负载均衡协助，但现在讨论的集群方式则需要负载均衡器的协助。**所以图中都把负载均衡省略了。最常见的负载方式是前面用apache拖所有节点，它支持将类似“326257DA6DB76F8D2E38F2C4540D1DEA.tomcat1”的会话id进行分解，定位到tomcat集群中以tomcat1命名的节点上（这种方式称为Session Stick，由apache jk模块实现）。每个会话存在一个原件和一个备份，且备份与原件不会保存在同一个节点上，如下图，例如当客户端发起请求后通过负载均衡被分发到tomcat1实例节点上，生成一个包含.tomcat1后缀的会话标识，并且tomcat1节点根据一定策略选出此次会话对象备份的节点，然后将包含了{会话id，备份ip}的信息发送给tomcat2、tomcat3、tomcat4，如图中虚线所示，这样每个节点都有一个会话id、备份ip列表，即每个节点都有每个会话的备份ip地址。

完成上面一步后就是将会话内容备份到备份节点上，假如tomcat1的s1、s2两个会话的备份地址为tomcat2，则把会话对象备份到tomcat2中，类似的有tomcat2把s3会话备份到tomcat4，tomcat4把s4、s5两个对话备份到tomcat3，这样集群中所有的会话都已经有了一份备份。当tomcat1一直不出故障，由于Session Stick技术客户端将一直访问到tomcat1节点上，保证一直能获取到会话。而当tomcat1出故障了，这时tomcat也提供了一个failover机制，apache感知到后端集群tomcat1节点被移除了，这时它会把请求随机分配到其他任意节点上，接下去会有两种情况：

①刚好分到了备份节点tomcat2上，此时仍能获取到s1会话，除此之外，tomcat2还要另外做的事是将这个s1会话标记为原件且继续选取一个备份地址备份s1会话，这样一来又有了备份。

②假如分到了非备份节点tomcat3，此时肯定找不到s1会话，于是它将向集群所有节点发问，“请问谁有s1会话的备份ip地址信息？”，因为只有tomcat2有s1的备份地址信息，它接收到询问后应答告知tomcat3节点s1会话的备份在tomcat2，根据这个信息就能查到s1会话了，并且tomcat3在自己本地生成s1会话并标为原件，tomcat2上的副本不变，这样一来同样能找到s1会话，正常完整整个请求处理。

 

![img](https://img-blog.csdn.net/20180730171912293?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### Cluster组件

Cluster其实就是集群的意思，它是为了更加方便上层调用而抽象出来的一个比较高的层次的一个概念。总的来讲，它最重要的两个接口就是发送和接收接口。对于Cluster来讲，它可以让你不必关心集群之间如何通信、与谁通信，你只要调用Cluster接口将消息发送出去即完成了集群内消息的传递。

Cluster组件包含：会话管理器、集群通信通道、部署器（Deployer）、集群监听器（ClusterListener）、集群阀门等。

#### Tomcat的Cluster组件

Tomcat集群组件对一个请求的大致流程处理过程如下：

1. 客户端发送请求之后被Tomcat接收，成为一个request对象传入Engine容器准备处理。
2. 首先通过JvmRouteBinderValue进行处理，对不符合的会话ID进行处理，最后再调用下一个阀门。
3. 通过Replication Value阀门继续处理，它会先调下一个阀门进行处理，之后才进行会话数据集群同步；
4. StandardEngineValue调用其他子容器对请求进行处理，找到对应的Servlet处理；
5. 响应客户端。
6. Replication Value调用Cluster组件将会话数据同步到集群的其他实例上；
7. 把会话数据传输给其他的实例；

#### Tomcat中Cluster的级别

Tomcat中的集群组件可以分为两个级别，分别为Engine级别和Host级别，即Cluster组件可以放到Engine容器中，也可以放到Host容器中。这也就意味着，如果是Engine级别，则整个集群组件由所有Host共享，如果是Host级别，则由该Host专享，其他的Host无法使用；

#### 如何让Tomcat实现集群功能

在Tomcat中使用集群功能相对简单，最简单的用法是直接在server.xml文件的<Engine>或者<Host>节点下添加<Cluster calssName=”org.apache.catalina.ha.tcp.SimpleTcpCluster”>配置，这意味着集群相关的配置都是使用默认的。

默认情况下，会使用DeltaManager作为会话管理器，使用GroupChannel作为集群通信通道，

组播地址和端口为228.0.0.4和45564；会使用ReplicationTransmitter作为消息发射器；会使用NioReceiver作为消息接收器。会添加TCPFailureDetector和MessageDispatcher15Interceptor两个拦截器；会使用ReplicationValue和JvmRouteBinderValue两个管道阀门；会使用FarmWarDeployer作为集群部署器；

还会添加JvmRouteSessionIDBinderListener和ClusterSessionListener集群监听器；

### 集群通信框架

- 现在如果要构造一个真正在生产环境上可使用的可靠的系统，基本都离不开集群的概念，总的来说集群是指由若干互相独立的机器通过高速网络组成的一个整体服务，整个集群的内部实现相对外部是透明的，对外部而言它就像一个独立的服务器。要使若干机器协同工作不是一件简单的事，其核心是如何在多机器之间进行通信及各种任务调度使之协同工作。
- 在工程上常见的两种集群是——负载均衡集群和高可用集群。
- 负载均衡集群（Load Balance Cluster），随着系统的处理量的不断增加，很快到达一台机器的处理极限，所以需要更多的机器来分担处理，负载均衡集群一般是通过一定的分发算法把访问流量均匀分布到集群里面的各个机器上进行处理，理想的集群是通过添加机器使处理能力达到线性增长，但现实中往往处理过程并非是无状态的，会涉及到一些共享状态变量，所以当集群数量达到一定程度后处理能力并不能按线性增长且可靠性可能也会降低。关于负载均衡集群如何协调分配分发请求的问题，即可以使用专门的负载均衡硬件如F5，也可以使用软件级别的方式实现负载均衡如LVS、HAProxy等。
- 高可用集群（High Available Cluster），高可用集群通过软件把若干机器连接起来，这种集群更偏重的是当集群中某个机器发生故障后能通过自动切换或流量转移等措施来保证整个集群对外的可用性。在实际运用中很经典的就是mysql数据库的主从节点，一般情况如果是一主多从我们会使用读写分离措施，主节点负责执行写操作而从节点执行读操作，一旦主节点发生故障后系统将自动从多个从节点中选举出一个节点作为主节点继续对外提供服务，保证了服务的高可用。高可用集群与负载均衡集群一般不会绝对地划分开，我们不会将一台机器什么也不做就放着就等其他机器出故障再去候补，为了使用地更加高效可以把这些备用（standby）的机器用于负载均衡，故障发生后再接管故障节点的职责。

#### Tribes

Tribes是一个具备让你通过网络向组成员发送和接收信息、动态检测发现其他节点的组通信能力的高扩展性的独立的消息框架。在组成员之间进行信息复制及成员维护是一个相对复杂的事情，因为不仅要考虑各种通信协议还要有必要的机制提供不同的消息传输保证级别，且成员关系的维护要及时准确，同时针对IO不同场景需提供不同的IO模式，这些都是组成员消息传输要遇到的需要深入考虑的几点。而Tribes很好地将点对点、点对组的通信抽象得即简单又相对灵活。

#### 集群成员维护服务——MembershipService

一个集群包含若干成员，要对这些成员进行管理就必须要有一张包含所有成员的列表，当要对某个节点做操作时通过这个列表可以准确找到该节点的地址进而对该节点发送操作消息。如何维护这张包含所有成员的列表是本节要讨论的主题。

成员维护是集群的基础功能，一般划分一个独立模块或层完成此功能，它提供成员列表查询、成员维护、成员列表改变事件通知等能力。由于tribes定位于基于同等节点之间的通信，所以并不存在主节点选举的问题，它所要具备的功能是自动发现节点，即新节点加入要通知集群其他成员更新成员列表，让每个节点都能及时更新成员列表，每个节点都维护一份集群成员表。如图，节点1、节点2、节点3使用组播通过交换机各自已经维护一份成员列表，且他们隔一段时间向交换机组播自己节点消息，即心跳操作。当第四个节点加入集群组，节点四向交换机组播自己的节点消息，原理三个节点接收到后各自把节点四加入到各自的成员列表中，而原来三个节点也不断向交换机发送节点消息，节点四接收到后依次更新成员列表信息，最终达到四个节点都拥有四个节点成员信息。

![img](https://img-blog.csdn.net/20180730172101146?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

看下tribes的集群是如何设计实现以上功能的，其成员列表的创建维护是基于经典的组播方式实现，每个节点都创建一个节点信息发射器和节点信息接收器，让他们运行于独立的线程中。发射器用于向组内发送自己节点的消息，而接收器则用于接收其他节点发送过来的节点消息并进行处理。要使节点之间通信能被识别就需要定义一个语义，即约定报文协议的结构，tribes的成员报文是这样定义的，两个固定值用于表示报文的开始和结束，开始标识TRIBES_MBR_BEGIN 的值为字节数组84, 82, 73, 66, 69, 83, 45, 66, 1, 0，结束标识TRIBES_MBR_END的值为字节数组84, 82, 73, 66, 69, 83, 45, 69, 1, 0，整个协议包结构为：**开始标识（*****\*10bytes\*******\*）\*******\*+\*******\*包长度（\*******\*4bytes\*******\*）\*******\*+\*******\*存活时间（\*******\*8bytes\*******\*）\*******\*+tcp\*******\*端口（\*******\*4bytes\*******\*）\*******\*+\*******\*安全端口（\*******\*4bytes\*******\*）\*******\*+udp\*******\*端口（\*******\*4bytes\*******\*）\*******\*+host\*******\*长度（\*******\*1byte\*******\*）\*******\*+host\*******\*（\*******\*nbytes\*******\*）\*******\*+\*******\*命令长度（\*******\*4bytes\*******\*）\*******\*+\*******\*命令（\*******\*nbytes\*******\*）\*******\*+\*******\*域名长度（\*******\*4bytes\*******\*）\*******\*+\*******\*域名（\*******\*nbytes\*******\*）\*******\*+\*******\*唯一会话\*******\*id\*******\*（\*******\*16bytes\*******\*）\*******\*+\*******\*有效负载长度（\*******\*4bytes\*******\*）\*******\*+\*******\*有效负载（\*******\*nbytes\*******\*）\*******\*+\*******\*结束标识（\*******\*10bytes\*******\*）\****。成员发射器按照协议组织成包结构并组播，接收器接收包并按照协议进行解包，根据包信息维护成员表。

第一步要先执行加入组播成员操作，接着分别启动接收器线程、发射器线程，一般接收器要优先启动。发射器每隔1秒组织协议包发送心跳，组播组内成员的接收器对接收到的协议报文进行解析，按照一定的逻辑更新各自节点本地成员列表，如果成员表已包含协议包的成员则只更新存活时间等消息。

Tribes利用上述原理维护集群成员，并且由独立模块MembershipService提供成员的相关服务，例如获取集群所有成员相关信息等。

#### 平行的消息发送通道——ChannelSender

前面的集群成员维护服务为我们提供了集群内所有成员的地址端口等信息，可以通过MembershipService可以轻易从节点本地的成员列表获取集群所有的成员信息，有了这些成员信息后就可以使用可靠的TCP/IP协议进行通信了。这节讨论的正是实际中真正用于消息传送通道的相关机制及实现细节。

 

如下图，四个节点本地都拥有了一张集群成员的信息列表，这时节点1有这么一个需求：为了保证数据的安全可靠，在往自己的内存存放一份数据的同时还要同步到其他三个节点的内存中。节点1有一个专门负责发送的组件ChannelSender，首先从成员列表中获取其他三个节点的通信地址和端口，再分别向此三个节点建立TCP/IP通道发送数据，其他节点有一个负责接收数据的服务，将数据更新到内存中。

 

![img](https://img-blog.csdn.net/20180730172101119?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

最理想的状态是发送给节点2、3、4都成功了，这样从整体看来ChannelSender像是提供了一个多通道的平行发送方式，所以也称为平行发送器。但现实中并不能保证同一批的消息往所有节点发送都成功，有可能发送到节点3时因为某种原因失败了，而节点2、4都成功了，这时通常要采取一些策略来应对，例如重新发送。Tribes所使用的策略是优先进行若干次尝试发送，若干次失败后将向上抛出异常信息，异常信息包含哪些节点发送失败及其原因，默认的尝试次数是1次。

 

***\*为确保数据确实被节点接收到，需要在应用层引入一个协议保证传输的可靠性，即是通知机制，发送者发送消息给接收者，接受者接收到后返回一个\*******\*ACK\*******\*表示自己已经接收成功。\****Tribes中详细的协议报文定义如下：START_DATA（7bytes）+消息长度（4bytes）+消息长度（nbytes）+END_DATA（7bytes）。START_DATA为数据开始标识，为固定数组值70,76,84,50,48,48,50，END_DATA为数据结束标识，为固定数组值84,76,70,50,48,48,51，ACK_DATA表示通知报文，为固定数组值6,2,3。所以如果传输的是通知报文的话即为START_DATA+ACK_DATA的长度+ACK_DATA+END_DATA。所以整个集群的消息同步如下图，节点1通过ChannelSender发送消息给节点2、3、4，发送成功的判定标准就是节点2、3、4返回给节点1一个ack标识，节点1接收到了则认为发送成功。

 

![img](https://img-blog.csdn.net/20180730172101162?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

为提高通信效率这里默认使用了NIO模式而非BIO模式（也可设置为BIO模式），使用NIO模式能统一管理所有通信的channel，避免了等待一个通道发送完毕另一个通道才能发送，如果逐个通信将导致阻塞IO时间很长通信效率低下。另外平行发送的过程需要一个锁保证消息的正确发送，例如有data1、data2、data3三个数据需要发送，应该是一个接一个数据包发送的而不能data1发一部分data2发一部分。

#### 消息接收通道——ChannelReceive

消息发送通道对应，发送的消息需要一个接收端接收消息，它就是ChannelReceiver。接收端负责接收处理其他节点从消息发送通道发送过来的消息，实际情况如图每个节点都有一个ChannelSender和ChannelReceiver，ChannelSender向其他节点的ChannelReceiver发送消息。本质是每个节点暴露一个端口作为服务端监听客户端，而每个节点又充当客户端连接其他节点的服务端，所以ChannelSender就是充当客户端的集合，ChannelReceiver充当服务端。

 

![img](https://img-blog.csdn.net/20180730172101192?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

集群消息复制过程中，每个节点ChannelReceiver负责接收来自其他节点的消息，假设一个n节点的集群，一般情况下每个ChannelReceiver对应n-1个连接，因为集群之间的通信连接都是长连接，长连接有助于提高通信效率，如下图，4个节点的集群，node1的ChannelReceiver的客户端连接数为3，分别是node2、node3、node4三个节点作为客户端发起的socket连接。这三个节点产生的数据会通过此通道同步到node1节点，同样地，node2的ChannelReceiver拥有node1、node3、node4的客户端连接，这三个节点产生的数据也会同步到node2节点。Node3、node4也拥有三个客户端连接。为提高处理效率，此处还是使用NIO处理模型。

![img](https://img-blog.csdn.net/20180730172101151?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

除此之外，再接收操作中为了优化性能采取了很多措施，例如引入任务池，即是把接收任务提前定义好放入内存中，接收时可直接获取使用而不用再实例化；例如一次获取若干个报文进行处理，即使用nio模式读取消息到缓冲区后直接处理整个缓冲区的消息，它可能包含若干个报文；网络IO需要优化的地方及手段都比较多，tribes确实已经做了很多优化方面的工作。

#### 通道拦截器——ChannelInterceptor

拦截器应该可以说是一个很经典的设计模式，它有点类似于过滤器，当某信息从一个地方流向目的地的过程中，可能需要统一对信息进行处理，如果考虑到系统的可扩展性和灵活性通常就会使用拦截器模式，它就像一个个关卡被设置在信息流动的通道中，并且可以按照实际需要添加和减少关卡。**Tribes****为了在应用层提供对源消息统一处理的渠道引入通道拦截器**，用户在应用层只需要根据自己需要添加拦截器即可，例如，压缩解压拦截器、消息输出输入统计拦截器、异步消息发送器等等。

 

拦截器的数据流向示意图可以参考前面的tribes简介章节，数据从IO层流向应用层，中间就会经过一个拦截器栈，应用层处理完就会返回一个ack给发送端表示已经接收并处理完毕（消息可靠级别为SYNC_ACK），，最底层的协调者ChannelCoordinator永远作为第一个加入拦截器栈的拦截器，往上则是按照添加顺序排列，且每个拦截器的previous、next分别指向前一个拦截器和下一个拦截器。

 

![img](https://img-blog.csdn.net/20180730172101548?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 

 Tribes的拦截器整体设计就如上面，整个拦截器的执行顺序如下，当执行写操作时，数据流向GzipInterceptor -> ChannelCoordinator -> Network IO；当执行读操作时，数据流向则为Network IO -> ChannelCoordinator -> GzipInterceptor。

#### 应用层处理入口——MembershipListener与ChannelListener

Tribes为了更清晰更好地划分职责，它被设计成用IO层和应用层，IO层专心负责网络传输方面的逻辑处理，把接收到的数据往应用层传送，当然应用层发送的数据也是通过此IO层发送，

数据传往应用层后必须要留一些处理入口供应用层进行逻辑处理，而考虑系统解耦，这个入口最好的方式是使用**监听器模式**，在底层发生各种事件时触发所有安装好的监听器，使之执行监听器里面的处理逻辑。这些事件主要包含了集群成员的加入和退出、消息报文接收完毕等信息，所以整个消息流转过程被分成两类监听器：

***\*一类是跟集群成员的变化相关的监听器\*******\*MembershipListener\*******\*；\****

***\*另外一类是跟集群消息接收发送相关的监听器\*******\*ChannelListener\*******\*。\****

应用层只要关注这两个接口就行了，写好各种处理逻辑的监听器添加到通道中即可。

 

![img](https://img-blog.csdn.net/20180730172101689?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2ODA3ODYy/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**【应用层入口】**

我们可以在应用层自定义若干监听器并且添加到GroupChannel中的两个监听器列表中，GroupChannel其实可以看成是一个封装了IO层的抽象容器，它会在各个适当的时期遍历监听器列表中的所有监听器并调用监听器对应的方法，即执行应用层定义的业务逻辑，至此完成了数据从IO层流向应用层并完成处理。

两种类型的监听器给应用层提供了处理入口，应用层只需关注逻辑处理，而其他的IO操作则交由IO层，**这两层通过监听器模式串联起来，优雅地将模块解耦**。

#### 如何使用Tribes进行数据传输

#### Tomcat使用Tribes同步会话

Tomcat集群中最重要的交换信息就是会话消息，对某个Tomcat实例某会话做的更改，要同步到集群中其他Tomcat实例的该会话对象上，这样才能保证集群中所有的实例的会话数据保持一致。

如果集群中有一个Tomcat实例的会话变了，它会通过会话管理器将改变的动作消息封装成消息然后调用集群对象Cluster，通过Cluster将消息发送出去，同时Cluster有依赖于Tribes。最后消息其实就是交由Tribes真正发送出去的。通信过程是以ClusterMessage为对象进行传输的，它会先序列化再进行传输，到其他Tomcat实例的时候会反序列化，消息由Tribes接收之后向Cluster上传。最后达到会话管理器，它根据动作消息同步会话；

所以Cluster其实就是实现了ChannelListener的监听器，当Tribes接收到消息之后，就会调用此监听器的messageReceived方法处理逻辑，此方法又会继续向上通知Manager的messageDataReceived方法，在此方法内完成会话同步处理逻辑；

#### Tomcat使用Tribes部署集群应用

集群应用部署是一个很重要的应用场景，设想一下如果没有集群应用部署功能，每当我们发布应用时都要登陆每台机器对每个tomcat实例进行部署，这些工作量都是繁杂且重复的。于是需要一种功能可以在集群中某实例部署后，集群中的其他tomcat实例会自动完成部署。

集群部署主要分两部分内容。

- 第一部分是关于应用传输问题，主要是关于在tomcat中如何一个web应用传输到其它tomcat实例上；
- 第二部分是应用部署方式及应用更新方式，主要关于在tomcat中如何以集群同步方式部署一个web应用，以及集群实例在接收到新版本web应用时如何进行更新。

tomcat的集群是基于tribes网络框架的。

#### 监控与管理

Tomcat在对内部监控上主要使用了JMX。JMX即Java管理扩展（Java Management Extension），作为一个Java管理体系的规范标准，其主要负责系统管理，是基于此规范而扩展的系统拥有管理监控功能。通过它对Tomcat运行时进行监控和管理，包括服务器性能、JVM相关性能、Web连接数、线程池、数据库连接池、配置文件重新加载等。并且提供了一些远程可视化管理。它实时性高，同时也为分布式系统的管理提供了一个基础框架，提供了较为丰富的管理手段；

#### Java管理扩展——JMX

总体来讲，JMX体系结构分为三个层次：

- 设备层：Mbean

主要定义了信息模型，在JMX中，各种管理对象以管理构件的形式存在，需要管理的时候，向Mbean服务器进行注册，它定义了如何实现JMX管理资源的规范，只要将资源配置入JMX框架中就可以成为JMX一个管理构件（MBean）。资源可以是一个java应用，一个服务或者一个设备。另外，该层还定义了一个通知机制以及一些辅助元数据类；

- 代理层：MbeanServer

主要定义了各种服务以及通信模型。所有的构件需要向一个核心的Mbean服务注册，才可以被管理。

- 分布服务层：HTML适配器、SNMP适配器、RMI适配器

它是JMX架构的最外一层，它负责使JMX代理对外界可用，主要定义了能够对代理层进行操作的管理接口和构件，具体内容依靠适配器实现，这样外部管理者就可以操作代理。

## Nginx

[Nginx中文文档](https://www.nginx.cn/doc/)

### 核心模块

#### 主模块

这里是控制 Nginx 的基本功能的指令.

#### 事件模块

### 基本模块

#### http核心模块

#### HttpIndex模块

这个模块提供一个简单方法来实现在轮询和客户端IP之间的后端服务器负荷平衡。

__示例__

```
upstream backend  {
  server backend1.example.com weight=5;
  server backend2.example.com:8080;
  server unix:/tmp/backend3;
}
 
server {
  location / {
    proxy_pass  http://backend;
  }
}
```

#### HttpAccess模块

此模块提供了一个简易的基于主机的访问控制.

ngx_http_access_module 模块使有可能对特定IP客户端进行控制. 规则检查按照第一次匹配的顺序

__示例__

```
location / {
: deny    192.168.1.1;
: allow   192.168.1.0/24;
: allow   10.1.1.0/16;
: deny    all;
}
```

在上面的例子中,仅允许网段 10.1.1.0/16 和 192.168.1.0/24中除 192.168.1.1之外的ip访问.

当执行很多规则时,最好使用 ngx_http_geo_module 模块.

#### HttpAuthBasic模块

该模块可以使你使用用户名和密码基于 HTTP 基本认证方法来保护你的站点或其部分内容。

__实例配置__

```
location  /  {
: auth_basic            "Restricted";
: auth_basic_user_file  conf/htpasswd;
}
```

#### HttpAutoindex模块

此模块用于自动生成目录列表.

ngx_http_autoindex_module只在 ngx_http_index_module模块未找到索引文件时发出请求.

__示例__

```
location  /  {
: autoindex  on;
}
```

#### Browser模块

本模块的变量基于请求头(header)中的"User-agent":

#### Charset模块

此模块将文本编码添加到“指示的Content-Type”响应标题中。

此外，模块可以将一种编码的数据重新编码为另一种。 需要注意的是，重新编码仅在一个方向上完成-从服务器到客户端，并且只能对一个字节的编码进行重新编码。

#### HttpEmptyGif模块

本模块在内存中常驻了一个 1x1 的透明 GIF 图像，可以被非常快速的调用。

__示例__

```
location = /_.gif {
: empty_gif;
}
```

#### HttpFcgi模块

这个模块允许Nginx 与FastCGI 进程交互，并通过传递参数来控制FastCGI 进程工作。

FastCGI是一个可伸缩地、高速地在HTTP服务器和动态脚本语言间通信的接口（FastCGI接口在Linux下是socket（可以是文件socket，也可以是ip socket）），主要优点是把动态语言和HTTP服务器分离开来。多数流行的HTTP服务器都支持FastCGI，包括Apache、Nginx和lightpd。

__示例__

```
location / {
  fastcgi_pass   localhost:9000;
  fastcgi_index  index.php;

  fastcgi_param  SCRIPT_FILENAME  /home/www/scripts/php$fastcgi_script_name;
  fastcgi_param  QUERY_STRING     $query_string;
  fastcgi_param  REQUEST_METHOD   $request_method;
  fastcgi_param  CONTENT_TYPE     $content_type;
  fastcgi_param  CONTENT_LENGTH   $content_length;
}
```

#### Geo模块

该模块创建变量，其值取决于客户端的IP地址。

__示例__

```
geo  $geo  {
  default          0;
  127.0.0.1/32     2;
  192.168.1.0/24   1;
  10.1.0.0/16      1;

}
```

#### HttpGzip模块

这个模块支持在线实时压缩输出数据流

__示例__

```
: gzip             on;
: gzip_min_length  1000;
: gzip_proxied     expired no-cache no-store private auth;
: gzip_types       text/plain application/xml;
```

内置变量 $gzip_ratio 可以获取到gzip的压缩比率

#### HttpHeaders模块

本模板可以设置HTTP报文的头标。

__示例__

```
: expires     24h;
: expires     0;
: expires     -1;
: expires     epoch;
: add_header  Cache-Control  private;
```

#### HttpIndex模块

该指令用来指定用来做默认文档的文件名，可以在文件名处使用变量。 如果您指定了多个文件，那么将按照您指定的顺序逐个查找。 可以在列表末尾加上一个绝对路径名的文件。

__示例__

```
index  index.$geo.html  index.0.html  /index.html;
```

#### HttpReferer模块

此模块可以使用请求标头中“ Referer”行的错误值来阻止对站点的访问。

请记住，很容易欺骗此标头。 因此，使用此模块的目的不是100％阻止这些请求，而是阻止典型浏览器发出的大量请求。 另外，请考虑即使对于正确的请求，典型的浏览器也不总是提供“ Referer”标头。

__示例__

```
location /photos/ {
  valid_referers none blocked www.mydomain.com mydomain.com;
  if ($invalid_referer) {
    return   403;
  }
}
```

#### HttpLog模块

__示例__

```
log_format  gzip  '$remote_addr - $remote_user [$time_local]  '
: '"$request" $status $bytes_sent '
: '"$http_referer" "$http_user_agent" "$gzip_ratio"';
access_log  /spool/logs/nginx-access.log  gzip  buffer=32k;
```

#### map

该模块允许您将一组值分类或映射到一组不同的值中，并将结果存储在变量中。

__示例__

```
map  $http_host  $name  {
  hostnames;
  default          0;
  example.com      1;
  *.example.com    1;
  test.com         2;
  *.test.com       2;
  .site.com        3;
}
```

一种用途是使用映射代替编写大量服务器/位置指令或重定向：

__示例__

```
map $uri $new {
  default        http://www.domain.com/home/;
  /aa            http://aa.domain.com/;
  /bb            http://bb.domain.com/;
  /john          http://my.domain.com/users/john/;
}
server {
  server_name   www.domain.com;
  rewrite  ^    $new   redirect;
}
```

#### Memcached

你可以利用本模块来进行简单的缓存以提高系统效率。本模块计划在未来进行扩展。

__示例__

```
server {
: location / {
: set  $memcached_key  $uri;
: memcached_pass   name:11211;
: default_type     text/html;
: error_page       404 = /fallback;
: }

: location = /fallback {
: proxy_pass       backend;
: }
}
```

#### HttpProxy模块

此模块专伺将请求导向其它服务.

这是种 HTTP/1.0 版本的无请求保持代理.

(因为每个请求都是在后台连接中创建和销毁的) Nginx 和浏览器使用 HTTP/1.1 进行对话，而在后台服务中使用 HTTP/1.0;

__示例__

```
location / {
: proxy_pass        http://localhost:8000;
: proxy_set_header  X-Real-IP  $remote_addr;
}
```

注意一点,当使用HTTP PROXY 模块时(或者甚至是使用FastCGI时),用户的整个请求会在nginx中缓冲直至传送给后端被代理的服务器.因此,上传进度的测算就会运作得不正确,如果它们通过测算后端服务器收到的数据来工作的话

#### HttpRewrite模块

该模块允许使用正则表达式改变URI，并且根据变量来转向以及选择配置。

如果在server级别设置该选项，那么他们将在location之前生效。如果在location还有更进一步的重写规则，location部分的规则依然会被执行。如果这个URI重写是因为location部分的规则造成的，那么location部分会再次被执行作为新的URI。

这个循环会执行10次，然后Nginx会返回一个500错误。

#### HttpSSI模块

此模块处理服务器端包含文件(ssi)的处理. 列表中的命令当前并未完全支持.

__示例__

```
location / {
: ssi  on;
}
```

#### HttpUserId

ngx_http_userid_module模块发出用于标识客户端的cookie。 为了进行记录，可以使用变量

```
$uid_got
$uid_set
```

在SSI中不能访问变量$ uid_got和$ uid_set，因为SSI过滤器模块在userid过滤器之前起作用。 该模块与Apache的mod_uid兼容。

__示例__

```
userid          on;
userid_name     uid;
userid_domain   example.com;
userid_path     /;
userid_expires  365d;
userid_p3p      'policyref="/w3c/p3p.xml", CP="CUR ADM OUR NOR STA NID"';
```

### 其他模块

#### Addition模块

此模块在当前位置的响应之前和之后添加其他位置的响应。

它被实现为输出过滤器，到其他位置的主请求和子请求的内容没有完全缓冲，仍然以流方式发送到客户端。 由于发送HTTP标头时最终响应主体的长度未知，因此此处始终使用HTTP分块编码。

#### EmbeddedPerl

该模块可以直接在Nginx中执行Perl，并通过SSI调用Perl。

#### flv

本模块提供FLV文件加载基于时间位移.

模块ngx_http_flv_module提供特殊处理的文件,它处理:

- 添加 FLV 头请求的文件;
- 传输文件，从开始的位移，在请求参数中指定 start=XXX.

本模块必需在编译nginx时加上--with-http_flv_module.

__示例__

```
location ~ \.flv$ {
  flv;
}
```

#### HttpGzipStatic

在开始压缩创建硬盘上的文件之前，本模块将查找同目录下同名的.gz压缩文件，以避免同一文件再次压缩。

nginx 0.6.24开始加入ngx_http_gzip_static_module . 编译时加上:

```
./configure --with-http_gzip_static_module
```

__示例__

```
gzip_static on;
gzip_http_version   1.1;
gzip_proxied        expired no-cache no-store private auth;
gzip_disable        "MSIE [1-6]\.";
gzip_vary           on;
```

#### RandomIndex

从目录中选择一个随机目录索引。 .

__示例__

```
location  /  {
  random_index  on;
}
```

#### HttpGeoIP

本模块ngx_http_geoip_module的变量基于IP地址匹配[MaxMind](http://www.maxmind.com/) GeoIP 二进制文件. 这个模块开始出现在nginx0.8.6。

首先，模块必需有geo数据库和读取数据库类

```
#下载免费的geo_city数据库
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
#下载免费的geo_coundty数据库
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCountry/GeoIP.dat.gz
#在debian中安装libgeoip:
sudo apt-get install libgeoip-dev
#其它系统，你可以下载并编译一个源文件
wget http://geolite.maxmind.com/download/geoip/api/c/GeoIP.tar.gz
```

在centos可以用yum安装:

```
yum install geoip
```

__编译__

```
./configure --with-http_geoip_module
```

__示例__

```
http {
    geoip_country  GeoIP.dat;
    geoip_city     GeoLiteCity.dat;
    ...
```

#### HttpRealIp

该模块可以将客户端的IP地址更改为请求标头中的值 (e. g.`X-Real-IP`or`X-Forwarded-For`).

如果nginx在L7负载平衡的某些代理后面工作，并且请求来自本地IP，但是代理使用客户端的IP添加请求标头，则很有用。

默认情况下未构建此模块，请使用configure选项启用它

```
--with-http_realip_module
```

用户注意：“您将建立一个受信任代理的列表（请参见下文），并且标头中第一个不受信任的IP将用作客户端IP。” 来源：Apache模块[mod_extract]（http://web.warhound.org/mod_extract_forwarded/README）的自述文件。 关于为什么以及如何使用此安全功能很有帮助的信息。

__示例__

```
set_real_ip_from   192.168.1.0/24;
set_real_ip_from   192.168.2.1;
real_ip_header     X-Real-IP;
```

#### HttpSSL

T他的模块启用HTTPS支持。

它支持检查客户端证书，但有两个限制：

-无法分配已废除证书的列表（吊销列表）
-如果您拥有链证书文件（有时称为中间证书），则不必像在Apache中那样单独指定它。相反，您需要将链证书中的信息添加到主证书文件的末尾。这可以通过在命令行上键入“ cat chain.crt >> mysite.com.crt”来完成。完成此操作后，您将不再使用链证书文件，只需将Nginx指向主证书文件即可。

默认情况下，不构建模块，必须指定使用./configure的*-with-http_ssl_module *参数构建模块。构建此模块需要OpenSSL库和包含文件，它们通常是单独软件包中的必要文件。

以下是示例配置，为了减少CPU负载，建议仅运行一个工作进程并启用保持活动连接：

```
worker_processes 1;
http {
  server {
    listen               443;
    ssl                  on;
    ssl_certificate      /usr/local/nginx/conf/cert.pem;
    ssl_certificate_key  /usr/local/nginx/conf/cert.key;
    keepalive_timeout    70;
  }
}
```

使用链式证书时，只需将额外的证书附加到您的.crt文件中（示例中为cert.pem）。 您自己的证书必须位于文件顶部，否则密钥与密钥不匹配。

#### StubStatus模块

这个模块能够获取Nginx自上次启动以来的工作状态

此模块非核心模块，需要在编译的时候手动添加编译参数`--with-http_stub_status_module`

__示例__

```
location /nginx_status {
: # copied from http://blog.kovyrin.net/2006/04/29/monitoring-nginx-with-rrdtool/
: stub_status on;
: access_log   off;
: allow SOME.IP.ADD.RESS;
: deny all;
}
```

#### HttpSubstitution

本模块可以在nginx的回应中查找和替换文本.在编译nginx时必需加上--with-http_sub_module option

__示例__

```
location / {
  sub_filter      </head>
  '</head><script language="javascript" src="$script"></script>';
  sub_filter_once on;
}
```

#### HttpDav模块

这个模块可以为Http webDAV 增加 PUT, DELETE, MKCOL, COPY 和 MOVE 等方法。

这个模块在默认编译的情况下不是被包含的，你需要在编译时指定如下参数：

```
./configure --with-http_dav_module
```

__示例__

```
location / {
  root     /data/www;
  client_body_temp_path  /data/client_temp;
  dav_methods  PUT DELETE MKCOL COPY MOVE;
  create_full_put_path   on;
  dav_access             group:rw  all:r;
  limit_except  GET {
    allow  192.168.1.0/32;
    deny   all;
  }
}
```

#### GooglePerftools

此模块可为工作人员启用Google Performance Tools性能分析。 此模块出现在nginx版本0.6.29中。默认情况下，未构建该模块，因此必须使用以下命令启用其构建./configure --with-google_perftools_module。

__示例__

```
google_perftools_profiles /path/to/profile;
```

配置文件将存储为/ path / to / profile.<worker_pid>

#### HttpXSLT

该模块是一个过滤器，可借助一个或多个XSLT模板转换XML响应。

此模块在0.7.8中引入，需要通过启用

```
./configure --with-http_xslt_module
```

__示例__

```
location / {
  xml_entities       /site/dtd/entities.dtd;
  xslt_stylesheet    /site/xslt/one.xslt   param=value;
  xslt_stylesheet    /site/xslt/two.xslt;
}
```

#### HttpSecureLink

这个模块计算和检测URL请求中必须的安全标识
这个模块没有默认编译，在编译Nginx时，必须使用明确的配置参数

```
--with-http_secure_link_module
```


来说明配置.这个模块在Nginx 0.7.18及以上版本中被支持.

__示例__

```
location /prefix/ {
    secure_link_secret   secret_word;
    if ($secure_link = "") {
        return 403;
    }
}
```

#### HttpImageFilter

**Version:** 0.7.54+

这个模块用来分发JPEG，GIF和PNG图片。这个没有默认开启，在编译nginx中通过./configure参数配置

```
--with-http_image_filter_module  
```

编译和运行这个模块必须安装libgd库。我们推荐使用最新版本的Libgd.

__示例__

```
location /img/ {
    proxy_pass     http://backend;
    image_filter   resize  150 100;
    error_page     415   = /empty;
}
location = /empty {
    empty_gif;
}
```

### mail模块

#### MailCore

Nginx 能够处理和代理以下邮件协议:

- IMAP
- POP3
- SMTP

#### MailAuth

__示例__

```
{
  auth_http           localhost:9000/cgi-bin/nginxauth.cgi;
  auth_http_timeout   5;
}
```

#### MailProxy

Ngin可以代理IMAP，POP3和SMTP协议

#### MailSSL

这个模块使得POP3／IMAP／SMTP可以使用SSL／TLS.配置已经定义了HTTP SSL模块，但是不支持客户端证书检测。

### 完整示例

#### 两个虚拟主机(纯静态-html 支持) 

```
http {
: server {
: listen          80;
: server_name     www.domain1.com;
: access_log      logs/domain1.access.log main;
: location / {
: index index.html;
: root  /var/www/domain1.com/htdocs;
: }
: }
: server {
: listen          80;
: server_name     www.domain2.com;
: access_log      logs/domain2.access.log main;
: location / {
: index index.html;
: root  /var/www/domain2.com/htdocs;
: }
: }
}
```

#### 虚拟主机标准配置(简化) 

```
http {
: server {
: listen          80 default;
: server_name     _ *;
: access_log      logs/default.access.log main;
: location / {
: index index.html;
: root  /var/www/default/htdocs;
: }
: }
}
```

#### 在父文件夹中建立子文件夹以指向子域名

这是一个添加子域名(或是当DNS已指向服务器时添加一个新域名)的简单方法。需要注意的是，我已经将FCGI配置进该文件了。如果你只想使服务器为静态文件服务，可以直接将FCGI配置信息注释掉，然后将默认主页文件变成index.html。

这个简单的方法比起为每一个域名建立一个 vhost.conf 配置文件来讲，只需要在现有的配置文件中增加如下内容：

```
server {
: # Replace this port with the right one for your requirements
: # 根据你的需求改变此端口
: listen       80;  #could also be 1.2.3.4:80 也可以是1.2.3.4:80的形式
: # Multiple hostnames seperated by spaces.  Replace these as well.
: # 多个主机名可以用空格隔开，当然这个信息也是需要按照你的需求而改变的。
: server_name  star.yourdomain.com *.yourdomain.com www.*.yourdomain.com;
: #Alternately: _ *
: #或者可以使用：_ * (具体内容参见本维基其他页面)
: root /PATH/TO/WEBROOT/$host;
: error_page  404              http://yourdomain.com/errors/404.html;
: access_log  logs/star.yourdomain.com.access.log;
: location / {
: root   /PATH/TO/WEBROOT/$host/;
: index  index.php;
: }
: # serve static files directly
: # 直接支持静态文件 (从配置上看来不是直接支持啊)
: location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|html)$ {
: access_log        off;
: expires           30d;
: }
: location ~ .php$ {
: # By all means use a different server for the fcgi processes if you need to
: # 如果需要，你可以为不同的FCGI进程设置不同的服务信息
: fastcgi_pass   127.0.0.1:YOURFCGIPORTHERE;
: fastcgi_index  index.php;
: fastcgi_param  SCRIPT_FILENAME  /PATH/TO/WEBROOT/$host/$fastcgi_script_name;
: fastcgi_param  QUERY_STRING     $query_string;
: fastcgi_param  REQUEST_METHOD   $request_method;
: fastcgi_param  CONTENT_TYPE     $content_type;
: fastcgi_param  CONTENT_LENGTH   $content_length;
: fastcgi_intercept_errors on;
: }
: location ~ /\.ht {
: deny  all;
: }
: }
```

#### [Nginx官方网站](http://www.sysoev.ru/nginx/docs/example.html) 的例子

```
#!nginx
: # 使用的用户和组
: user  www www;
: # 指定工作衍生进程数
: worker_processes  2;
: # 指定 pid 存放的路径
: pid /var/run/nginx.pid;
: # [ debug | info | notice | warn | error | crit ] 
: # 可以在下方直接使用 [ debug | info | notice | warn | error | crit ]  参数
: error_log  /var/log/nginx.error_log  info;
: events {
: # 允许的连接数
: connections   2000;
: # use [ kqueue | rtsig | epoll | /dev/poll | select | poll ] ;
: # 具体内容查看 http://wiki.codemongers.com/事件模型
: use kqueue;
: }
: http {
: include       conf/mime.types;
: default_type  application/octet-stream;
: log_format main      '$remote_addr - $remote_user [$time_local]  '
: '"$request" $status $bytes_sent '
: '"$http_referer" "$http_user_agent" '
: '"$gzip_ratio"';
: log_format download  '$remote_addr - $remote_user [$time_local]  '
: '"$request" $status $bytes_sent '
: '"$http_referer" "$http_user_agent" '
: '"$http_range" "$sent_http_content_range"';
: client_header_timeout  3m;
: client_body_timeout    3m;
: send_timeout           3m;
: client_header_buffer_size    1k;
: large_client_header_buffers  4 4k;
: gzip on;
: gzip_min_length  1100;
: gzip_buffers     4 8k;
: gzip_types       text/plain;
: output_buffers   1 32k;
: postpone_output  1460;
: sendfile         on;
: tcp_nopush       on;
: tcp_nodelay      on;
: send_lowat       12000;
: keepalive_timeout  75 20;
: #lingering_time     30;
: #lingering_timeout  10;
: #reset_timedout_connection  on;
: server {
: listen        one.example.com;
: server_name   one.example.com  www.one.example.com;
: access_log   /var/log/nginx.access_log  main;
: location / {
: proxy_pass         http://127.0.0.1/;
: proxy_redirect     off;
: proxy_set_header   Host             $host;
: proxy_set_header   X-Real-IP        $remote_addr;
: #proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
: client_max_body_size       10m;
: client_body_buffer_size    128k;
: client_body_temp_path      /var/nginx/client_body_temp;
: proxy_connect_timeout      90;
: proxy_send_timeout         90;
: proxy_read_timeout         90;
: proxy_send_lowat           12000;
: proxy_buffer_size          4k;
: proxy_buffers              4 32k;
: proxy_busy_buffers_size    64k;
: proxy_temp_file_write_size 64k;
: proxy_temp_path            /var/nginx/proxy_temp;
: charset  koi8-r;
: }
: error_page  404  /404.html;
: location /404.html {
: root  /spool/www;
: charset         on;
: source_charset  koi8-r;
: }
: location /old_stuff/ {
: rewrite   ^/old_stuff/(.*)$  /new_stuff/$1  permanent;
: }
: location /download/ {
: valid_referers  none  blocked  server_names  *.example.com;
: if ($invalid_referer) {
: #rewrite   ^/   http://www.example.com/;
: return   403;
: }
: #rewrite_log  on;
: # rewrite /download/*/mp3/*.any_ext to /download/*/mp3/*.mp3
: rewrite ^/(download/.*)/mp3/(.*)\..*$
: /$1/mp3/$2.mp3                   break;
: root         /spool/www;
: #autoindex    on;
: access_log   /var/log/nginx-download.access_log  download;
: }
: location ~* ^.+\.(jpg|jpeg|gif)$ {
: root         /spool/www;
: access_log   off;
: expires      30d;
: }
: }
: }
```

#### 负载均衡

一个简单的负载均衡的示例，把www.domain.com均衡到本机不同的端口，也可以改为均衡到不同的地址上。

```
http {
: upstream myproject {
: server 127.0.0.1:8000 weight=3;
: server 127.0.0.1:8001;
: server 127.0.0.1:8002;
: server 127.0.0.1:8003;
: }
: server {
: listen 80;
: server_name www.domain.com;
: location / {
: proxy_pass http://myproject;
: }
: }
}
```

## 集群模式

### 一致性Hash问题

[一致性Hash问题_CoderTnT的博客-CSDN博客](https://blog.csdn.net/codertnt/article/details/80005005)

一致性Hash算法也是使用取模的方法，一致性Hash算法是对2^32取模。简单来说，一致性Hash算法将整个哈希值空间组织成一个虚拟的圆环，如假设某哈希函数H的值空间为0-2^32-1（即哈希值是一个32位无符号整形）整个空间按顺时针方向组织，圆环的正上方的点代表0，0点右侧的第一个点代表1，以此类推，2、3、4、5、6……直到2^32-1，也就是说0点左侧的第一个点代表2^32-1， 0和2^32-1在零点中方向重合，我们把这个由2^32个点组成的圆环称为Hash环。

下一步将各个服务器使用Hash进行一个哈希，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置。 

接下来使用如下算法定位数据访问到相应服务器：将数据key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，第一台遇到的服务器就是其应该定位到的服务器。

一致性Hash算法对于节点的增减都只需重定位环空间中的一小部分数据，具有较好的容错性和可扩展性

一致性Hash算法在服务节点太少时，容易因为节点分部不均匀而造成数据倾斜（被缓存的对象大部分集中缓存在某一台服务器上）问题，此时必然造成大量数据集中到Node A上，而只有极少量会定位到Node B上。为了解决这种数据倾斜问题，一致性Hash算法引入了虚拟节点机制，即对每一个服务节点计算多个哈希，每个计算结果位置都放置一个此服务节点，称为虚拟节点。具体做法可以在服务器IP或主机名的后面增加编号来实现

同时数据定位算法不变，只是多了一步虚拟节点到实际节点的映射，例如定位到“Node A#1”、“Node A#2”、“Node A#3”三个虚拟节点的数据均定位到Node A上。这样就解决了服务节点少时数据倾斜的问题。在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。

### Session共享问题

[Session共享问题---理论 - 独孤靖云 - 博客园 (cnblogs.com)](https://www.cnblogs.com/lxhyty/p/11321709.html)

负载均衡的目的本来就是要为了平均分配请求，所以没有固定第一次访问和第二次访问的是同一台服务器，实际上无法确定的。第一标访问可能是a服务器，第二秒访问的可能是c服务器。这样的话，生成的session文件不可能恰巧都在同一台服务器上，所以，当同一个登录会员，访问第一台服务器生成了一个session数据。第二秒负载请求到第三台服务器，结果获取不到刚才生成的session数据。

### session复制

使用一些文件同步工具(linux下的rsync)，当a服务器中的session数据有更改的时候，就会把这些更改也同步到b，c服务器上去。通过复制的方式，最终a,b,c各个服务器上都拷贝了一份session数据。

坏处：

- ·速度慢。复制数据会出现延迟。比如第一秒访问是a服务器，修改了session数据，负载均衡，可能下一秒访问的是b服务器，session数据如果没有被复制到b服务器，则是读取不到session数据的，出现时间上的延迟。这种复制数据要消耗很多的网络带宽。在实际中业界用的比较少。机器的数量越多，复制数据的性能消耗越大。不具备高扩展性。
- ·复制session的方式，无论是网络带宽成本还是硬件开销上都很大。

### session存储客户端

把原来存储在服务器磁盘上的session数据存储到客户端的cookie中，一般是把session数据按照自己定义的加密规则，加密后保存在cookie中。
好处：
服务器的压力减小了，因为session数据不存在服务器磁盘上。根本就不会出现session读取不到的问题。
坏处：
1）网络请求占用很多。每次请求时，客户端都要通过cookie发送session数据给服务器
2）浏览器对cookie的大小存在限制。每个浏览器限制是不同的。

- ·Firefox和Safari允许cookie多达4097个字节
- ·Opera允许cookie多达4096个字节
- ·IE允许cookie多达4095个字节

3）一般session中存的都是重要性数据（账号、昵称、用户id等），会存在安全问题

### 访问规则

设计用一种算法（简单理解为规则）例如nginx的ip_hash机制，session分类保存在各台服务器下，那么读取的时候就按照这种规则去读取，就能定位到原来的服务器。其原理是存session和读session数据保证都在一台服务器操作，就不会需要涉及到共享，具体实现方式是通过约定一种分发机制来实现。也叫作sticky模式（粘性回话模式），同一个用户的访问请求都被派送到同一个服务器上。
坏处：
　　如果这台机子挂掉，那么后续的请求按照session的规则还是会分发到这台服务器上去，但是现在不可用了，比如用户编号是1-200涉及到的session数据保存到a服务器上去。所以只要一台出问题，1-200的用户就无法实现登录了。后面就不可用了

### session中间层

做一个中间层服务器，专门来存储所有访问涉及到的session。也就是所有的session都存储在这里。服务器端统一从这里读取session数据。

1.NFS做中间层
通过nfs的方式，各个php服务器操作session数据的时候，是读取本地磁盘目录，但实际上是一个共享网络文件。各个php服务器实际上操作的都是同一个目录的文件。

2.关系型数据库做中间层
把以前存储在文件中的session数据存储到数据库中去，那么这样做，其实就不用到php内置的session机制了（像session_start()之类的函数都不需要去用了）。
从数据库拿session数据，约定什么情况下数据过期了然后自动清理，这里是指删除数据库中的行。保存在文件中的时候，php有垃圾回收机制回去自动清理过期的session文件。
有些做法跟这种思想是类似的：比如ecshop、phpcms是把session数据都存储在数据库中去。服务端就是从数据库中拿session的数据。
坏处：

- ·放在数据库里面，访问量小没问题。大流量网站这么做，只会拖慢速度。因为查询数据库，造成数据库压力大。高并发访问的情况下，会出现很大的性能问题。
- ·在线人数决定了其瓶颈，主要问题是影响性能。在线人数，因为登录的session数据存储在数据库中，只要是登录的用户就会涉及到频繁的操作数据库。

3.非关系型数据库做中间层
将session数据保存在memcached，redis之类内存数据库中，因为内存的数据读取速度是很快的，与磁盘读取的速度不是一个数量级的，所以性能很高，用户并发量很大的时候尤其合适。而且方便统计在线人数，内存数据库系统能够控制内存中的过期数据自动失效（刚好符合session过期需要）

### LVS+Keepalived+Nginx实现高可用

[LVS+KeepAlived+Nginx高可用实现方案_码霸霸的博客-CSDN博客](https://blog.csdn.net/lupengfei1009/article/details/86514445)

#### LVS

- **LVS**是**Linux Virtual Server**的简写，意即**Linux虚拟服务器**，是一个虚拟的服务器集群系统。
- **宗旨**
  1. 使用集群技术和Linux操作系统实现一个高性能、高可用的服务器.
  2. 很好的可伸缩性（Scalability）
  3. 很好的可靠性（Reliability）
  4. 很好的可管理性（Manageability）。
- **特点**
  可伸缩网络服务的几种结构，它们都需要一个前端的负载调度器（或者多个进行主从备份）。我们先分析实现虚拟网络服务的主要技术，指出IP负载均衡技术是在负载调度器的实现技术中效率最高的。在已有的IP负载均衡技术中，主要有通过网络地址转换（Network Address Translation）将一组服务器构成一个高性能的、高可用的虚拟服务器，我们称之为VS/NAT技术（Virtual Server via Network Address Translation）。在分析VS/NAT的缺点和网络服务的非对称性的基础上，我们提出了通过IP隧道实现虚拟服务器的方法VS/TUN （Virtual Server via IP Tunneling），和通过直接路由实现虚拟服务器的方法VS/DR（Virtual Server via Direct Routing），它们可以极大地提高系统的伸缩性。VS/NAT、VS/TUN和VS/DR技术是LVS集群中实现的三种IP负载均衡技术。

#### KeepAlived

- keepalived是一个类似于layer3, 4 & 5交换机制的软件，也就是我们平时说的第3层、第4层和第5层交换。Keepalived是自动完成，不需人工干涉。

- Keepalived的作用是检测服务器的状态，如果有一台web服务器宕机，或工作出现故障，Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中，这些工作全部自动完成，不需要人工干涉，需要人工做的只是修复故障的服务器。

- Layer3,4,5工作在IP/TCP协议栈的IP层，TCP层，及应用层,原理分别如下：    
  - Layer3：Keepalived使用Layer3的方式工作式时，Keepalived会定期向服务器群中的服务器发送一个ICMP的数据包（既我们平时用的Ping程序）,如果发现某台服务的IP地址没有激活，Keepalived便报告这台服务器失效，并将它从服务器群中剔除，这种情况的典型例子是某台服务器被非法关机。Layer3的方式是以服务器的IP地址是否有效作为服务器工作正常与否的标准。
  - Layer4:如果您理解了Layer3的方式，Layer4就容易了。Layer4主要以TCP端口的状态来决定服务器工作正常与否。如web server的服务端口一般是80，如果Keepalived检测到80端口没有启动，则Keepalived将把这台服务器从服务器群中剔除。
  - Layer5：Layer5对指定的URL执行HTTP GET。然后使用MD5算法对HTTP GET结果进行求和。如果这个总数与预期值不符，那么测试是错误的，服务器将从服务器池中移除。该模块对同一服务实施多URL获取检查。如果您使用承载多个应用程序服务器的服务器，则此功能很有用。此功能使您能够检查应用程序服务器是否正常工作。MD5摘要是使用genhash实用程序（包含在keepalived软件包中）生成的。
    SSL_GET与HTTP_GET相同，但使用SSL连接到远程Web服务器。
    MISC_CHECK：此检查允许用户定义的脚本作为运行状况检查程序运行。结果必须是0或1.该脚本在导演盒上运行，这是测试内部应用程序的理想方式。可以使用完整路径（即/path_to_script/script.sh）调用可以不带参数运行的脚本。那些需要参数的需要用双引号括起来（即“/path_to_script/script.sh arg 1 … arg n”）
- **作用**
  主要用作RealServer的健康状态检查以及LoadBalance主机和BackUP主机之间failover的实现。
  高可用web架构: **LVS+keepalived+nginx+apache+php+eaccelerator**（+nfs可选 可不选）

### 时钟同步问题

[集群时钟同步问题_小城我家-CSDN博客](https://blog.csdn.net/ko0491/article/details/107483334)

时钟此处指服务器时间，如果集群中各个服务器时钟不⼀致势必导致⼀系列问题。

举⼀个例⼦，电商⽹站业务中，新增⼀条订单，那么势必会在订单表中增加了⼀条记录，该条记录中应
该会有“下单时间”这样的字段，往往我们会在程序中获取当前系统时间插⼊到数据库或者直接从数据库
服务器获取时间。那我们的订单⼦系统是集群化部署，或者我们的数据库也是分库分表的集群化部署，
然⽽他们的系统时钟缺不⼀致，⽐如有⼀台服务器的时间是昨天，那么这个时候下单时间就成了昨天，
那我们的数据将会混乱！如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200721104357813.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tvMDQ5MQ==,size_16,color_FFFFFF,t_70)

1.分布式集群中各个服务器节点都可以连接互联⽹

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020072110471286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tvMDQ5MQ==,size_16,color_FFFFFF,t_70)

定时同步时间

2.分布式集群中某⼀个服务器节点可以访问互联⽹或者所有节点都不能够访问互联⽹

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200721104822916.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tvMDQ5MQ==,size_16,color_FFFFFF,t_70)

⾸先设置好A的时间
把A配置为时间服务器（修改/etc/ntp.conf⽂件）

```shell
1、如果有 restrict default ignore，注释掉它
2、添加如下⼏⾏内容
restrict 172.17.0.0 mask 255.255.255.0 nomodify notrap # 放开局
域⽹同步功能,172.17.0.0是你的局域⽹⽹段
server 127.127.1.0 # local clock
fudge 127.127.1.0 stratum 10
3、重启⽣效并配置ntpd服务开机⾃启动
service ntpd restart
chkconfig ntpd on
123456789
```

集群中其他节点就可以从A服务器同步时间了

```shell
ntpdate 172.17.0.17
```

### 分布式调度问题

[分布式调度问题_weixin_44795847的博客-CSDN博客](https://blog.csdn.net/weixin_44795847/article/details/107662042)

调度—>定时任务，分布式调度—>在分布式集群环境下定时任务这件事
Elastic-job（当当⽹开源的分布式调度框架）

1. 运⾏在分布式集群环境下的调度任务（同⼀个定时任务程序部署多份，只应该有⼀个定时任务在执⾏）
2. 分布式调度—>定时任务的分布式—>定时任务的拆分（即为把⼀个⼤的作业任务拆分为多个⼩的作业任务，同时执⾏）

![img](https://img-blog.csdnimg.cn/20200729133705347.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDc5NTg0Nw==,size_16,color_FFFFFF,t_70)

#### 定时任务与消息队列的区别

- 共同点
  - 异步处理
    ⽐如注册、下单事件
  - 应⽤解耦
    不管定时任务作业还是MQ都可以作为两个应⽤之间的⻮轮实现应⽤解耦，这个⻮轮可以中转数据，当然单体服务不需要考虑这些，服务拆分的时候往往都会考虑
  - 流量削峰
    双⼗⼀的时候，任务作业和MQ都可以⽤来扛流量，后端系统根据服务能⼒定时处理订单或者从MQ抓取订单抓取到⼀个订单到来事件的话触发处理，对于前端⽤户来说看到的结果是已经下单成功了，下单是不受任何影响的
- 本质不同
  - 定时任务作业是时间驱动，⽽MQ是事件驱动；
  - 时间驱动是不可代替的，⽐如⾦融系统每⽇的利息结算，不是说利息来⼀条（利息到来事件）就算⼀下，⽽往往是通过定时任务批量计算；
  - 定时任务作业更倾向于批处理， MQ倾向于逐条处理；

#### 定时任务的实现方式

##### 任务调度框架Quartz使用

##### 分布式调度框架Elastic-Job

Elastic-Job是当当⽹开源的⼀个分布式调度解决⽅案，基于Quartz⼆次开发的，由两个相互独⽴的⼦项⽬Elastic-Job-Lite和Elastic-Job-Cloud组成。Elastic-Job-Lite，它定位为轻量级⽆中⼼化解决⽅案，使⽤Jar包的形式提供分布式任务的协调服务，⽽Elastic-Job-Cloud⼦项⽬需要结合Mesos以及Docker在云环境下使⽤。

**主要功能介绍：**

- 分布式调度协调
  在分布式环境中，任务能够按指定的调度策略执⾏，并且能够避免同⼀任务多实例重复执⾏
- 丰富的调度策略 基于成熟的定时任务作业框架Quartz cron表达式执⾏定时任务
- 弹性扩容缩容 当集群中增加某⼀个实例，它应当也能够被选举并执⾏任务；当集群减少⼀个实例时，它所执⾏的任务能被转移到别的实例来执⾏。
- 失效转移 某实例在任务执⾏失败后，会被转移到其他实例执⾏
- 错过执⾏作业重触发 若因某种原因导致作业错过执⾏，⾃动记录错过执⾏的作业，并在上次作业完成后⾃动触发。
- ⽀持并⾏调度 ⽀持任务分⽚，任务分⽚是指将⼀个任务分为多个⼩任务项在多个实例同时执⾏。
- 作业分⽚⼀致性 当任务被分⽚后，保证同⼀分⽚在分布式环境中仅⼀个执⾏实例。

Elastic-Job依赖于Zookeeper进⾏分布式协调，所以需要安装Zookeeper软件,Zookeeper的本质功能：存储+通知。

### 分布式ID

[9种 分布式ID生成方式_u011277123的博客-CSDN博客](https://blog.csdn.net/u011277123/article/details/108324479)

在我们业务数据量不大的时候，单库单表完全可以支撑现有业务，数据再大一点搞个MySQL主从同步读写分离也能对付。

但随着数据日渐增长，主从同步也扛不住了，就需要对数据库进行分库分表，但分库分表后需要有一个唯一ID来标识一条数据，数据库的自增ID显然不能满足需求；特别一点的如订单、优惠券也都需要有唯一ID做标识。此时一个能够生成全局唯一ID的系统是非常必要的。那么这个全局唯一ID就叫分布式ID。

- 全局唯一：必须保证ID是全局性唯一的，基本要求
- 高性能：高可用低延时，ID生成响应要快，否则反倒会成为业务瓶颈
- 高可用：100%的可用性是骗人的，但是也要无限接近于100%的可用性
- 好接入：要秉着拿来即用的设计原则，在系统设计和实现上要尽可能的简单
- 趋势递增：最好趋势递增，这个要求就得看具体业务场景了，一般不严格要求

#### UUID

UUID的生成简单到只有一行代码，输出结果 c2b8c2b9e46c47e3b30dca3b0d447718，但UUID却并不适用于实际的业务需求。像用作订单号UUID这样的字符串没有丝毫的意义，看不出和订单相关的有用信息；而对于数据库来说用作业务主键ID，它不仅是太长还是字符串，存储性能差查询也很耗时，所以不推荐用作分布式ID。

**优点：**

- 生成足够简单，本地生成无网络消耗，具有唯一性

**缺点：**

- 无序的字符串，不具备趋势自增特性
- 没有具体的业务含义
- 长度过长16 字节128位，36位长度的字符串，存储以及查询对MySQL的性能消耗较大，MySQL官方明确建议主键要尽量越短越好，作为数据库主键 UUID 的无序性会导致数据位置频繁变动，严重影响性能。

String uuid = UUID.randomUUID().toString().replaceAll("-","");

#### 数据库自增ID

基于数据库的auto_increment自增ID完全可以充当分布式ID，当我们需要一个ID的时候，向表中插入一条记录返回主键ID，但这种方式有一个比较致命的缺点，访问量激增时MySQL本身就是系统的瓶颈，用它来实现分布式服务风险比较大，不推荐！

**优点：**

- 实现简单，ID单调自增，数值类型查询速度快

**缺点：**

- DB单点存在宕机风险，无法扛住高并发场景

```
CREATE DATABASE `SEQ_ID`;
CREATE TABLE SEQID.SEQUENCE_ID (
    id bigint(20) unsigned NOT NULL auto_increment, 
    value char(10) NOT NULL default '',
    PRIMARY KEY (id),
) ENGINE=MyISAM;
insert into SEQUENCE_ID(value)  VALUES ('values');
```

#### 数据库集群模式

那对上边的方式做一些高可用优化，换成主从模式集群。害怕一个主节点挂掉没法用，那就做双主模式集群，也就是两个Mysql实例都能单独的生产自增ID。

**优点：**

- 解决DB单点问题

**缺点：**

- 不利于后续扩容，而且实际上单个数据库自身压力还是大，依旧无法满足高并发场景。

#### 数据库的号段模式

号段模式是当下分布式ID生成器的主流实现方式之一，号段模式可以理解为从数据库批量的获取自增ID，每次从数据库取出一个号段范围，例如 (1,1000] 代表1000个ID，具体的业务服务将本号段，生成1~1000的自增ID并加载到内存。

```
CREATE TABLE id_generator (
  id int(10) NOT NULL,
  max_id bigint(20) NOT NULL COMMENT '当前最大id',
  step int(20) NOT NULL COMMENT '号段的布长',
  biz_type    int(20) NOT NULL COMMENT '业务类型',
  version int(20) NOT NULL COMMENT '版本号',
  PRIMARY KEY (`id`)
) 
```

等这批号段ID用完，再次向数据库申请新号段，对max_id字段做一次update操作，update max_id= max_id + step，update成功则说明新号段获取成功，新的号段范围是(max_id ,max_id +step]。

```
update id_generator set max_id = #{max_id+step}, version = version + 1 where version = # {version} and biz_type = XXX
```

由于多业务端可能同时操作，所以采用版本号version乐观锁方式更新，这种分布式ID生成方式不强依赖于数据库，不会频繁的访问数据库，对数据库的压力小很多。

#### Redis模式

Redis也同样可以实现，原理就是利用redis的 incr命令实现ID的原子性自增。

```
127.0.0.1:6379> set seq_id 1     // 初始化自增ID为1
OK
127.0.0.1:6379> incr seq_id      // 增加1，并返回递增后的数值
(integer) 2
```

用redis实现需要注意一点，要考虑到redis持久化的问题。redis有两种持久化方式RDB和AOF

- RDB会定时打一个快照进行持久化，假如连续自增但redis没及时持久化，而这会Redis挂掉了，重启Redis后会出现ID重复的情况。
- AOF会对每条写命令进行持久化，即使Redis挂掉了也不会出现ID重复的情况，但由于incr命令的特殊性，会导致Redis重启恢复的数据时间过长。

#### 雪花算法（Snowflake）模式

雪花算法（Snowflake）是twitter公司内部分布式项目采用的ID生成算法，开源后广受国内大厂的好评，在该算法影响下各大公司相继开发出各具特色的分布式生成器。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wNi10dC5ieXRlaW1nLmNvbS9vcmlnaW4vcGdjLWltYWdlLzE4NDdmMDViNjRkMDRkYzM4YjRmNDY1ZGRlOGRmNGQ0?x-oss-process=image/format,png)

Snowflake ID组成结构：正数位（占1比特）+ 时间戳（占41比特）+ 机器ID（占5比特）+ 数据中心（占5比特）+ 自增值（占12比特），总共64比特组成的一个Long类型。

- 第一个bit位（1bit）：Java中long的最高位是符号位代表正负，正数是0，负数是1，一般生成ID都为正数，所以默认为0。
- 时间戳部分（41bit）：毫秒级的时间，不建议存当前时间戳，而是用（当前时间戳 - 固定开始时间戳）的差值，可以使产生的ID从更小的值开始；41位的时间戳可以使用69年，(1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69年
- 工作机器id（10bit）：也被叫做workId，这个可以灵活配置，机房或者机器号组合都可以。
- 序列号部分（12bit），自增值支持同一毫秒内同一个节点可以生成4096个ID

根据这个算法的逻辑，只需要将这个算法用Java语言实现出来，封装为一个工具方法，那么各个业务应用可以直接使用该工具方法来获取分布式ID，只需保证每个业务应用有自己的工作机器id即可，而不需要单独去搭建一个获取分布式ID的应用。

```
package com.igetcool.teach.mgt.service.impl;
/**
 * *    Twitter的SnowFlake算法,使用SnowFlake算法生成一个整数，然后转化为62进制变成一个短地址URL    *    *    https://github.com/beyondfengyu/SnowFlake
 */
public class SnowFlakeShortUrl {
    /**
     * 起始的时间戳
     */
    private final static long START_TIMESTAMP = 1480166465631L;
    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12;     //序列号占用的位数
    private final static long MACHINE_BIT = 5;       //机器标识占用的位数
    private final static long DATA_CENTER_BIT = 5;   //数据中心占用的位数
    /**
     * 每一部分的最大值
     */
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_DATA_CENTER_NUM = -1L ^ (-1L << DATA_CENTER_BIT);
    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATA_CENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTAMP_LEFT = DATA_CENTER_LEFT + DATA_CENTER_BIT;
    private long dataCenterId;          //数据中心
    private long machineId;             //机器标识
    private long sequence = 0L;         //序列号
    private long lastTimeStamp = -1L;   //上一次时间戳
    private long getNextMill() {
        long mill = getNewTimeStamp();
        while (mill <= lastTimeStamp) {
            mill = getNewTimeStamp();
        }
        return mill;
    }
    private long getNewTimeStamp() {
        return System.currentTimeMillis();
    }
    /**
     * 根据指定的数据中心ID和机器标志ID生成指定的序列号      *      *    @param    dataCenterId    数据中心ID      *    @param    machineId  机器标志ID
     */
    public SnowFlakeShortUrl(long dataCenterId, long machineId) {
        if (dataCenterId > MAX_DATA_CENTER_NUM || dataCenterId < 0) {
            throw new IllegalArgumentException("DtaCenterId can't be greater than MAX_DATA_CENTER_NUM or less than 0！");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("MachineId can't be greater than MAX_MACHINE_NUM or less than 0！");
        }
        this.dataCenterId = dataCenterId;
        this.machineId = machineId;
    }
    /**
     * 产生下一个ID      *      *    @return
     */
    public synchronized long nextId() {
        long currTimeStamp = getNewTimeStamp();
        if (currTimeStamp < lastTimeStamp) {
            throw new RuntimeException("Clock    moved    backwards.        Refusing    to    generate    id");
        }
        if (currTimeStamp == lastTimeStamp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currTimeStamp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }
        lastTimeStamp = currTimeStamp;
        return (currTimeStamp - START_TIMESTAMP) << TIMESTAMP_LEFT;
        //时间戳部分     |   dataCenterId    <<    DATA_CENTER_LEFT
        // 数据中心部分  |   machineId    <<    MACHINE_LEFT
        // 机器标识部分  |   sequence;
        // 序列号部分
    }
    public static void main(String[] args) {
        SnowFlakeShortUrl snowFlake = new SnowFlakeShortUrl(2, 3);
        for (int i = 0; i < (1 << 4); i++) {
            //10进制
            System.out.println(snowFlake.nextId());
        }
    }
}
```

#### 百度（uid-generator）

uid-generator是由百度技术部开发，项目GitHub地址 https://github.com/baidu/uid-generator

uid-generator是基于Snowflake算法实现的，与原始的snowflake算法不同在于，uid-generator支持自定义时间戳、工作机器ID和 序列号 等各部分的位数，而且uid-generator中采用用户自定义workId的生成策略。

uid-generator需要与数据库配合使用，需要新增一个WORKER_NODE表。当应用启动时会向数据库表中去插入一条数据，插入成功后返回的自增ID就是该机器的workId数据由host，port组成。

**对于uid-generator ID组成结构**：

workId，占用了22个bit位，时间占用了28个bit位，序列化占用了13个bit位，需要注意的是，和原始的snowflake不太一样，时间的单位是秒，而不是毫秒，workId也不一样，而且同一应用每次重启就会消费一个workId。

#### 美团（Leaf）

Leaf由美团开发，github地址：https://github.com/Meituan-Dianping/Leaf

Leaf同时支持号段模式和snowflake算法模式，可以切换使用。

##### 号段模式

先导入源码 https://github.com/Meituan-Dianping/Leaf ，在建一张表leaf_alloc

```
DROP TABLE IF EXISTS `leaf_alloc`;
CREATE TABLE `leaf_alloc` (
  `biz_tag` varchar(128)  NOT NULL DEFAULT '' COMMENT '业务key',
  `max_id` bigint(20) NOT NULL DEFAULT '1' COMMENT '当前已经分配了的最大id',
  `step` int(11) NOT NULL COMMENT '初始步长，也是动态调整的最小步长',
  `description` varchar(256)  DEFAULT NULL COMMENT '业务key的描述',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '数据库维护的更新时间',
  PRIMARY KEY (`biz_tag`)
) ENGINE=InnoDB;
```

然后在项目中开启号段模式，配置对应的数据库信息，并关闭snowflake模式

```
leaf.name=com.sankuai.leaf.opensource.test
leaf.segment.enable=true
leaf.jdbc.url=jdbc:mysql://localhost:3306/leaf_test?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8
leaf.jdbc.username=rootleaf.jdbc.password=rootleaf.snowflake.enable=false
#leaf.snowflake.zk.address=
#leaf.snowflake.port=
```

启动leaf-server 模块的 LeafServerApplication项目就跑起来了

号段模式获取分布式自增ID的测试url ：http：//localhost：8080/api/segment/get/leaf-segment-test

监控号段模式：http://localhost:8080/cache

##### snowflake模式

Leaf的snowflake模式依赖于ZooKeeper，不同于原始snowflake算法也主要是在workId的生成上，Leaf中workId是基于ZooKeeper的顺序Id来生成的，每个应用在使用Leaf-snowflake时，启动时都会都在Zookeeper中生成一个顺序Id，相当于一台机器对应一个顺序节点，也就是一个workId。

```
leaf.snowflake.enable=true
leaf.snowflake.zk.address=127.0.0.1
leaf.snowflake.port=2181
```

snowflake模式获取分布式自增ID的测试url：http://localhost:8080/api/snowflake/get/test

#### 滴滴（Tinyid）

Tinyid由滴滴开发，Github地址：https://github.com/didi/tinyid。

Tinyid是基于号段模式原理实现的与Leaf如出一辙，每个服务获取一个号段（1000,2000]、（2000,3000]、（3000,4000]

Tinyid提供http和tinyid-client两种方式接入。

## Web服务综合解决方案

### 动静分离思想及架构设计

动静分离是指，静态页面与动态页面解耦分离，用不同系统承载对应流量的架构设计方法。

### 页面动态模块化渲染（Nginx+lua）

Lua 是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。

1、分发层nginx，lua应用，会将商品id，商品店铺id，都转发到后端的应用nginx
2、应用nginx的lua脚本接收到请求
3、获取请求参数中的商品id，以及商品店铺id
4、根据商品id和商品店铺id，在nginx本地缓存中尝试获取数据
5、如果在nginx本地缓存中没有获取到数据，那么就到redis分布式缓存中获取数据，如果获取到了数据，还要设置到nginx本地缓存中
※不推荐使用nginx+lua直接去获取redis数据，因为openresty没有太好的redis cluster的支持包，所以建议是发送http请求到缓存数据生产服务，由该服务提供一个http接口，缓存数据生产服务可以基于redis cluster api从redis中直接获取数据，并返回给nginx

### 内容分布网络CDN加速实现原理

[网络CDN加速及其原理_jwq101666的博客-CSDN博客](https://blog.csdn.net/jwq101666/article/details/78575370)

[CDN是什么与CDN加速的原理 - 马一特 - 博客园 (cnblogs.com)](https://www.cnblogs.com/mayite/p/9210108.html)

 CDN的全称是Content Delivery Network，即内容分发网络。其目的是通过在现有的Internet中增加一层新的网络架构(CACHE)，将网站的内容发布到最接近用户的网络“边缘”，使用户可以就近取得所需的内容，提高用户访问网站的响应速度。CDN有别于镜像，因为它比镜像更智能，或者可以做这样一个比喻：CDN=更智能的镜像+缓存+流量导流。因而，CDN可以明显提高Internet网络中信息流动的效率。从技术上全面解决由于网络带宽小、用户访问量大、网点分布不均等问题，提高用户访问网站的响应速度。

当用户访问已经加入CDN服务的网站时，首先通过DNS重定向技术确定最接近用户的最佳CDN节点，同时将用户的请求指向该节点。当用户的请求到达指定节点时，CDN的服务器（节点上的高速缓存）负责将用户请求的内容提供给用户。具体流程为: 用户在自己的浏览器中输入要访问的网站的域名，浏览器向本地DNS请求对该域名的解析，本地DNS将请求发到网站的主DNS，主DNS根据一系列的策略确定当时最适当的CDN节点，并将解析的结果（IP地址）发给用户，用户向给定的CDN节点请求相应网站的内容。

#### CDN设计思路

- 避让：尽可能避开互联网上有可能影响数据传输速度和稳定性的瓶颈和环节，使内容传输的更快、更稳定。
- 检测：通过在网络各处放置节点服务器所构成的在现有的互联网基础之上的一层智能虚拟网络，CDN系统能够实时监测网络流量和各节点的连接、负载状况以及到用户的距离和响应时间等综合信息将用户的请求
- 分发：根据监测情况重新导向离用户最近的服务节点上

#### 基础架构

最简单的CDN网络由一个DNS服务器和几台缓存服务器组成：

1. 当用户点击网站页面上的内容URL，经过本地DNS系统解析，DNS系统会最终将域名的解析权交给CNAME指向的CDN专用DNS服务器。
2. CDN的DNS服务器将CDN的全局负载均衡设备IP地址返回给用户
3. 用户向CDN的全局负载均衡设备发起内容URL访问请求
4. CDN全局负载均衡设备根据用户IP地址，以及用户请求的内容URL，选择一台用户所属区域的区域负载均衡设备，告诉用户向这台设备发起请求
5. 区域负载均衡设备会为用户选择一台合适的缓存服务器提供服务，选择的依据包括：根据用户IP地址，判断哪一台服务器距用户最近；根据用户所请求的URL中携带的内容名称，判断哪一台服务器上有用户所需内容；查询各个服务器当前的负载情况，判断哪一台服务器尚有服务能力。基于以上这些条件的综合分析之后，区域负载均衡设备会向全局负载均衡设备返回一台缓存服务器的IP地址
6. 全局负载均衡设备把服务器的IP地址返回给用户
7. 用户向缓存服务器发起请求，缓存服务器响应用户请求，将用户所需内容传送到用户终端。如果这台缓存服务器上并没有用户想要的内容，而区域均衡设备依然将它分配给了用户，那么这台服务器就要向它的上一级缓存服务器请求内容，直至追溯到网站的源服务器将内容拉到本地

![img](https://pic4.zhimg.com/80/v2-5793aec83fc645e002a1cd70ab7209a3_hd.jpg)

#### 服务模式

简单地说，CDN是一个经策略性部署的整体系统，包括分布式存储、负载均衡、网络请求的重定向和内容管理4个要件，而内容管理和全局的网络流量管理(Traffic Management)是CDN的核心所在。通过用户就近性和服务器负载的判断，CDN确保内容以一种极为高效的方式为用户的请求提供服务。

#### 应用

　　国内访问量较高的网站、直播、视频平台，均使用CDN网络加速技术，虽然网站的访问巨大，但无论在什么地方访问都会感觉速度很快。而一般的网站如果服务器在网通，电信用户访问很慢，如果服务器在电信，网通用户访问又很慢。

　　通过在现有的Internet中增加一层新的网络架构，将网站的内容发布到最接近用户的cache服务器内，通过DNS负载均衡的技术，判断用户来源就近访问cache服务器取得所需的内容，解决Internet网络拥塞状况，提高用户访问网站的响应速度，如同提供了多个分布在各地的加速器，以达到快速、可冗余的为多个网站加速的目的。

　　CDN服务最初用于确保快速可靠地分发静态内容，这些内容可以缓存，最适合在网速庞大的网络中存储和分发，该网络在几十多个国家的十几个网络中的覆盖CDN网络服务器。由于动态内容必须通过互联网来传输，因此要提供快速的网络体验。如今的CDN可谓是大文件、小文件、点播、直播、动静皆宜！

#### 主要特点

- 本地Cache加速，提高了企业站点（尤其含有大量图片和静态页面站点）的访问速度，并大大提高以上性质站点的稳定性
- 镜像服务消除了不同运营商之间互联的瓶颈造成的影响，实现了跨运营商的网络加速，保证不同网络中的用户都能得到良好的访问质量。
- 远程加速 远程访问用户根据DNS负载均衡技术 智能自动选择Cache服务器，选择最快的Cache服务器，加快远程访问的速度
- 带宽优化 自动生成服务器的远程Mirror（镜像）cache服务器，远程用户访问时从cache服务器上读取数据，减少远程访问的带宽、分担网络流量、减轻原站点WEB服务器负载等功能。
- 集群抗攻击 广泛分布的CDN节点加上节点之间的智能冗余机制，可以有效地预防黑客入侵以及降低各种D.D.o.S攻击对网站的影响，同时保证较好的服务质量 。

#### 关键技术

内容发布：它借助于建立索引、缓存、流分裂、组播（Multicast）等技术

内容路由：它是整体性的网络负载均衡技术，通过内容路由器中的重定向（DNS）机制，在多个远程POP上均衡用户的请求，以使用户请求得到最近内容源的响应；

内容交换：它根据内容的可用性、服务器的可用性以及用户的背景，在POP的缓存服务器上，利用应用层交换、流分裂、重定向（ICP、WCCP）等技术，智能地平衡负载流量；

性能管理：它通过内部和外部监控系统，获取网络部件的状况信息，测量内容发布的端到端性能（如包丢失、延时、平均带宽、启动时间、帧速率等），保证网络处于最佳的运行状态。

![img](https://pic1.zhimg.com/80/v2-eaf80abf6a52913375d2ade0dde79ed0_hd.jpg)

#### 适用范围

**一般来说以资讯、内容等为主的网站，具有一定访问体量的网站**

例如资讯网站、政府机构网站、行业平台网站、商城等以动态内容为主的网站

例如论坛、博客、交友、SNS、网络游戏、搜索/查询、金融等。提供http下载的网站

例如软件开发商、内容服务提供商、网络游戏运行商、源码下载等有大量流媒体点播应用的网站

例如：拥有视频点播平台的电信运营商、内容服务提供商、体育频道、宽频频道、在线教育、视频博客等

### SEO搜索引擎优化

SEO（Search Engine Optimization）：汉译为搜索引擎优化。是一种方式：利用[搜索引擎](https://baike.baidu.com/item/搜索引擎/104812)的规则提高网站在有关搜索引擎内的[自然排名](https://baike.baidu.com/item/自然排名/2092669)。目的是让其在行业内占据领先地位，获得[品牌](https://baike.baidu.com/item/品牌/235720)收益。很大程度上是网站经营者的一种商业行为，将自己或自己公司的排名前移。

搜索引擎优化的技术手段主要有[黑帽](https://baike.baidu.com/item/黑帽/8850364)（black hat）、[白帽](https://baike.baidu.com/item/白帽/7149278)（white hat）两大类。通过作弊手法欺骗搜索引擎和访问者，最终将遭到搜索引擎惩罚的手段被称为黑帽，比如隐藏关键字、制造大量的meta字、alt标签等。而通过正规技术和方式，且被搜索引擎所接受的SEO技术，称为白帽。

1．[白帽](https://baike.baidu.com/item/白帽/7149278)方法

搜索引擎优化的白帽法遵循搜索引擎的接受原则。他们的建议一般是为用户创造内容、让这些内容易于被搜索引擎机器人索引、并且不会对搜寻引擎系统耍花招。一些网站的员工在设计或构建他们的网站时出现失误以致该网站排名靠后时，白帽法可以发现并纠正错误，譬如机器无法读取的选单、无效链接、临时改变导向、效率低下的索引结构等。

2．[黑帽](https://baike.baidu.com/item/黑帽/8850364)方法

黑帽方法通过欺骗技术和滥用搜索算法来推销毫不相关、主要以商业为着眼的网页。黑帽SEO的主要目的是让网站得到他们所希望的排名进而获得更多的曝光率，这可能导致令普通用户不满的搜索结果。因此搜索引擎一旦发现使用“黑帽”技术的网站，轻则降低其排名，重则从搜索结果中永远剔除该网站。选择黑帽SEO服务的商家，一部分是因为不懂技术，在没有明白SEO价值所在的情况下被服务商欺骗；另一部分则只注重短期利益，存在赚一笔就走人的心态。