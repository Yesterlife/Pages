---
layout: post
title: "解离"
description: "浅析证书锁定机制"
cover_url: https://i.loli.net/2020/05/28/3TG2qs9MhYQxlep.png
cover_meta: illustration by [Waisshu](https://www.pixiv.net/artworks/61298977)
tags: 
  - Develop
  - Android
  - Https
---

近一段时间为了解析某点的计分机制，采用 Charles 对其客户端发送的 Https 请求进行截取，没想到开启 Https 请求截取之后客户端显示无法连接到服务器了。

最后发现客户端通过证书锁定机制以保证连接的合法性。无意中发现这一知识点，就顺便写下这篇文字闲扯一下。

## 截取原理

Https 本身是加密传输的，但并不意味着不存在任何漏洞。

Charles 截获解析 Https 传输的数据利用的是中间人攻击（Man-in-the-middle Attack，简称 MITM），客户端发起 Https 请求会被 Charles 拦截，并发给客户端由 Charles 自己生成的证书，同时 Charles 将伪装成客户端向服务端请求数据。

于是 Charles 担任了 “数据传输代理” 的角色，我们可以在数据传输的过程中查看或修改加密的报文。不过前提条件是，Charles 所伪造的证书需要被客户端信任，否则 Charles 还是无法解析报文明文。

## 证书锁定

Charles 伪造的证书需要被客户端信任，但如果客户端说不呢？

如果是浏览器，会弹出安全警告提醒证书被伪造；应用如果不信任证书，也可以断开连接。但是，在上文说述的事例中，Charles 的证书已经被手动安装并被操作系统信任了，为什么客户端仍旧断开连接？

可能你已经猜出来了。不像操作系统或者浏览器需要连接未知的服务器，应用程序仅需要信任它所连接的几个服务器的证书就行。

这便是证书锁定（也可叫证书固定，SSL Pinning，或者准确一点说是其中的一种，Certificate Pinning），应用程序的代码仅接受指定域名的证书，而不接受操作系统或浏览器内置的其它证书，以保障与服务端通信的唯一性和安全性。

### 实现方式

虽然不清楚某点客户端的实现方式，但是在自己的圈子里还是可以了解到的。

如果采用 HttpsURLConnection，Android 开发者文档给出了使用华盛顿大学机构 CA 的完整示例。

``` kotlin
// Load CAs from an InputStream
// (could be from a resource or ByteArrayInputStream or ...)
val cf: CertificateFactory = CertificateFactory.getInstance("X.509")
// From https://www.washington.edu/itconnect/security/ca/load-der.crt
val caInput: InputStream = BufferedInputStream(FileInputStream("load-der.crt"))
val ca: X509Certificate = caInput.use {
    cf.generateCertificate(it) as X509Certificate
}
System.out.println("ca=" + ca.subjectDN)

// Create a KeyStore containing our trusted CAs
val keyStoreType = KeyStore.getDefaultType()
val keyStore = KeyStore.getInstance(keyStoreType).apply {
    load(null, null)
    setCertificateEntry("ca", ca)
}

// Create a TrustManager that trusts the CAs inputStream our KeyStore
val tmfAlgorithm: String = TrustManagerFactory.getDefaultAlgorithm()
val tmf: TrustManagerFactory = TrustManagerFactory.getInstance(tmfAlgorithm).apply {
    init(keyStore)
}

// Create an SSLContext that uses our TrustManager
val context: SSLContext = SSLContext.getInstance("TLS").apply {
    init(null, tmf.trustManagers, null)
}

// Tell the URLConnection to use a SocketFactory from our SSLContext
val url = URL("https://certs.cac.washington.edu/CAtest/")
val urlConnection = url.openConnection() as HttpsURLConnection
urlConnection.sslSocketFactory = context.socketFactory
val inputStream: InputStream = urlConnection.inputStream
copyInputStreamToOutputStream(inputStream, System.out)
```

Square 也提供了用于在 OkHttp 或 Retrofit 中使用的 `CertificatePinner`，官方采用的例子是 `https://publicobject.com`。

``` kotlin
val hostname = "publicobject.com";
val certificatePinner = CertificatePinner.Builder()
    .add(hostname, "sha256/afwiKY3RxoMmLkuRW1l7QsPZTJPwDS2pdDROQjXw8ig=")
    .add(hostname, "sha256/klO23nT2ehFDXCfx3eHTDRESMz3asj1muO+4aIdjiuY=")
    .add(hostname, "sha256/grX4Ta9HpZx6tSHkmCrvpApTQGo67CYDnvprLg5yRME=")
    .add(hostname, "sha256/lCppFqbkrlJ3EcVFAkeip0+44VaoJUymbnOaEUk7tEU=")
    .build()
val client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()

val request = Request.Builder()
    .url("https://$hostname")
    .build()
client.newCall(request).execute()
```

或者通过证书文件创建 `CertificatePinner`。

``` kotlin
fun createCertificatePinnerByCert(): CertificatePinner {

    val cf: CertificateFactory = CertificateFactory.getInstance("X.509")
    val caInput: InputStream = BufferedInputStream(FileInputStream("certificate.crt"))
    val ca: X509Certificate? = caInput.use {
        cf.generateCertificate(it) as X509Certificate
    }
    caInput.close()

    val certPin = ca?.run {
        CertificatePinner.pin(this)
    } ?: ""

    return CertificatePinner.Builder()
        .add(UrlConfig.RELEASE_BASE_URL, certPin)
        .build()
}
```

### 证书锁定的弊端

~~那么，代价是什么呢？~~

证书锁定需要指定特定的证书，这意味着服务端的证书难以进行更替。

更要命的是，CA 颁发的证书是具备有效期的，证书过期之后需要使用新证书重新打包上架应用。

所以使用证书锁定一定要经过服务端管理员的同意，否则会导致不小的问题。~~或许服务端管理员会提着刀来见你。~~

## 绕过证书锁定

绕过证书锁定的主要手段是逆向源码或 hook 相关函数实现。后者可通过 Xposed 或 Magisk 等框架实现，这些方案都需要设备获取最高权限（Root 权限），所以对于普通用户相对安全。

## 末言

最重要的一点，**开发过程中不要无脑信任所有证书**。

证书锁定在 Https 上提供了更近一层的防护作用，可以有效防止中间人攻击。虽然永远都不存在完美的方案，但提升破解成本，提高应用安全性，这就足够了。