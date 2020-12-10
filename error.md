# 错误信息

```java
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class java.util.Collections$3 and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS) (through reference chain: org.springframework.web.multipart.support.StandardMultipartHttpServletRequest["request"]->org.apache.catalina.connector.RequestFacade["attributeNames"])
	at com.fasterxml.jackson.databind.exc.InvalidDefinitionException.from(InvalidDefinitionException.java:77) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.SerializerProvider.reportBadDefinition(SerializerProvider.java:1191) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.DatabindContext.reportBadDefinition(DatabindContext.java:313) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.impl.UnknownSerializer.failForEmpty(UnknownSerializer.java:71) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.impl.UnknownSerializer.serialize(UnknownSerializer.java:33) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.BeanPropertyWriter.serializeAsField(BeanPropertyWriter.java:727) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.std.BeanSerializerBase.serializeFields(BeanSerializerBase.java:719) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(BeanSerializer.java:155) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.BeanPropertyWriter.serializeAsField(BeanPropertyWriter.java:727) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.std.BeanSerializerBase.serializeFields(BeanSerializerBase.java:719) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(BeanSerializer.java:155) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.DefaultSerializerProvider._serialize(DefaultSerializerProvider.java:480) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ser.DefaultSerializerProvider.serializeValue(DefaultSerializerProvider.java:319) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ObjectWriter$Prefetch.serialize(ObjectWriter.java:1396) ~[jackson-databind-2.9.7.jar:2.9.7]
	at com.fasterxml.jackson.databind.ObjectWriter.writeValue(ObjectWriter.java:913) ~[jackson-databind-2.9.7.jar:2.9.7]
	at org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.writeInternal(AbstractJackson2HttpMessageConverter.java:285) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.http.converter.AbstractGenericHttpMessageConverter.writeInternal(AbstractGenericHttpMessageConverter.java:111) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.http.converter.AbstractHttpMessageConverter.write(AbstractHttpMessageConverter.java:226) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.cloud.openfeign.support.SpringEncoder.encode(SpringEncoder.java:111) ~[spring-cloud-openfeign-core-2.0.2.RELEASE.jar:2.0.2.RELEASE]
	at feign.ReflectiveFeign$BuildEncodedTemplateFromArgs.resolve(ReflectiveFeign.java:368) ~[feign-core-9.7.0.jar:?]
	at feign.ReflectiveFeign$BuildTemplateByResolvingArgs.create(ReflectiveFeign.java:223) ~[feign-core-9.7.0.jar:?]
	at feign.SynchronousMethodHandler.invoke(SynchronousMethodHandler.java:73) ~[feign-core-9.7.0.jar:?]
	at feign.ReflectiveFeign$FeignInvocationHandler.invoke(ReflectiveFeign.java:102) ~[feign-core-9.7.0.jar:?]
	at com.sun.proxy.$Proxy141.insertsRequest(Unknown Source) ~[?:?]
	at com.gaohwangh.consumer.service.PapersConsumerService.insertsRequest(PapersConsumerService.java:40) ~[classes/:?]
	at com.gaohwangh.consumer.controller.PapersController.insertsRequest(PapersController.java:56) ~[classes/:?]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_111]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_111]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_111]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_111]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:209) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:136) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:102) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:891) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:797) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:991) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:925) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:981) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:884) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:707) ~[javax.servlet-api-3.1.0.jar:3.1.0]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:858) ~[spring-webmvc-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:790) ~[javax.servlet-api-3.1.0.jar:3.1.0]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52) ~[tomcat-embed-websocket-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.springframework.boot.actuate.web.trace.servlet.HttpTraceFilter.doFilterInternal(HttpTraceFilter.java:90) ~[spring-boot-actuator-2.0.7.RELEASE.jar:2.0.7.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.springframework.web.filter.HttpPutFormContentFilter.doFilterInternal(HttpPutFormContentFilter.java:109) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:93) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.filterAndRecordMetrics(WebMvcMetricsFilter.java:117) ~[spring-boot-actuator-2.0.7.RELEASE.jar:2.0.7.RELEASE]
	at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.doFilterInternal(WebMvcMetricsFilter.java:106) ~[spring-boot-actuator-2.0.7.RELEASE.jar:2.0.7.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.11.RELEASE.jar:5.0.11.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:198) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:493) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:140) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:342) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:800) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:806) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1498) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [?:1.8.0_111]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [?:1.8.0_111]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-8.5.35.jar:8.5.35]
	at java.lang.Thread.run(Thread.java:745) [?:1.8.0_111]

```

fastjson

```java
java.lang.IllegalStateException: It is illegal to call this method if the current request is not in asynchronous mode (i.e. isAsyncStarted() returns false)
```

原因:

```properties
使用fastjson报的错误
原因是序列化了

ServletRequest
ServletResponse
MultipartFile
```

