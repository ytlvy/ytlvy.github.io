---
layout: post
title: "URL Identifier &amp;&amp; URL Schemes"
date: 2015-05-27 15:47:04 +0800
comments: true
categories: IOS
---
## URL Identifier && URL Schemes
    The URL Identifier is the reversed domain address which is should be the same as your Bundle Identifier e.g. com.companyname.appname

    The URL Schemes is the start of the URL e.g 'appname'. When you call this as a URL it targets the bundle identifier which launches the app.

    so... 'appname://' (in safari) will call com.companyname.appname