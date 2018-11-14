---
layout:     post                    # 使用的布局（不需要改）
title:      Android Charles 证书固定              # 标题 
subtitle:   NetworkSecurityConfig  #副标题
date:       2018-04-02              # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签   
    - android
    - 网络安全

---
最近在做公司的接口全部换上了https，我们的应用也随之进行了升级，接入https。

但是在使用charles进行接口抓包调试的时候就出现了一些问题。

我的计划是既要能使我们的app在release和debug下都能进行charles抓包调试定位问题，还要能有效的防止其他人对应用进行抓包获取数据。

## 之前做法
由于我们使用了okhttp框架，在此之前我们一直使用okhttp来做证书固定

```
private static final String CA_DOMAIN = "*.mydomain.com";
    private static final String CA_PUBLIC_KEY = "sha256/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";

    .....

    CertificatePinner pinner = new CertificatePinner.Builder()
                .add(CA_DOMAIN, CA_PUBLIC_KEY)
                .build();

    .....
    
    OkHttpClient.Builder clientBuilder = new OkHttpClient.Builder()
            .certificatePinner(pinner);

作者：firzencode
链接：https://www.jianshu.com/p/19f311d81b6d
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

```
这样做的好处是可以一定程度上防止被抓包，但是如果我们自己需要进行调试的时候，则要手动注释掉代码块才行，或者通过某种debug开关来控制，这样的做法在release上也不会生效，所以要另找方案。

## 网络安全性配置
[网络安全性配置](https://developer.android.com/training/articles/security-config.html?hl=zh-cn#TrustingDebugCa)是Android官方推荐的配置方案。
我们来看一段源码，在**NetworkSecurityConfig**类中

```
public static final Builder getDefaultBuilder(int targetSdkVersion) {
        Builder builder = new Builder()
                .setCleartextTrafficPermitted(DEFAULT_CLEARTEXT_TRAFFIC_PERMITTED)
                .setHstsEnforced(DEFAULT_HSTS_ENFORCED)
                // System certificate store, does not bypass static pins.
                .addCertificatesEntryRef(
                        new CertificatesEntryRef(SystemCertificateSource.getInstance(), false));
        // Applications targeting N and above must opt in into trusting the user added certificate
        // store.
        if (targetSdkVersion <= Build.VERSION_CODES.M) {
            // User certificate store, does not bypass static pins.
            builder.addCertificatesEntryRef(
                    new CertificatesEntryRef(UserCertificateSource.getInstance(), false));
        }
        return builder;
    }

```
可以看到对于targetSdkVersion在24及以上的时候，Android证书认证将不接受user级别的，默认配置只支持system级别，而23及以下则会支持user级别。

所以我们根据wiki，首先新建一个xml文件 ***res/xml/network_security_config.xml*** :

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config>
        <trust-anchors>
            <certificates src="user"/>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>
</network-security-config>

```

然后在Androidmanifest.xml配置该文件

```
 <application
        ...
        android:networkSecurityConfig="@xml/network_security_config">
       
       ...
 </application>

```

在使用charles的时候，只要在手机中安装证书，就可以实现抓包，然而这样的做法显然是很不安全的，任何一个人通过反编译代码发现这种配置都可以实现抓包，没有达到我们的目的。所以需要对方案进行改进：
我们首先将网站服务器的证书和charles的证书pem文件抓取下来加入到***res/raw***文件夹下，然后修改 ***res/xml/network_security_config.xml*** :


```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config>
        <trust-anchors>
            <certificates src="system"/>
            <certificates src="@raw/api"/>
            <certificates src="@raw/charles"/>
        </trust-anchors>
    </base-config>
</network-security-config>

```

抓取网站服务器的CA命令如下：

```
openssl s_client -connect mydomain.com:443 -servername mydomain.com | openssl x509 -out test.pem

```
这样的效果就是app只能接受来自system、api及charles的CA，有效地防止了其他人抓包。


### 防反编译重打包
如果有人想要反编译应用，更换raw中的pem文件，在重新打包同样可以实现抓包，这个时候就需要一些逆向安全机制，我们需要验证签名的MD5是否和公司签名MD5一致，如果出现重打包的情况，就要自动杀死应用，终止应用的运行。

### 参考内容：

1、[证书固定、CertificatePinner与Charles抓包的问题](https://www.jianshu.com/p/19f311d81b6d)

2、[如何绕过安卓的网络安全配置功能](http://www.freebuf.com/articles/terminal/165671.html)

3、[网络安全性配置](https://developer.android.com/training/articles/security-config.html?hl=zh-cn#TrustingDebugCa)

4、[Android 代理系统级 HttpUrlConnection 请求到第三方网络库 - 支持 Http/2.0 及 HttpDNS](https://fucknmb.com/2017/07/13/Android%E4%BB%A3%E7%90%86%E7%B3%BB%E7%BB%9F%E7%BA%A7HttpUrlConnection%E8%AF%B7%E6%B1%82%E5%88%B0%E7%AC%AC%E4%B8%89%E6%96%B9%E7%BD%91%E7%BB%9C%E5%BA%93-%E6%94%AF%E6%8C%81Http-2-0%E5%8F%8AHttpDNS/)