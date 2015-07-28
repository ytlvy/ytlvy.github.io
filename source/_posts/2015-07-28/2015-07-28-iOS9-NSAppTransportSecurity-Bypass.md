title: iOS9 NSAppTransportSecurity Bypass
categories: IOS
tags:
  - ATS
  - IOS9
date: 2015-07-28 21:38:09
author:
description:
photos:
---

## Configuring App Transport Security Exceptions in iOS 9 and OSX 10.11

### What is App Transport Security (ATS)?
At WWDC 2015, Apple announced “App Transport Security” for iOS 9 and OSX 10.11 El Capitan. The [“What’s New in iOS”](https://developer.apple.com/library/prerelease/ios/releasenotes/General/WhatsNewIniOS/Articles/iOS9.html#//apple_ref/doc/uid/TP40016198-SW1) guide for iOS 9 explains:

> App Transport Security (ATS) lets an app add a declaration to its Info.plist file that specifies the domains with which it needs secure communication. ATS prevents accidental disclosure, provides secure default behavior, and is easy to adopt. You should adopt ATS as soon as possible, regardless of whether you’re creating a new app or updating an existing one.
> If you’re developing a new app, you should use HTTPS exclusively. If you have an existing app, you should use HTTPS as much as you can right now, and create a plan for migrating the rest of your app as soon as possible.

In simple terms, this means that if your application attempts to connect to any HTTP server (in this example, yourserver.com) that doesn’t support the latest SSL technology (TLSv1.2), your connections will fail with an error like this:
```
CFNetwork SSLHandshake failed (-9801)
Error Domain=NSURLErrorDomain Code=-1200 "An SSL error has occurred and a secure connection to the server cannot be made." UserInfo=0x7fb080442170 {NSURLErrorFailingURLPeerTrustErrorKey=<SecTrustRef: 0x7fb08043b380>, NSLocalizedRecoverySuggestion=Would you like to connect to the server anyway?, _kCFStreamErrorCodeKey=-9802, NSUnderlyingError=0x7fb08055bc00 "The operation couldn’t be completed. (kCFErrorDomainCFNetwork error -1200.)", NSLocalizedDescription=An SSL error has occurred and a secure connection to the server cannot be made., NSErrorFailingURLKey=https://yourserver.com, NSErrorFailingURLStringKey=https://yourserver.com, _kCFStreamErrorDomainKey=3}
```
Curiously, you’ll notice that the connection attempts to change the http protocol to https to protect against mistakes in your code where you may have accidentally misconfigured the URL. In some cases, this might actually work, but it’s also confusing.

### __WARNING: ATS is good for you and your users and you shouldn’t disable it!__

The reason why Apple is pushing so aggressively to force secure connections is because it’s the right thing to do. Protecting personal data from being compromised over insecure wireless connections, among other things, is great for users. Just because these exceptions exist doesn’t mean you should actually use them.

If your application is connecting to third party APIs that you can’t control (such as in my case, where my application Routesy connects to public transit APIs that don’t yet support SSL) or serving as a means to load syndicated content (a browser or a news reader, for instance), these techniques might be useful to you.

The bottom line is, if you run your own API server, FIX YOUR SSL. Thanks to Dave DeLong for reminding me that I should clarify that disabling ATS is a bad idea.

That being said…
<!-- more -->
### How to Bypass App Transport Security
Unfortunately, the pre-release documentation doesn’t currently include any references to this key, so many developers who are testing their preexisting apps with the new betas have been receiving this error and aren’t sure what to do about it. Thanks to some digging through the strings in the CFNetwork executable bundled with Xcode 7, I was able to find the keys necessary to configure your Info.plist.

Per-Domain Exceptions

To configure a per-domain exception so that your app can connect to a non-secure (or non TLSv1.2-enabled secure host), add these keys to your Info.plist (and note that Xcode doesn’t currently auto-complete these keys as of the first Xcode 7 beta seed):

```
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSExceptionDomains</key>
  <dict>
    <key>yourserver.com</key>
    <dict>
      <!--Include to allow subdomains-->
      <key>NSIncludesSubdomains</key>
      <true/>
      <!--Include to allow insecure HTTP requests-->
      <key>NSTemporaryExceptionAllowsInsecureHTTPLoads</key>
      <true/>
      <!--Include to specify minimum TLS version-->
      <key>NSTemporaryExceptionMinimumTLSVersion</key>
      <string>TLSv1.1</string>
    </dict>
  </dict>
</dict>
```

There are other keys that you can use to configure App Transport Security as well, such as:
```
NSRequiresCertificateTransparency
NSTemporaryExceptionRequiresForwardSecrecy
NSTemporaryThirdPartyExceptionAllowsInsecureHTTPLoads
NSTemporaryThirdPartyExceptionMinimumTLSVersion
NSTemporaryThirdPartyExceptionRequiresForwardSecrecy
```

When the Apple documentation is updated, you should familiarize yourself with these other keys and how they’re used. Also, note that some of these keys were listen incorrectly in the “Privacy and Your App” session at WWDC 2015 (NSExceptionAllowsInsecureHTTPLoads instead of NSTemporaryExceptionAllowsInsecureHTTPLoads, for instance). The keys listed above are the correct ones.

### But What If I Don’t Know All the Insecure Domains I Need to Use?

If your app (a third-party web browser, for instance) needs to load arbitrary content, Apple provides a way to disable ATS altogether, but I suspect it’s wise for you to use this capability sparingly:
```
<key>NSAppTransportSecurity</key>
<dict>
    <!--Connect to anything (this is probably BAD)-->
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```


