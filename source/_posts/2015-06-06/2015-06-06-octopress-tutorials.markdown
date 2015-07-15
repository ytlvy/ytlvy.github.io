---
layout: post
title: "octopress tutorials"
date: 2015-06-06 14:05:03 +0800
comments: true
categories: Other
tags: blog
---
## Octopress

### 官方文档
[官方文档](http://octopress.org/docs/setup/)

### 准备环境
需要ruby1.9以上,可以选择用rbenv来安装多版本ruby.

#### 安装rbenv
 ```
brew update
brew install rbenv
brew install ruby-build

rbenv install 1.9.3-p0
rbenv local 1.9.3-p0
rbenv rehash
 ```

你有可能需要安装老版本的GCC编译器才能顺利安装Ruby 1.9.3:
```
brew tap homebrew/dupes
brew install apple-gcc42
```

<!--more-->

### 安装Octopress

#### 下载代码
```
git clone git://github.com/imathis/octopress.git octopress
```

修改Gemfile源为:http://ruby.taobao.org 
修改系统gem源
```
gem sources -l
gem sources -r https://rubygems.org    #删除 http://rubygems.org
gem sources -a http://ruby.taobao.org  #添加 http://ruby.taobao.org
```

#### 安装依赖库
```
cd octopress
gem install bundler
bundle install
```

如使用rbenv
```
gem install bundler
rbenv rehash
bundle install
```

#### 安装默认主题
```
rake install
```


#### 新文章
```
rake new_post["Welcome"]
```


#### 编译预览
```
rake generate 
rake preview
```

### 部署到github
#### github 创建同名仓库

#### 本地设置
```
rake setup_github_pages
```
根据提示, 输入项目路径: git@github.com:superman/super.github.io.git

#### 同步代码到github
```
git add .
git commit -m "first commit"
git push origin source   
rake deploy #rake deploy 会直接 push Octopress目录中 master 分支
```


### 写博流程
Octopress 博文必须存储在`source/_posts`目录下，并且需要按照Jekyll的命名规范对文章进行命名：`YYYY-MM-DD-post-title.markdown`。文章的名字会被当做url的一部分，而其中的日期用于对博文的区分和排序。创建和部署博文的一个完整流程：
```
$ rake new_post["New Post"]
$ rake generate
$ git add .
$ git commit -am "add comment here." 
$ git push origin source
$ rake deploy
```

### Octopress DIY 定制
#### 添加侧边栏文章分类
1. 在 plugins 目录中创建`category_list_tag.rb`文件，文件内容如下：
```
module Jekyll
  class CategoryListTag < Liquid::Tag
    def render(context)
      html = ""
      categories = context.registers[:site].categories.keys
      categories.sort.each do |category|
        posts_in_category = context.registers[:site].categories[category].size
        category_dir = context.registers[:site].config['category_dir']
        category_url = File.join(category_dir, category.gsub(/_|\P{Word}/, '-').gsub(/-{2,}/, '-').downcase)
        html << "<li class='category'><a href='/#{category_url}/'>#{category} (#{posts_in_category})</a></li>\n"
      end
      html
    end
  end
end

Liquid::Template.register_tag('category_list', Jekyll::CategoryListTag)
```

侧边栏的 DIY 定制一般都是新建一个 xxx.html 文件，放在source`\_includes\custom\asides`目录下，然后文件模板必须为：
```
<section>
     <h1>xxx</h1>
     <!--添加代码-->
</section>
```
然后在 `_config.yml`文件中，找到default_asides数组，添加`custom/asides/xxx.html` 。侧边栏的排序就根据数组的排序进行 DIY

2. 把下面内容添加到`source/_includes/asides/category_list.html`文件
```
<section> 
  <h1>文章分类</h1> 
  <ul id="categories"> 
    大括号%   category_list  %大括号 
    </ul> 
</section>
```
3. 修改`_config.yml`文件，用 command+f 找到`default_asides`，然后添加`asides/category_list.html`，值之间以逗号隔开。
```
default_asides: [asides/category_list.html, asides/recent_posts.html]
```

#### 添加社会化评论
##### Disqus
1. 登录 Disqus，创建一个账号，然后点击 `Add Disqus to your site`，输入 `Site name` ，例如 boy，然后点击 Finish registration。这里页面会自动生成shortname，记起来，待会有用。

2. 打开_config.yml文件，找到

```
# Disqus Comments 
disqus_short_name:  
disqus_show_comment_count: false
```
设置 disqus_short_name 为刚刚创建的 Disqus 生成的 shortname，例如 boy，
把`false`改为`true`。特别注意，冒号后面要空出一个空格。

##### 多说
用微博登陆多说，然后创建个人站点
在 _config.yml 中添加

```
# duoshuo comments
duoshuo_comments: true
duoshuo_short_name: yourshortname
```

在 `source/_layouts/post.html` 中的 disqus代码
```
 {% if site.disqus_short_name and page.comments == true %} 
      <section>
        <h1>Comments</h1>
        <div id="disqus_thread" aria-live="polite">{% include post/disqus_thread.html %}</div>
      </section>
    {% endif %}
```

下方添加多说评论模块

```

{% if site.duoshuo_short_name and site.duoshuo_comments == true and page.comments == true %}
  <section>
    <h1>Comments</h1>
    <div id="comments" aria-live="polite">{% include post/duoshuo.html %}</div>
  </section>
{% endif %}
```

在`/source/_includes/post/`目录下创建`duoshuo.html`， 添加代码

```
<div class="ds-thread" data-title="{% if site.titlecase %}{{ page.title | titlecase }}{% else %}{{ page.title }}{% endif %}"></div>

    <script type="text/javascript">
      var duoshuoQuery = {short_name:"{{ site.duoshuo_short_name }}"};
      (function() {
        var ds = document.createElement('script');
        ds.type = 'text/javascript';ds.async = true;
        ds.src = 'http://static.duoshuo.com/embed.js';
        ds.charset = 'UTF-8';
        (document.getElementsByTagName('head')[0] 
        || document.getElementsByTagName('body')[0]).appendChild(ds);
      })();
    </script>
```

修改_includes/article.html 文件，在

```javascript
 {% if site.disqus_short_name and page.comments != false and post.comments != false and site.disqus_show_comment_count == true %}
         | <a href="{% if index %}{{ root_url }}{{ post.url }}{% endif %}#disqus_thread">Comments</a>
        {% endif %}

```

下面添加如下代码

```javascript

 {% if site.duoshuo_short_name and page.comments != false and post.comments != false and site.duoshuo_comments == true %}
    <a href="{% if index %}{{ root_url }}{{ post.url }}{% endif %}#comments">Comments</a>
 {% endif %}

```

#### 首页样式修改

source/index.html

```javascript
<!-- This loops through the paginated posts
{% for post in paginator.posts %}
  <h1><a href="{{ post.url }}">{{ post.title }}</a></h1>
  <p class="author">
    <span class="date">{{ post.date }}</span>
  </p>
  <div class="content">
    {{ post.content }}
  </div>
{% endfor %} -->
```


#### 修改导航栏
导航栏的setting在`source\_includes\custom\navigation.html`

我们可以将 Blog 和 Archives 修改为首页和归档，也可以在此添加一个标签页，此时应使用命令`rake new_page['about']`创建一个页面，页面路径为`source\about\index.markdown`;

修改后的文件如下：
```html
<ul class="main-navigation"> 
  <li><a href="/">首页</a></li> 
  <li><a href="/blog/archives">归档</a></li> 
  <li><a href="/about">关于</a></li> 
</ul>
```

#### 添加个人二维码
在侧边栏显示二维码，下载![插件](https://github.com/sailor79/Octopress-dynamic-QR-Code-aside)， 将 `qrcode.html` 放入 `source/_includes/custom/asides/ `中， 在 `_config.yml` 中`default_asides` 添加 `custom/asides/qrcode.html`。 然后打开`qrcode.html`，做`image src`的修改.或者将 `qrcode.html` 代码添加到你想展示的页面的HTML文件中亦可。


#### 添加社会化分享
在www.addthis.com上获取分享按钮代码，粘贴到 `source/_includes/post/sharing.html` 中，例如我的代码如下：
```
<div class="sharing">
  <!-- AddThis Button BEGIN -->
  <div class="addthis_toolbox addthis_default_style addthis_32x32_style">
      <a class="addthis_button_sinaweibo"></a>
      <a class="addthis_button_facebook"></a>
      <a class="addthis_button_twitter"></a>
      <a class="addthis_button_google_plusone_share"></a>
      <a class="addthis_button_delicious"></a>
      <a class="addthis_button_compact"></a><a class="addthis_counter addthis_bubble_style"></a>
  </div>
  <script type="text/javascript" src="//s7.addthis.com/js/300/addthis_widget.js#pubid=undefined" async="async"></script>
  <!-- AddThis Button END -->
</div>
```


#### 文章只显示部分正文
在markdown非代码段, 添加<!--more-->, 其后的内容将不会显示，只会显示<!--more-->以上部分，而且会在文章尾部添加 Read On超链接，如果要更改超链接内容，可以在 `_config.yml`中找到 `Read On`，然后修改为“`继续阅读`”。


#### 添加百度统计或CNZZ统计
到百度统计或CNZZ统计上注册账号，然后添加脚本文件到`source/_includes/after_footer.html`文件中


#### 安装第三方主题
比较常见的有`slash`主题。Slash is a minimal theme for Octopress.

```bash
$ cd octopress
$ git clone git://github.com/tommy351/Octopress-Theme-Slash.git .themes/slash
$ rake install['slash']
$ rake generate

```

![第三方主题汇总](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)

### Octopress 优化
#### 替换Google JS公共库
Octopress 默认使用的是 Google 的 JS 公共库地址，加载的过程特别的慢。所以我们要把它改为百度的 JS 公共库 ，需要把`/source/_includes/head.html` 文件中的 Google 公共库地址改为：
```js
<script src="//libs.baidu.com/jquery/1.7.2/jquery.min.js"></script>
```

#### 去掉Twitter
把在根目录下的 `_config.yml`文件中 Twitter 内容给注释掉。
```js
# Twitter
#twitter_user:
#twitter_tweet_button: true
```

把 `\source\_includes\after_footer.html` 文件中的 Twitter 内容给注释掉：

```
<!--{% include twitter_sharing.html %}-->
```

#### 删除 Google font
把在`\source\_includes\custom\head.html` 中的Google font样式给删除：

```
<link href="//fonts.googleapis.com/css?family=PT+Serif:regular,italic,bold,bolditalic" rel="stylesheet" type="text/css">
<link href="//fonts.googleapis.com/css?family=PT+Sans:regular,italic,bold,bolditalic" rel="stylesheet" type="text/css">
```

#### 搜索优化
为了让自己搭建的博客更容易被搜索引擎搜到，最好将网站地址提交给各大搜索引擎，下面有两个连接搜集了各个搜索引擎的网站提交入口：
```
http://urlc.cn/tool/addurl.html
http://tool.lusongsong.com/addurl.html
```

markdown添加SEO信息

```
tags: [octopress,seo]
keywords:  octopress, analytics
description: 如何搭建Octopress博客
```

### GitCafe 备份
![官方教程](https://gitcafe.com/GitCafe/Help/wiki/Pages-%E7%9B%B8%E5%85%B3%E5%B8%AE%E5%8A%A9#wiki)

#### 在GitCafe上新建一个博客项目
创建与用户名相同名称的项目

#### 提交到 gitcafe

方法一 设置多个Git Remote源
对于Octopress，我们只需要每次提交网站内容时，执行完 rake deploy之后，再执行以下脚本即可

```bash
cd _deploy
git remote add gitcafe git@gitcafe.com:tangqiaoboy/tangqiaoboy.git >> /dev/null 2>&1
git push -u gitcafe master:gitcafe-pages
```


方法二 修改Rakefile
在Rakefile第269行后增加如下代码, `rake deploy`自动提交

```ruby
system "git remote add gitcafe git@gitcafe.com:tangqiaoboy/tangqiaoboy.git >> /dev/null 2>&1"
system "git push -u gitcafe master:gitcafe-pages"
```

#### gitcafe 设置域名
GitCafe的自定义域名设置比github要友好得多，它不但提供了图形界面设置，并且支持同时设置多个域名。在`项目管理`–>`域名管理`中，我们可以找到相应的设置项，如下所示：
![](http://blog.devtang.com/images/gitcafe-set-domain.jpg)
在设置完之后，我们需要去域名解析的服务商那儿，将对应的域名用A记录类型，解析到117.79.146.98即可

#### gitcafe
270 line

```ruby
    system "git remote add gitcafe git@gitcafe.com:ytlvy/ytlvy.git >> /dev/null 2>&1"
    system "git push -u gitcafe master:gitcafe-pages"
```


### Bug

#### 当文章有多个标签时候，执行 rake generate会出现下面情况

```bash
AppledeiMac:octopress apple$ rake generate
##Generating Site with Jekyll
    write source/stylesheets/screen.css
Configuration file: /Users/apple/octopress/_config.yml
            Source: source
       Destination: public
      Generating...
  Liquid Exception: comparison of Array with Array failed in _layouts/page.html
```

解决办法是，在`source/_includes/custom/asides/tags.html`中把`limit`参数去掉。 

#### 另外，如果执行 rake generate出现 `Layout 'nil'`

```bash
Build Warning: Layout 'nil' requested in tags/octopress/atom.xml does not exist.
     Build Warning: Layout 'nil' requested in tags/ios/atom.xml does not exist.
```

修改以下文件中得 `layout: nil` 为 `layout: null`

- source/atom.xml
- source/robots.txt
- source/_includes/custom/category_feed.xml


#### 翻页bug

修改`_config.yml` 中 `paginate_path` 由`\post\:num` 为 `:num`


#### typekit 加载403error
注册typekit账号, 在typekit中添加你自己域名信息
![](http://help.typekit.com/customer/portal/attachments/62656)

获得自己的资源代码

```js
<script src="//use.typekit.net/ilg6roq.js"></script>
<script>try{Typekit.load();}catch(e){}</script>
```