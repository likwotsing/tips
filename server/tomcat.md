# tomcat性能优化

tomcat默认参数是为开发环境制定，而不适合生产环境，尤其是内存和线程的配置，默认都很低，容易成为性能瓶颈。

## tomcat内存优化

linux

```js
// tomcat/bin/catalina.sh, 在前面加入
JAVA_OPTS="-XX:PermSize=64M -XX:MaxPermSize=128m -Xms512m -Xmx1024m -Duser.timezone=Asia/Shanghai"
```

windows

```js
// tomcat/bin/catalina.bat, 在前面加入
set JAVA_OPTS=-XX:PermSize=64M -XX:MaxPermSize=128m -Xms512m -Xmx1024m
```

最大堆内存是1024M，可以按照机器具体硬件配置优化

## tomcat线程优化

```js
<Connector port="80" protocol="HTTP/1.1" maxThreads="600" minSpareThreads="100" maxSpareThreads="500" acceptCount="700"
connectionTimeout="20000" redirectPort="8443" />
```

maxThreads="600" // 最大线程数

minSpareThreads="100" // 初始化创建的线程数

maxSpareThreads="500" // 一旦创建的线程超过该值，Tomcat就会关闭不再需要的socket线程

acceptCount="700" // 指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过该值的请求将不予处理



```js
<Connector port="8009" protocol="AJP/1.3" maxThreads="600" minSpareThreads="100" maxSpareThreads="500" acceptCount="700"
connectionTimeout="20000" redirectPort="8443" />
```

如果使用apache和tomcat做集群的负载均衡，并且使用ajp协议做apache和tomcat的协议转发，那么还需要优化ajp connector。



由于tomcat有多个connector，所以tomcat线程的配置，支持多个connector共享一个线程池。

```js
//conf/server.xml, 增加以下内容
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-" maxThreads="500" minSpareThreads="20" maxIdleTime="60000" />
```

最大线程500（一般服务器足够），最小空闲线程数20，线程最大空闲时间60秒。

然后，修改<Connector...>节点，增加executor属性，设置为线程池的名字：

```js
<Connector executor="tomcatThreadPool" port="80" protocol="HTTP/1.1"  connectionTimeout="60000" keepAliveTimeout="15000" maxKeepAliveRequests="1"  redirectPort="443" />
```

可以多个connector共用一个线程池，所以ajp connector也同样可以设置使用tomcatThreadPool线程池。

## 禁用DNS查询

当web应用程序想要记录客户端的信息时，它也会记录客户端的IP地址或者通过域名服务器查找机器名转换为IP地址。

DNS查询需要占用网络，并且包括可能从很多很远的服务器或者不起作用的服务器上去获取对应的IP的过程，这样会消耗一定的时间。

修改`server.xml`文件中的Connector节点，修改enableLookups参数值：`enableLookups="false"`，如果为true，则可以通过调用request.getRemoteHost()进行DNS查询来得到远程客户端的实际主机名，若为false则不进行DNS查询，而是返回其IP地址。

## 设置session过期时间

```js
// conf/web.xml，单位为分钟
<session-config>   
    <session-timeout>180</session-timeout>     
</session-config> 
```

## APR插件提高Tomcat性能

APR(Apache Portable Runtime)是一个高可移植库，它是Apache HTTP Server 2.x的核心。

APR有很多用于，包括访问高级IO功能（例如sendfile，epoll和OpenSSL），OS级别功能（随机数生成，系统状态等），本地进程管理（共享内存，NT管道和UNIX sockets）。这些功能可以使Tomcat作为一个通常的前台WEB服务器，能更好地核其它本地web技术集成，总体上让Java更有效率作为一个高性能web服务器平台而不是简单作为后台容器。

如果不用APR，一个线程同一时间只能处理一个用户，势必会造成阻塞，所以生产环境下用APR是非常必要的。

```js
(1)安装APR tomcat-native
    apr-1.3.8.tar.gz   安装在/usr/local/apr
    #tar zxvf apr-1.3.8.tar.gz
    #cd apr-1.3.8
    #./configure;make;make install
    
    apr-util-1.3.9.tar.gz  安装在/usr/local/apr/lib
    #tar zxvf apr-util-1.3.9.tar.gz
    #cd apr-util-1.3.9  
    #./configure --with-apr=/usr/local/apr ----with-java-home=JDK;make;make install
    
    #cd apache-tomcat-6.0.20/bin  
    #tar zxvf tomcat-native.tar.gz  
    #cd tomcat-native/jni/native  
    #./configure --with-apr=/usr/local/apr;make;make install
    
  (2)设置 Tomcat 整合 APR
    修改 tomcat 的启动 shell （startup.sh），在该文件中加入启动参数：
      CATALINA_OPTS="$CATALINA_OPTS -Djava.library.path=/usr/local/apr/lib" 。
 
  (3)判断安装成功:
    如果看到下面的启动日志，表示成功。
      2007-4-26 15:34:32 org.apache.coyote.http11.Http11AprProtocol init
```

