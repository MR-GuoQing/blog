Okhttp源码解析

Thursday, August 16, 2018
7:04 PM

1、okhttp使用流程
	OkhttpClient client = new OkHttpClient();
	Request request = new Request().Builder().get().url(url).build();
	clent.newCall(request).execute();
首先初始化OkhttpClient，新建Request，OkhttpClient和request都是采用构建者模式，可以通过Builder()来添加、配置相关的参数，可以。通过client 的newCall方法将request封装成一个call对象（Call代表了一个准备执行的请求），然后调用call的同步execute()或异步enqueue()方法发送请求。
2、Okhttp发送请求的流程
首先分析call的异步请求，call是个接口，从它的实现类RealCall的enqueue()方法开始。

enqueue方法会调用dispatcher的enqueue方法（dispatch后面分析），dispatch的enqueue主要是将callback加入它的异步队列中，而callback是被封装成AsyncCall的。AsyncCall继承自Runnable接口的线程，所以逻辑在其run方法中。

可以看到主要是调用getResponseInterceptorChain()方法，然后将response的结果放入callback中。

getResponseInterceptorChain()方法是处理拦截器，依次将client.interceptors()(用户自定义的拦截器)、retryandFollowUpInterceptors(重试及重定向)、BrigeInterceptor、CachInterceptor、ConnectInterceptor、CallServerInterceptor添加到List中实现不同的功能。然后新建一个RealInterceptorChain，并调用其proceed方法，以使各个interceptor顺序执行。Interceptor的实现使用的是责任链模式，下面具体分析RealInterceptorChain。
3、InterceptorChain

Private final List<Interceptor>interceptors;   // 拦截器容器
Private final StreamAllocation streamAllocation;//流管理器
Private final HttpCodec httpCodec;//http 流
Private final RealConnection connection;//Socket链接
Private final int index;//Interceptor 当前索引
Private final Request request;//请求
Private int calls;//proceed 方法执行的次数
RealInterceptorChain继承自Chain接口，成员变量StreamAllocation、HttpCodec、RealConnection用于Socket链接，主要是proceed方法

首先新建一个RealInterceptorChain用于下一个interceptor的调用，通过index+1来确保每次调用proceed方法是使用的是不同的interceptor，最后调用拦截器的intercept方法。下图是retryandfollowupinterceptor拦截器的intercept方法。

可以看出拦截器会调用chain 的proceed方法，并把request传递到下一个拦截器中，确保每个interceptor被执行。
4、Dispatcher

Dispathcer主要用来异步请求中线程的管理策略，dispatcher的成员变量如图所示，maxRequest代表最大的并发请求数64，maxRequestPerHost每个主机的最大连接数位5个，executorService线程池，readyAsyncCalls等待执行的异步队列，runningAsyncCalls正在执行的异步队列，runningSyncCalls正在执行的同步队列.
