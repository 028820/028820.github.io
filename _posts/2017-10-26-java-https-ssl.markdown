---
layout: post
title:  Java HTTPS之SSL性能缺陷及修复
date:   2017-10-26
categories: blog

---

由于微服务的兴起，大多数API都是基于HTTP协议，方便不同的语言的客户端调用。
起初我们使用老牌[Apahce CXF](http://cxf.apache.org){:target="_blank"}生成HTTP Client Proxy配合[Swagger](https://swagger.io){:target="_blank"}生成API文档，当一个接口完成时，API文档也就完成了，并且能在浏览器上直接调用调式，相对于传统Java RPC开发效率高很多。

开发效率和性能似乎总是按某一种比例成反比关系。在一次性能测试中，发现[Apahce CXF](http://cxf.apache.org){:target="_blank"}生成 HTTP Client Proxy效率非常低(通过反射解析REST接口，效率不低才怪) 于是我们想将生成的Proxy缓存下来。之前没这样做是因为在生成Proxy需要动态读取域名，适应一些特殊场景(如:QA环境某个服务切换到开发人员本机)。一次官方文档，发现[Apahce CXF](http://cxf.apache.org){:target="_blank"}明确说HTTP Client有[Thread-safty](http://cxf.apache.org/docs/jax-rs-client-api.html#JAX-RSClientAPI-ThreadSafety){:target="_blank"}问题，然而提供的解决办法是Copy一份实例，这虽然解决了反射解析代理的性能问题，但又带来内存频繁使用销毁问题，所以需要找替代方案。

最终我们用[Jersey](https://jersey.github.io/){:target="_blank"}替换了[Apahce CXF](http://cxf.apache.org){:target="_blank"}解决了Proxy复用问题，并且也能很好的满足动态切换制定HTTP服务地址需求。

事情并没有结束，当我们将HTTP升级为HTTPS服务时，又出现问题了，在压测中发现JVM里名称为HandshakeCompletedNotify-Thread的线程大量频繁创建和死亡并且执行时间很短。这是一个很严重的问题。尤其是当你限制了JVM线程数量和设置了较大的线程堆栈后，会造成线程数浪费和内存频繁创建销毁从而增加GC频率降低性能。

经过一系列的定位排查，定位到这是由于[sun.security.ssl.SSLSocketImpl](http://hg.openjdk.java.net/jdk8u/jdk8u60/jdk/file/tip/src/share/classes/sun/security/ssl/SSLSocketImpl.java#l1084){:target="_blank"}造成的。

```
	//
	// Tell folk about handshake completion, but do
	// it in a separate thread.
	//
	if (handshakeListeners != null) {
	    HandshakeCompletedEvent event =
	        new HandshakeCompletedEvent(this, sess);
	
	    Thread t = new NotifyHandshakeThread(
	        handshakeListeners.entrySet(), event);
	    t.start();
	}
```
因为[Jersey](https://jersey.github.io/){:target="_blank"}底层依赖
[sun.net.www.protocol.https.HttpsClient](http://hg.openjdk.java.net/jdk8u/jdk8u60/jdk/file/48e79820c798/src/share/classes/sun/net/www/protocol/https/HttpsClient.java#l751){:target="_blank"}实现了HandshakeCompletedListener回调接口。
```
    /**
     * This method implements the SSL HandshakeCompleted callback,
     * remembering the resulting session so that it may be queried
     * for the current cipher suite and peer certificates.  Servers
     * sometimes re-initiate handshaking, so the session in use on
     * a given connection may change.  When sessions change, so may
     * peer identities and cipher suites.
     */
    public void handshakeCompleted(HandshakeCompletedEvent event)
    {
        session = event.getSession();
    }
```
然而回调实现仅仅是记录SSL证书秘钥相关东西，其实并不需要通过新建线程的方式去Call。并且我们判断任何基于SSL握手回调实现，不会消耗太长时间，也不应该消耗太长时间。所以我们决定将[sun.security.ssl.SSLSocketImpl](http://hg.openjdk.java.net/jdk8u/jdk8u60/jdk/file/tip/src/share/classes/sun/security/ssl/SSLSocketImpl.java#l1084){:target="_blank"}源码修改成线程池处理回调消息，并支持线程数和默认参数，打到我们Java Docker镜像里，升级全部Java运行环境解决掉这个大坑。