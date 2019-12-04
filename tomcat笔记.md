# tomcat笔记

## 参考文献

```http
http://objcoding.com/2019/05/30/tomcat-architecture/ 聊聊Tomcat的架构设计
https://juejin.im/post/5af176196fb9a07ac90d2ac8  tomcat 源码分析


```



## 关联知识

~~~http
1、队列式同步器 AbstractQueuedSynchronizer https://juejin.im/post/5c3ac10351882524bb0b337f

~~~



## 总体架构概览

- 划分成 container（容器） 与 connector （链接器）

  ![](./资源\tomcat 流程图.png)

    																	w-工作流程图						
  - 在上图中 一个tomcat server 可以包含 多个service， 而 connector 与 container 又包含在service中，service服务中可存在 多个连接器 已支持多种网络协议（如http1，HTTP2，ajp.
  - container 容器外还有一层engine 引擎 包裹负责与处理连接器的请求与响应，连接器与容器之间通过 ServletRequest 和 ServletResponse 对象进行交流。
  - 下一步看protocolHandler 适配器模式
  
- 入口点：
  - 启动类：Catalina.startup.Bootstrap
  - 启动过程：
  
- connector

  	- 调用地点：Catalina.createStartDigester？
   
   - 三大线程：Acceptor、Pollor、Worker

  	- Acceptor 
        	  - 主要工作：负责接受TCP请求（socket）
                  
                	  - 位置：org.apache.tomcat.util.net.Acceptor
                
          	 	 	  - 启动地点：org.apache.tomcat.util.net.NioEndpoint.startInternal()
          
                ​    
          
                	  - 待完成项：
          
            	  	Acceptor 第83行  endpoint.countUpOrAwaitConnection(); 原理分析
              
          - 关键代码：
      ```java
          Acceptor: 
              // socket 是一个SocketChannel对象，对应一个tcp连接
              95 			socket = endpoint.serverSocketAccept();
                  ...
              // 配置socket,将SocketChannel, 转换成NioChannel 然后将socket请求交给Pollor
              115         if (!endpoint.setSocketOptions(socket)) {
                  
        
                
              
      ```
      
   -  Poller
  
     - 主要工作：不断轮询其selector，检查准备就绪的socket(有数据可读或可写)，实现io的多路复用
     
     - 位置：org.apache.tomcat.util.net.NioEndpoint（NioEndpoint 的内部类）
     
     - 启动地点：
     
          ```
          org.apache.tomcat.util.net.NioEndpoint.startInternal()
          
           276				poller = new Poller();
           277           	Thread pollerThread = new Thread(poller, getName() + "-		                     	  ClientPoller);
          ```
       
        - 关键代码

  ~~~JAVA
    NioEndpoint:
          ///	将将SocketChannel转换为channel(NioChannel),socketWrapper 是装饰模式的					NioChannel，目的是功能增强
          // register 方法 向轮询器(poller)注册一个新创建的套接字,并为这个套接字，新建一个					PollerEvent事件
          425         poller.register(channel, socketWrapper);
  
          	................
              
          ///register 方法，从事件站栈eventCache取出，或新建事件
          651         if (v != null) {
                          r = eventCache.pop();
                      }
                      if (r == null) {
                          r = new PollerEvent(socket, OP_REGISTER);
                      } else {
                          r.reset(socket, OP_REGISTER);
                      }
          // 在这里将poller 加入PollerEvent 队列
          659         addEvent(r);
  
       		................
              
              
         /// 在Poller run 方法内，调用 events() 方法,从PollerEvent 队列，拿出一个   				    PollerEvent， 为PollerEvent队中取出的对象 包含的 NioChannel 实例注册    			    SelectionKey.OP_READ 事件
          700   	    hasEvents = events();     // 调用 events() 方法
  
              ................
                  
          625         PollerEvent pe = null;
                      for (int i = 0, size = events.size(); i < size && (pe =                                 events.poll()) !=null; i++ ) { //events.poll()取出PollerEvent实列
                          result = true;
                          try {
          629                   pe.run();       //调用PollerEvent实列方法注册事件
                              
            ................
                  
          //     注册SelectionKey.OP_READ事件                
          504          socket.getIOChannel().
                  		register(socket.getSocketWrapper().
                          	getPoller().getSelector(),
                           	SelectionKey.OP_READ,socket.getSocketWrapper());
             ................
                  
         	// 		再有读写事件到来时，调用processKey，将soket交给woker,sk是一个SelectionKey对象
          743          processKey(sk, socketWrapper);
                              
  ~~~

   - Woker:

          - 主要工作：处理poller传递过来的sk（SelectionKey），socketWrapper（socket）对象，拆取socketWrapper，获取inputbuffer，从字节缓冲区中读取http请求头，及body

          - 位置：

                 - org.apache.tomcat.util.net.NioEndpoint（NioEndpoint 的内部类）（通过Excutor调度执行）
                 - org.apache.tomcat.util.net.AbstractEndpoint（抽象节点类）具体处理sk、socketWrapper，再将socketWrapper传递给SocketProcessorBase，SocketProcessorBase选择具体实现
                 - org.apache.tomcat.util.net.SocketProcessorBase(被Excutor调度的runable抽象类)

       - 关键代码：

            ```java
             NioEndpoint:
            			// Create worker collection
                        if (getExecutor() == null) {
            270              createExecutor();
                        }
            
            	................
                    
             AbstractEndpoint:
            		///这里创建了一个线程池，用于执行worker的解析http报文的任务
            903	 	public void createExecutor() {
                        internalExecutor = true;
                		/*  TaskQueue 实现 
                		 *  LinkedBlockingQueue接口（最大值队列长度Integer.Max,属于无界队列）
                		 */
                        TaskQueue taskqueue = new TaskQueue();
                        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-",
                                         daemon, getThreadPriority());
                        executor = new ThreadPoolExecutor(getMinSpareThreads(), 
                                         getMaxThreads(),60,TimeUnit.SECONDS,taskqueue, tf);
                        taskqueue.setParent( (ThreadPoolExecutor) executor);
                	}
            
            	................
                    
               NioEndpoint:
            		 // 处理socket读写事件
                     protected void processKey(SelectionKey sk, 
                                              NioSocketWrapper socketWrapper) {
                         ...
                             // 这里判读断读事件是否到来，然后处理socket
                             if (sk.isReadable()) {
                                 if (socketWrapper.readOperation != null) {
                                     // 使用线程池读取socket
                                     if (!socketWrapper.readOperation.process()) {
                                         closeSocket = true;
                                     }
                                     // 上一步线程池没有处理socket，再次尝试，如果线程池还是失联，直接运                             行SocketProcessorBase接口的run方法
                                 } else if (!processSocket(socketWrapper, SocketEvent.OPEN_READ, true)) {
                                     closeSocket = true;
                                 }
                             } 
                         ...
                             
                ................
                             
            AbstractEndpoint：                 
                      //将socket将给线程池调度执行SocketProcessorBase的run()方法，或者手动执行
            1068      public boolean processSocket(SocketWrapperBase<S> ,
                                            SocketEvent event, boolean dispatch) {
                             try {
                                 if (socketWrapper == null) {
                                     return false;
                                 }
                                 // 这里获取SocketProcessorBase 是一个socket处理抽象类，实现runnable接口，用来                理socket，读取http报文，生成request或response，。携带一个泛型对象,通过泛型决                定运行时子类？
                                 SocketProcessorBase<S> sc = null;
                                 if (processorCache != null) {
                                     sc = processorCache.pop();
                                 }
            
                                 if (sc == null) {
                                     sc = createSocketProcessor(socketWrapper, event);
                                 } else {
                                     sc.reset(socketWrapper, event);
                                 }
                                 Executor executor = getExecutor();
                                 // 线程池调度执行SocketProcessorBase实现的run方法
                                 if (dispatch && executor != null) {
                                     executor.execute(sc);
                                 } else {
                                     // 手动执行SocketProcessorBase实现的run方法
                                     sc.run();
                                 }
                         ...
                             
                 ................          
                    
            ```

            

          

   - 浏览至： NioEndpoint 743行