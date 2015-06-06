---
layout: post
title: "MAC 代理设置"
date: 2015-05-27 15:51:23 +0800
comments: true
categories: MAC
---
####Command line proxy
    export http_proxy="http://127.0.0.1:8087"
    export https_proxy="http://127.0.0.1:8087"
    unset http_proxy
    unset https_proxy

### svn proxy
    sublime ~/.subversion/servers