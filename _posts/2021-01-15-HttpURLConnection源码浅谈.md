---
layout: post
title:  "HttpUrlConnection源码浅谈"
date:   2021-01-15 19:37:00 +0800
categories: android
tags:   android
description:
---

### **前言**
熟悉android源码的同学应该知道HttpUrlConnection的底层实现从android4.4开始就由httpclient   
替换成了okhttp。所以在分析HttpUrlConnection源码的同时，也是在阅读okhttp的源码。分析的方法   
是先画出时序图，然后结合时序图去分析关键方法中的关键代码。   

{%highlight java%}
URL url = new URL("http://yoururl.com");
HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();
conn.setReadTimeout(10000);
conn.setConnectTimeout(15000);
conn.setRequestMethod("POST");
conn.setDoInput(true);
conn.setDoOutput(true);

List<NameValuePair> params = new ArrayList<NameValuePair>();
params.add(new BasicNameValuePair("firstParam", paramValue1));
params.add(new BasicNameValuePair("secondParam", paramValue2));
params.add(new BasicNameValuePair("thirdParam", paramValue3));

OutputStream os = conn.getOutputStream();
BufferedWriter writer = new BufferedWriter(
        new OutputStreamWriter(os, "UTF-8"));
writer.write(getQuery(params));
writer.flush();
writer.close();
os.close();

conn.connect();
{%endhighlight%}
本文基于android4.4.4源码分析，各个版本代码略有不同。  

---

### **时序图**
![p]({{ site.baseurl }}/assets/images/2021-pic/p4.png)  

### **关键方法**
**1.new URL(String spec)**
![p1]({{ site.baseurl }}/assets/images/2021-pic/p1.jpg)

**2.setupStreamHandler()**
![p]({{ site.baseurl }}/assets/images/2021-pic/p2.png)  
可以看到选择的是HttpHandler

**13.initHttpEngine()**
![p]({{ site.baseurl }}/assets/images/2021-pic/p3.png)

**14.execute(false)**
{%highlight java%}
/**
 * Sends a request and optionally reads a response. Returns true if the
 * request was successfully executed, and false if the request can be
 * retried. Throws an exception if the request failed permanently.
 */
private boolean execute(boolean readResponse) throws IOException {
  try {
    httpEngine.sendRequest();
    if (readResponse) {
      httpEngine.readResponse();
    }

    return true;
  } catch (IOException e) {
    ...
}
{%endhighlight%}  

**15.sendRequest()**
{%highlight java%}
public final void sendRequest() throws IOException {
  if (responseSource != null) {
    return;
  }

  prepareRawRequestHeaders(); // 加载常用头部，"Keep-Alive"，"gzip"等，加载cookie
  initResponseSource();
  OkResponseCache responseCache = client.getOkResponseCache();
  if (responseCache != null) {
    responseCache.trackResponse(responseSource);
  }

  // The raw response source may require the network, but the request
  // headers may forbid network use. In that case, dispose of the network
  // response and use a GATEWAY_TIMEOUT response instead, as specified
  // by http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9.4.
  if (requestHeaders.isOnlyIfCached() && responseSource.requiresConnection()) {
    if (responseSource == ResponseSource.CONDITIONAL_CACHE) {
      Util.closeQuietly(cachedResponseBody);
    }
    this.responseSource = ResponseSource.CACHE;
    this.cacheResponse = GATEWAY_TIMEOUT_RESPONSE;
    RawHeaders rawResponseHeaders = RawHeaders.fromMultimap(cacheResponse.getHeaders(), true);
    setResponse(new ResponseHeaders(uri, rawResponseHeaders), cacheResponse.getBody());
  }

  if (responseSource.requiresConnection()) {
    sendSocketRequest();
  } else if (connection != null) {
    client.getConnectionPool().recycle(connection);
    connection = null;
  }
}
{%endhighlight%}    
prepareRawRequestHeaders(); // 加载常用头部，"Keep-Alive"，"gzip"等，加载cookie  

仔细看下initResponseSource()的源码
{%highlight java%}
/**
 * Initialize the source for this response. It may be corrected later if the
 * request headers forbids network use.
 */
private void initResponseSource() throws IOException {
  responseSource = ResponseSource.NETWORK; // 设置responseSource为从网络获取
  if (!policy.getUseCaches()) return; // 禁止使用缓存，return

  OkResponseCache responseCache = client.getOkResponseCache(); // 获取本地缓存
  if (responseCache == null) return; // 缓存为空，则返回

  CacheResponse candidate = responseCache.get(
      uri, method, requestHeaders.getHeaders().toMultimap(false)); // 根据uri,method作为key,获取cache候选人
  if (candidate == null) return;

  Map<String, List<String>> responseHeadersMap = candidate.getHeaders(); // 获取缓存的头部
  cachedResponseBody = candidate.getBody(); // 获取缓存的身部
  if (!acceptCacheResponseType(candidate)
      || responseHeadersMap == null
      || cachedResponseBody == null) {
    Util.closeQuietly(cachedResponseBody); // 关闭cachedResponseBody这个inputStream
    return;
  }

  RawHeaders rawResponseHeaders = RawHeaders.fromMultimap(responseHeadersMap, true);
  cachedResponseHeaders = new ResponseHeaders(uri, rawResponseHeaders);
  long now = System.currentTimeMillis();
  this.responseSource = cachedResponseHeaders.chooseResponseSource(now, requestHeaders);
  if (responseSource == ResponseSource.CACHE) { // 如果是缓存类型，直接setResponse
    this.cacheResponse = candidate;
    setResponse(cachedResponseHeaders, cachedResponseBody);
  } else if (responseSource == ResponseSource.CONDITIONAL_CACHE) { //如果是需要验证的缓存类型，则只是记录cacheResponse
    this.cacheResponse = candidate;
  } else if (responseSource == ResponseSource.NETWORK) { // 不支持缓存，关闭cachedResponseBody这个inputStream
    Util.closeQuietly(cachedResponseBody);
  } else {
    throw new AssertionError();
  }
}
{%endhighlight%}      
client.getOkResponseCache()源码分析，其中client是okHttpClient
{%highlight java%}
public OkResponseCache getOkResponseCache() {
    if (responseCache instanceof HttpResponseCache) {
      return ((HttpResponseCache) responseCache).okResponseCache;
    } else if (responseCache != null) {
      return new OkResponseCacheAdapter(responseCache);
    } else {
      return null;
    }
  }
{%endhighlight%}   

{%highlight java%}
public final class HttpResponseCache extends ResponseCache implements Closeable {

    private final com.android.okhttp.HttpResponseCache delegate;

    private HttpResponseCache(com.android.okhttp.HttpResponseCache delegate) {
        this.delegate = delegate;
    }

    /**
     * Returns the currently-installed {@code HttpResponseCache}, or null if
     * there is no cache installed or it is not a {@code HttpResponseCache}.
     */
    public static HttpResponseCache getInstalled() {
        ResponseCache installed = ResponseCache.getDefault();
        if (installed instanceof com.android.okhttp.HttpResponseCache) {
            return new HttpResponseCache(
                    (com.android.okhttp.HttpResponseCache) installed);
        }

        return null;
    }

    /**
     * Creates a new HTTP response cache and {@link ResponseCache#setDefault
     * sets it} as the system default cache.
     *
     * @param directory the directory to hold cache data.
     * @param maxSize the maximum size of the cache in bytes.
     * @return the newly-installed cache
     * @throws IOException if {@code directory} cannot be used for this cache.
     *     Most applications should respond to this exception by logging a
     *     warning.
     */
    public static HttpResponseCache install(File directory, long maxSize) throws IOException {
        ResponseCache installed = ResponseCache.getDefault();
        if (installed instanceof com.android.okhttp.HttpResponseCache) {
            com.android.okhttp.HttpResponseCache installedCache =
                    (com.android.okhttp.HttpResponseCache) installed;
            // don't close and reopen if an equivalent cache is already installed
            if (installedCache.getDirectory().equals(directory)
                    && installedCache.getMaxSize() == maxSize
                    && !installedCache.isClosed()) {
                return new HttpResponseCache(installedCache);
            } else {
                // The HttpResponseCache that owns this object is about to be replaced.
                installedCache.close();
            }
        }

        com.android.okhttp.HttpResponseCache responseCache =
                new com.android.okhttp.HttpResponseCache(directory, maxSize);
        ResponseCache.setDefault(responseCache);
        return new HttpResponseCache(responseCache);
    }
{%endhighlight%}   
这里使用了代理模式，android.net.http.HttpResponseCache对象代理了com.squareup.okhttp.HttpResponseCache
对象，真正的实现类是由com.squareup.okhttp.HttpResponseCache完成。   

{%highlight java%}
/**
   * Although this class only exposes the limited ResponseCache API, it
   * implements the full OkResponseCache interface. This field is used as a
   * package private handle to the complete implementation. It delegates to
   * public and private members of this type.
   */
  final OkResponseCache okResponseCache = new OkResponseCache() {
    @Override public CacheResponse get(URI uri, String requestMethod,
        Map<String, List<String>> requestHeaders) throws IOException {
      return HttpResponseCache.this.get(uri, requestMethod, requestHeaders);
    }

    @Override public CacheRequest put(URI uri, URLConnection connection) throws IOException {
      return HttpResponseCache.this.put(uri, connection);
    }

    @Override public void maybeRemove(String requestMethod, URI uri) throws IOException {
      HttpResponseCache.this.maybeRemove(requestMethod, uri);
    }

    @Override public void update(
        CacheResponse conditionalCacheHit, HttpURLConnection connection) throws IOException {
      HttpResponseCache.this.update(conditionalCacheHit, connection);
    }

    @Override public void trackConditionalCacheHit() {
      HttpResponseCache.this.trackConditionalCacheHit();
    }

    @Override public void trackResponse(ResponseSource source) {
      HttpResponseCache.this.trackResponse(source);
    }
  };

  public HttpResponseCache(File directory, long maxSize) throws IOException { // HttpResponseCache使用硬盘作为缓存介质
    cache = DiskLruCache.open(directory, VERSION, ENTRY_COUNT, maxSize);
  }
{%endhighlight%}   
可以看出okResponseCache代理了HttpResponseCache.this，也是一个代理模式。所以真正干事的还是com.squareup.okhttp.HttpResponseCache.  
继续看initResponseSource()代码，代码的注释已经讲的很清楚了。基本就是缓存有效，就赋值给HttpEngine的cacheResponse field.

**18.RouteSelector.next()**
{%highlight java%}
public Connection next(String method) throws IOException {
   // Always prefer pooled connections over new connections.
   for (Connection pooled; (pooled = pool.get(address)) != null; ) {
     if (method.equals("GET") || pooled.isReadable()) return pooled;
     pooled.close();
   }

   // Compute the next route to attempt.
   // 这一段代码里比较绕，各种bool值来控制程序流向，具体要仔细看代码，但是没有太大意义。
   // 具体来说就是实现了多host,多代理的循环尝试方案。在后续的递归next(method)进行递归尝试。
   if (!hasNextTlsMode()) {
     if (!hasNextInetSocketAddress()) {
       if (!hasNextProxy()) {
         if (!hasNextPostponed()) {
           throw new NoSuchElementException();
         }
         return new Connection(nextPostponed());
       }
       lastProxy = nextProxy();
       resetNextInetSocketAddress(lastProxy);
     }
     lastInetSocketAddress = nextInetSocketAddress();
     resetNextTlsMode();
   }

   boolean modernTls = nextTlsMode() == TLS_MODE_MODERN;
   Route route = new Route(address, lastProxy, lastInetSocketAddress, modernTls);
   if (routeDatabase.shouldPostpone(route)) {
     postponedRoutes.add(route);
     // We will only recurse in order to skip previously failed routes. They will be
     // tried last.
     return next(method);
   }

   return new Connection(route);
 }

{%endhighlight%}
先看pooled.isReadable()的源码，pooled是Connection实例
{% highlight java %}
/**
   * Returns true if we are confident that we can read data from this
   * connection. This is more expensive and more accurate than {@link
   * #isAlive()}; callers should check {@link #isAlive()} first.
   */
  public boolean isReadable() {
    if (!(in instanceof BufferedInputStream)) {
      return true; // Optimistic.
    }
    if (isSpdy()) {
      return true; // Optimistic. We can't test SPDY because its streams are in use.
    }
    BufferedInputStream bufferedInputStream = (BufferedInputStream) in;
    try {
      int readTimeout = socket.getSoTimeout();
      try {
        socket.setSoTimeout(1);
        bufferedInputStream.mark(1);
        if (bufferedInputStream.read() == -1) {
          return false; // Stream is exhausted; socket is closed.
        }
        bufferedInputStream.reset();
        return true;
      } finally {
        socket.setSoTimeout(readTimeout);
      }
    } catch (SocketTimeoutException ignored) {
      return true; // Read timed out; socket is good.
    } catch (IOException e) {
      return false; // Couldn't read; socket is closed.
    }
  }
{%endhighlight%}
isReadable调用bufferedInputStream.mark(1)，然后去读bufferedInputStream,看是否超时来判断connection的in field是否可读。   
后续的一系列if语句的作用，我在代码的注释里，已经写的很清楚了。

**20.Connection.connect()**
{%highlight java%}
public void connect(int connectTimeout, int readTimeout, TunnelRequest tunnelRequest)
     throws IOException {
   if (connected) {
     throw new IllegalStateException("already connected");
   }
   connected = true;
   socket = (route.proxy.type() != Proxy.Type.HTTP) ? new Socket(route.proxy) : new Socket();
   Platform.get().connectSocket(socket, route.inetSocketAddress, connectTimeout);
   socket.setSoTimeout(readTimeout);
   in = socket.getInputStream();
   out = socket.getOutputStream();

   if (route.address.sslSocketFactory != null) {
     upgradeToTls(tunnelRequest);
   }

   // Use MTU-sized buffers to send fewer packets.
   int mtu = Platform.get().getMtu(socket);
   if (mtu < 1024) mtu = 1024;
   if (mtu > 8192) mtu = 8192;
   in = new BufferedInputStream(in, mtu);
   out = new BufferedOutputStream(out, mtu);
 }
{%endhighlight%}      
就是简单的网络编程知识了，从socket里获取inputStream和outputStream。

**17.HttpEngine.connect()**
{% highlight java %}
protected final void connect() throws IOException {
   if (connection != null) {
     return;
   }
   if (routeSelector == null) {
     String uriHost = uri.getHost();
     if (uriHost == null) {
       throw new UnknownHostException(uri.toString());
     }
     SSLSocketFactory sslSocketFactory = null;
     HostnameVerifier hostnameVerifier = null;
     if (uri.getScheme().equalsIgnoreCase("https")) {
       sslSocketFactory = client.getSslSocketFactory();
       hostnameVerifier = client.getHostnameVerifier();
     }
     Address address = new Address(uriHost, getEffectivePort(uri), sslSocketFactory,
         hostnameVerifier, client.getAuthenticator(), client.getProxy(), client.getTransports());
     routeSelector = new RouteSelector(address, uri, client.getProxySelector(),
         client.getConnectionPool(), Dns.DEFAULT, client.getRoutesDatabase());
   }
   connection = routeSelector.next(method);
   if (!connection.isConnected()) {
     connection.connect(client.getConnectTimeout(), client.getReadTimeout(), getTunnelConfig());
     client.getConnectionPool().maybeShare(connection); // connectionPool添加这个连接
     client.getRoutesDatabase().connected(connection.getRoute()); // failedRoutes.remove(route);
   } else {
     connection.updateReadTimeout(client.getReadTimeout());
   }
   connected(connection);
   if (connection.getRoute().getProxy() != client.getProxy()) {
     // Update the request line if the proxy changed; it may need a host name.
     requestHeaders.getHeaders().setRequestLine(getRequestLine());
   }
 }
{%endhighlight%}
自己添加的中文注释和google工程师的英文注释已经很清楚了。我们重点关注下client.getRoutesDatabase().connected(connection.getRoute())   
其实RouteDatabase这个类的作用很简单，它的作用源码里的介绍已经很清楚了，如下:
{%highlight java%}
/**
 * A blacklist of failed routes to avoid when creating a new connection to a
 * target address. This is used so that OkHttp can learn from its mistakes: if
 * there was a failure attempting to connect to a specific IP address, proxy
 * server or TLS mode, that failure is remembered and alternate routes are
 * preferred.
 */
public final class RouteDatabase {
  private final Set<Route> failedRoutes = new LinkedHashSet<Route>();

  /** Records a failure connecting to {@code failedRoute}. */
  public synchronized void failed(Route failedRoute, IOException failure) {
    failedRoutes.add(failedRoute);

    if (!(failure instanceof SSLHandshakeException)) {
      // If the problem was not related to SSL then it will also fail with
      // a different TLS mode therefore we can be proactive about it.
      failedRoutes.add(failedRoute.flipTlsMode());
    }
  }

  /** Records success connecting to {@code failedRoute}. */
  public synchronized void connected(Route route) {
    failedRoutes.remove(route);
  }

  /** Returns true if {@code route} has failed recently and should be avoided. */
  public synchronized boolean shouldPostpone(Route route) {
    return failedRoutes.contains(route);
  }

  public synchronized int failedRoutesCount() {
    return failedRoutes.size();
  }
}
{%endhighlight%}

**ConnectionPool介绍**    
中间穿插介绍下okhttp的连接缓存池的代码，就看看它的核心recycle方法
{% highlight java%}
public void recycle(Connection connection) {
   if (connection.isSpdy()) { // spdy是http1.2的前身，主要实现了一个socket连接对于多个请求的多路复用
     return;
   }

   if (!connection.isAlive()) {
     Util.closeQuietly(connection);
     return;
   }

   try {
     Platform.get().untagSocket(connection.getSocket()); // 停止对该socket的统计工作
   } catch (SocketException e) {
     // When unable to remove tagging, skip recycling and close.
     Platform.get().logW("Unable to untagSocket(): " + e);
     Util.closeQuietly(connection);
     return;
   }

   synchronized (this) {
     connections.addFirst(connection); // connections中添加connection
     connection.resetIdleStartTime(); // 重置闲置时间为当前
   }

   executorService.submit(connectionsCleanupCallable); // 开启清理任务,清理失效的连接
 }

 /** We use a single background thread to cleanup expired connections. */
  private final ExecutorService executorService = new ThreadPoolExecutor(0, 1,
      60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(),
      Util.daemonThreadFactory("OkHttp ConnectionPool"));
  private final Callable<Void> connectionsCleanupCallable = new Callable<Void>() {
    @Override public Void call() throws Exception {
      List<Connection> expiredConnections = new ArrayList<Connection>(MAX_CONNECTIONS_TO_CLEANUP);
      int idleConnectionCount = 0;
      synchronized (ConnectionPool.this) {
        for (ListIterator<Connection> i = connections.listIterator(connections.size());
            i.hasPrevious(); ) {
          Connection connection = i.previous();
          if (!connection.isAlive() || connection.isExpired(keepAliveDurationNs)) { // connection空闲时间超过5min，则认为连接失效
            i.remove();
            expiredConnections.add(connection);
            if (expiredConnections.size() == MAX_CONNECTIONS_TO_CLEANUP) break;
          } else if (connection.isIdle()) {
            idleConnectionCount++;
          }
        }

        for (ListIterator<Connection> i = connections.listIterator(connections.size());
            i.hasPrevious() && idleConnectionCount > maxIdleConnections; ) {
          Connection connection = i.previous();
          if (connection.isIdle()) {
            expiredConnections.add(connection);
            i.remove();
            --idleConnectionCount;
          }
        }
      }
      for (Connection expiredConnection : expiredConnections) {
        Util.closeQuietly(expiredConnection);
      }
      return null;
    }
  }
{%endhighlight%}

**21.createRequestbody**   
这段代码就是给包装了下20步中connection返回的outputStream,并且把请求头部写入了。

**22.OutputStream os = conn.getOutputStream()**    
这一步就是给21步中封装的socket outputStreanm填充数据了。具体怎么填充，可以参考文章开头的例子。

**30.HttpTransport.getTransferStream()**   
{%highlight java%}
@Override public InputStream getTransferStream(CacheRequest cacheRequest) throws IOException {
    if (!httpEngine.hasResponseBody()) {
      return new FixedLengthInputStream(socketIn, cacheRequest, httpEngine, 0);
    }

    if (httpEngine.responseHeaders.isChunked()) {
      return new ChunkedInputStream(socketIn, cacheRequest, this);
    }

    if (httpEngine.responseHeaders.getContentLength() != -1) {
      return new FixedLengthInputStream(socketIn, cacheRequest, httpEngine,
          httpEngine.responseHeaders.getContentLength());
    }

    // Wrap the input stream from the connection (rather than just returning
    // "socketIn" directly here), so that we can control its use after the
    // reference escapes.
    return new UnknownLengthHttpInputStream(socketIn, cacheRequest, httpEngine);
  }
{%endhighlight%}

{%highlight java%}
final class UnknownLengthHttpInputStream extends AbstractHttpInputStream {
  private boolean inputExhausted;

  UnknownLengthHttpInputStream(InputStream is, CacheRequest cacheRequest, HttpEngine httpEngine)
      throws IOException {
    super(is, httpEngine, cacheRequest);
  }

  @Override public int read(byte[] buffer, int offset, int count) throws IOException {
    checkOffsetAndCount(buffer.length, offset, count);
    checkNotClosed();
    if (in == null || inputExhausted) {
      return -1;
    }
    int read = in.read(buffer, offset, count);
    if (read == -1) {
      inputExhausted = true;
      endOfInput(false);
      return -1;
    }
    cacheWrite(buffer, offset, read);
    return read;
  }
{%endhighlight%}

**可以看到在返回inputStream时，使用了装饰者模式。实际上读取数据只需要socketIn这个inputstream,UnknownLengthHttpInputStream这个包装类的**
**read方法会调用cacheWrite。即用户在读取response的同时，我们将response的body缓存到了本地文件中。此刻我们就完成了网络请求的本地缓存功能。**

{%highlight java%}
AbstractHttpInputStream(InputStream in, HttpEngine httpEngine, CacheRequest cacheRequest)
      throws IOException {
    this.in = in;
    this.httpEngine = httpEngine;

    OutputStream cacheBody = cacheRequest != null ? cacheRequest.getBody() : null;

    // some apps return a null body; for compatibility we treat that like a null cache request
    if (cacheBody == null) {
      cacheRequest = null;
    }

    this.cacheBody = cacheBody;
    this.cacheRequest = cacheRequest;
  }

  protected final void cacheWrite(byte[] buffer, int offset, int count) throws IOException {
    if (cacheBody != null) {
      cacheBody.write(buffer, offset, count);
    }
  }
{%endhighlight%}
这里的cacheRequest就是第28步返回的CacheRequestImpl实例。
而第29步的initContentStream就是给responseBodyIn赋值，也就是上述的包装类UnknownLengthHttpInputStream。
**用户调用conn.getInputStream调用的就是HttpEngine.getResponseBody,即返回responseBodyIn。**

至此，整个httpUrlConnection和其内部的okhttp就分析完成了~
