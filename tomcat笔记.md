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
  
  ### connector （链接器）
  
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
              
              
         /// 在Poller run 方法内，调用 events() 方法,从PollerEvent 队列，拿出一个   				     	PollerEvent， 为PollerEvent队中取出的对象 包含的 NioChannel 实例注册    			           SelectionKey.OP_READ 事件
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
                  
         	// 		再有读写事件到来时，调用processKey，将soket交给woker,
            //		sk是一个SelectionKey对   象
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
                 		 *  LinkedBlockingQueue接口（最大值队列长度Integer.Max,属于无界阻塞队列）
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
             754       protected void processKey(SelectionKey sk, 
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
                                  // 这里获取SocketProcessorBase 是一个socket处理抽象类，实runnable                       接口，用来理socket，读取http报文，生成request或response，。携带一个                       泛型对象,通过泛型决定运行时子类？
                                  <S> sc = null;
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
                 
             NioEndpoint:
                  ///进入SocketProcessorBase的run方法的NioEndpoint类的实现，
                       ///准备开始根据协议的不同，适配不同的协议处理类来处理socket
                  		   ...
             1573      if (handshake == 0) {
                           SocketState state = SocketState.OPEN;
                           // Process the request from this socket
                           if (event == null) {
                               // 这里回到AbstractEndpoint接口调用AbstractProtocol中的内部类
                               // ConnectionHandler(org.apache.coyote.AbstractProtocol类中的内部					// 类)的process方法
                               state = getHandler().process(socketWrapper,                                        SocketEvent.OPEN_READ);
                           } else {
                               state = getHandler().process(socketWrapper, event);
                           }
                            ...
                
                ......................
             
             AbstractProtocol:
                 	  /// 进入到AbstractProtocol后的内部类ConnectionHandler的process方法
                 	  /// 方法经过了一系列的准备后,调用AbstractProcessorLight(org.
                       /// apache.coyote.Processor接口的实现类，这个类时是所有协议处理类的基类)
                 	  /// 抽象类的process方法。
                 	  /// 并且，还完成了Processor类（Http11Processor或其他）的实列化工作
                		 	...
                          /// 如果Processor类实列不存在，则新建
                 		 if (processor == null) {
                                 processor = getProtocol().createProcessor();
                                 register(processor);
                          }
                 	    ...
             860          do {
                             state = processor.process(wrapper, status);
                             if (state == SocketState.UPGRADING) {
                         ...
                             
                ......................
                             
             AbstractProcessorLight:
                   /// 调用AbstractProcessorLight抽象service方法，
                   //  调度具体的实现来解析socket中的http协议,之后即将进入协议阶段，socket处理基本上完毕
                        ...
             52      } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
                         state = dispatch(status);
                         if (state == SocketState.OPEN) {
                             // There may be pipe-lined data to read. If the data isn't
                             // processed now, execution will exit this loop and call
                             // release() which will recycle the processor (and input
                             // buffer) deleting any pipe-lined data. To avoid this,
                             // process it now.
                             state = service(socketWrapper);
                         }
                          } 
                         ...
                             
                     } else if (status == SocketEvent.OPEN_READ) {
                             state = service(socketWrapper);
                         ...
                    
               ......................
                             
             Http11Processor：
                  /// service方法接收一个SocketWrapperBase<?>对象（socket），读取socket内的数据内容		 ///（http协议内容），并将读取后的协议内容对象交给Adapter
             271  @Override
                 public SocketState service(SocketWrapperBase<?> socketWrapper)
                     throws IOException {
                     RequestInfo rp = request.getRequestProcessor();
                     rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);
             
                     // Setting up the I/O
                     setSocketWrapper(socketWrapper);
                     ...
                     
                      // Parsing the request header
             291      try {
                         //这里开始解析请求头,不解析body?
                         if (!inputBuffer.parseRequestLine(keptAlive,                                       protocol.getConnectionTimeout(), protocol.getKeepAliveTimeout())){
                             if (inputBuffer.getParsingRequestLinePhase() == -1) {
                                 return SocketState.UPGRADING;
                             } else if (handleIncompleteRequestLineRead()) {
                                 break;
                             }
                         }
                      ...
                      //这里开始解析请求头，应该会解析body?
             309      if (!inputBuffer.parseHeaders()) {
                          // We've read part of the request, don't recycle it
                          // instead associate it with the socket
                          openSocket = true;
                          readComplete = false;
                          break;
                      }
                 	...
                     ///在这里调用CoyoteAdapter的service方法，
                     ///将Request/response对交给container容器处理
                     // Process the request in the adapter
             404     if (getErrorState().isIoAllowed()) {
                         try {
                             rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                             getAdapter().service(request, response);
             
                     ...........
                         
              org.apache.catalina.connector.CoyoteAdapter
                   /// 这里用connector的Request和Response构建实现HttpServletRequest接口的Request
                    // 并进行响应应的转换，然后调用container的invoke方法，传递转换后的Request/Response
             299   @Override
                   public void service(org.apache.coyote.Request req,
                        org.apache.coyote.Response res)throws Exception {
                       Request request = (Request) req.getNote(ADAPTER_NOTES);
                       Response response = (Response) res.getNote(ADAPTER_NOTES);
                       
                         ...
                             
                   // Parse and set Catalina and configuration specific
                   // request parameters
             337   postParseSuccess = postParseRequest(req, request, res, response);
                   if (postParseSuccess) {
                      //check valves if we support async
                      request.setAsyncSupported(connector.getService().
                          getContainer().getPipeline().isAsyncSupported());
                      // Calling the container
                      //调用容器的invoke方法，传递循序是engine->host->conetext->warapper
                      //之后请求便由container处理
                      connector.getService().getContainer().getPipeline().getFirst().invoke(
                          request, response);
                   }
                  
             ```
             
             


### container

- 组成：engine、host、context、wrapper

- 结构：

  ![](./资源\tomcat_container结构.png)

  

可见，一个tomcat 服务器只有一个engine引擎，可以配置多个虚拟主机（host）,每个虚拟主机可以有多个应用程序（context），每个应用可以配置多个servlet（在wrapper中调用）

- 关键代码：

  ```java
  org.apache.catalina.Valve
  		///Valve接口类来调用，各个容器中的invoke实现，调用顺序是connector -> engine -> host
  		/// -> conetext -> wrapper,传递参数为，connector传过来的请求/响应对，调用的这种模式是         /// 责任链模式
  117     public void invoke(Request request, Response response)
                  throws IOException, ServletException;
  		
      ............
  
  org.apache.catalina.core.StandardWrapperValve
  		/// 在engine，host，context，做完过滤后(又没有与请求对应的host与应用),之后将                   /// request/response对交到wrapper手中
          
           
           ///先从栈中取出一个servlet，来处理request，用完后会换回栈中，代码在267行，后略。
           // Allocate a servlet instance to process this request
  133       try {
              if (!unavailable) {
                  servlet = wrapper.allocate();
              }
              ...
          // Call the filter chain for this request
          // NOTE: This also calls the servlet's service() method
  178     Container container = this.container;
          try {
              if ((servlet != null) && (filterChain != null)) {
                  // Swallow output if needed
                  if (context.getSwallowOutput()) {
                      try {
                          SystemLogHandler.startCapture();
                          if (request.isAsyncDispatching()) {
                              request.getAsyncContextInternal().doInternalDispatch();
                          } else {
                              // 这里使用过滤链，来过滤请求，也是责任链模式，在过滤器尽头，则是                               // servlet.service 方法的调用
                              filterChain.doFilter(request.getRequest(),
                                      response.getResponse());
                          }
                      } finally {
                          String log = SystemLogHandler.stopCapture();
                          if (log != null && log.length() > 0) {
                              context.getLogger().info(log);
                          }
                      }
                  } else {
                      if (request.isAsyncDispatching()) {
                          request.getAsyncContextInternal().doInternalDispatch();
                      } else {
                           // 这里使用过滤链，来过滤请求，也是责任链模式
                          filterChain.doFilter
                              (request.getRequest(), response.getResponse());
                      }
                  }
  
              }
              
  	..............
  
  org.apache.catalina.core.ApplicationFilterChain
  	
      /// servlet的过滤链，在wrapper里最先调用，如果有filter1、2、3...n，会按预定好的顺序调用         /// filter，传递request/response对，filter内部的doFilter又会调用此处的doFilter，以此传递     /// 下去
      @Override
  137 public void doFilter(ServletRequest request, ServletResponse response)
          throws IOException, ServletException {
  
          if( Globals.IS_SECURITY_ENABLED ) {
              final ServletRequest req = request;
              final ServletResponse res = response;
              try {
                  java.security.AccessController.doPrivileged(
                      new java.security.PrivilegedExceptionAction<Void>() {
                          @Override
                          public Void run()
                              throws ServletException, IOException {
                              /// 调用具体的dofilter方法，filter的dofilter方方法与转到servlet                             /// 的service方法，均在这个方法内实现
                              internalDoFilter(req,res);
                              return null;
                          }
                      }
                  );
              }
         ...
         
  170    private void internalDoFilter(ServletRequest request,
                                    ServletResponse response)
          throws IOException, ServletException {
  
          // Call the next filter if there is one
          if (pos < n) {
              ApplicationFilterConfig filterConfig = filters[pos++];
              try {
                  Filter filter = filterConfig.getFilter();
  
                  if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                          filterConfig.getFilterDef().getAsyncSupported())) {
                      request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
                  }
                  if( Globals.IS_SECURITY_ENABLED ) {
                      final ServletRequest req = request;
                      final ServletResponse res = response;
                      Principal principal =
                          ((HttpServletRequest) req).getUserPrincipal();
  
                      Object[] args = new Object[]{req, res, this};
                      //调用filter的doFilter方法进行过滤
                      SecurityUtil.doAsPrivilege ("doFilter", filter, classType, args, principal);
                  } else {
                       //调用filter的doFilter方法进行过滤
                      filter.doFilter(request, response, this);
                  }
             ...
          // We fell off the end of the chain -- call the servlet instance
  206    try {
              if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                  lastServicedRequest.set(request);
                  lastServicedResponse.set(response);
              }
  
              if (request.isAsyncSupported() && !servletSupportsAsync) {
                  request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                          Boolean.FALSE);
              }
              // Use potentially wrapped request from this point
              if ((request instanceof HttpServletRequest) &&
                      (response instanceof HttpServletResponse) &&
                      Globals.IS_SECURITY_ENABLED ) {
                  final ServletRequest req = request;
                  final ServletResponse res = response;
                  Principal principal =
                      ((HttpServletRequest) req).getUserPrincipal();
                  Object[] args = new Object[]{req, res}; 
                  // 调用servlet的service方法，处理请求
                  SecurityUtil.doAsPrivilege("service",
                                             servlet,
                                           classTypeUsedInService,
                                             args,
                                             principal);
              } else {
                  // 调用servlet的service方法，处理请求
                  servlet.service(request, response);
              }
  
  ```
  
  