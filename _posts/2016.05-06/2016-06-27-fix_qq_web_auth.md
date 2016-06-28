---
layout: post
title: 解决QQ登录SDK不能网页授权登录的问题
date: 2016-06-27 10:25:10
categories: [Android]
tags: [Context]
---
### 现象
QQ登录SDK在用户设备没有安装手机QQ客户端的情况下，默认是会调起网页授权的，但是可能是因为腾讯的某些限制，新申请的app_id都无法使用网页授权，打开后有些是跳转到下载手Q页面，有些是一直显示“正在打开授权登录页...”
<!--more-->

### 分析
试了下市场上其他app使用QQ登录的情况，发现京东的客户端，在未安装客户端的情况下是可以打开网页授权的，抓包得到京东的授权链接：
>
[http://openmobile.qq.com/oauth2.0/m_authorize?status_os=5.0.2&client_id=100273020&status_userip=fec0%3A%3A10%3A5530%3Aabe3%3Abbb9%3Ac98e%2516&format=json&switch=1&status_version=21&appid_for_getting_config=100273020&status_machine=x600&pf=openmobile_android&sdkp=a&sdkv=2.4.lite&sign=88d1d495ffc03ae69a9339367172a5bf&time=1467062773&scope=all&redirect_uri=auth%3A%2F%2Ftauth.qq.com%2F&display=mobile&response_type=token&cancel_display=1](http://openmobile.qq.com/oauth2.0/m_authorize?status_os=5.0.2&client_id=100273020&status_userip=fec0%3A%3A10%3A5530%3Aabe3%3Abbb9%3Ac98e%2516&format=json&switch=1&status_version=21&appid_for_getting_config=100273020&status_machine=x600&pf=openmobile_android&sdkp=a&sdkv=2.4.lite&sign=88d1d495ffc03ae69a9339367172a5bf&time=1467062773&scope=all&redirect_uri=auth%3A%2F%2Ftauth.qq.com%2F&display=mobile&response_type=token&cancel_display=1)

同时在网上找到一个介绍说不用SDK直接打开网页授权的链接：
>[https://openmobile.qq.com/oauth2.0/m_authorize?status_userip=&scope=add_share,add_topic,list_album,upload_pic,get_simple_userinfo&redirect_uri=auth%3A%2F%2Ftauth.qq.com%2F&response_type=token&client_id=100353810](https://openmobile.qq.com/oauth2.0/m_authorize?status_userip=&scope=add_share,add_topic,list_album,upload_pic,get_simple_userinfo&redirect_uri=auth%3A%2F%2Ftauth.qq.com%2F&response_type=token&client_id=100353810)

这个链接也是能打开授权页的，但是参数比较少。

那么问题来了，腾讯到底是怎么限制是否能打开授权登录页的呢？
尝试使用我们自己的app_id替换链接中的`client_id`，发现第二个链接竟然也能打开，那么也就是说应该不是根据app_id来做的限制，那就可能是其他参数了。因此人肉一个个的测试，最终发现是根据`sdkv`这个参数来做限制的。

那么这个值该改成多少呢？

查看了QQ互联SDK的更新记录：

>V1.6  更新了如下内容：
      1.登录方式支持手机QQ登录。
      2.支持通过手机QQ分享信息。
      3.PCPUSH增加手机QQ渠道触达。

V1.6才支持手Q登录，那么之前的sdk应该都是网页授权咯？测试下，最终发现版本号1.4及以下的版本都能打开授权登录页。好了，万事俱备，只欠东风。

### 解决方案
通过字符串查找拼接网页授权url的地方，然后通过反射的方式来修改这个网页授权的url就ok了，具体步骤不展开了，代码如下：

```
    /**
     * 解决网页授权问题
     *
     * @return true: 成功; false: 失败
     */
    private boolean fixWebAuth(Context context, Tencent tencent) {
        if (AppUtils.isAppInstalled(context.getApplicationContext(), "com.tencent.mobileqq")) {
            return true;
        }
        if (tencent == null) {
            return false;
        }
        boolean ok = false;
        try {
            Field aField = tencent.getClass().getDeclaredField("a");
            aField.setAccessible(true);
            Object cVal = aField.get(tencent);
            Field authAgentField = cVal.getClass().getDeclaredField("a");
            authAgentField.setAccessible(true);
            authAgentField.set(cVal, new FixWebAuthAgent(tencent.getQQToken()));
            ok = true;
        } catch (Throwable ignore) {
        }
        return ok;
    }

    /**
     * 解决网页授权问题的AuthAgent
     */
    private static class FixWebAuthAgent extends AuthAgent {

        FixWebAuthAgent(QQToken qqToken) {
            super(qqToken);
        }

        @Override
        protected Bundle a() {
            Bundle bundle = super.a();
            if (bundle != null) {
                // 主要原因在这, 腾讯的server估计是判断这个版本号来进行限制登录的, 从1.5版本后不能打开网页授权了
                bundle.putString("sdkv", "1.4.0");
            }
            return bundle;
        }
    }
```

### 参考：
- [http://blog.csdn.net/cc191954/article/details/9145319](http://blog.csdn.net/cc191954/article/details/9145319)
