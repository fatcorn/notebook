# tomcat笔记

## 参考文献

```http
http://objcoding.com/2019/05/30/tomcat-architecture/ 聊聊Tomcat的架构设计
https://juejin.im/post/5af176196fb9a07ac90d2ac8  tomcat 源码分析


```



## 总体架构概览

- 划分成 container（容器） 与 connector （链接器）

  ![](https://raw.githubusercontent.com/objcoding/objcoding.github.io/master/images/tomcat_3.png)

    																	w-工作流程图						
  - 在上图中 一个tomcat server 可以包含 多个service， 而 connector 与 container 又包含在service中，service服务中可存在 多个连接器 已支持多种网络协议（如http1，HTTP2，ajp.
  - container 容器外还有一层engine 引擎 包裹负责与处理连接器的请求与响应，连接器与容器之间通过 ServletRequest 和 ServletResponse 对象进行交流。
  - 下一步看protocolHandler 适配器模式
  
- 入口点：
  - 启动类：Catalina.startup.bootstrap
  - 启动过程：
  
- connector

  	- 调用地点：Catalina.createStartDigester？
  	- 浏览至 Acceptor org.apache.tomcat.util.net.NioEndpoint;