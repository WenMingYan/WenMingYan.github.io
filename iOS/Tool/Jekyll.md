# Jekyll

> `Jekyll`是一个简单的博客形态的静态站点生产机器。他可以将纯文本转换为静态博客网站。

## `Jekyll`目录

    .
    ├── _config.yml
    ├── _data
    |   └── members.yml
    ├── _drafts
        ├── begin-with-the-crazy-ideas.md
        └── on-simplicity-in-technology.md
    ├── _includes
    |   ├── footer.html
    |   └── header.html
    ├── _layouts
        ├── default.html
        └── post.html
    ├── _posts
    |   ├── 2007-10-29-why-every-programmer-should-play-nethack.md
    |   └── 2009-04-26-barcamp-boston-4-roundup.md
    ├── _sass
        ├── _base.scss
    |   └── _layout.scss
    ├── _site
    ├── .jekyll-metadata
    └── index.html # can also be an 'index.md' with valid YAML Frontmatter

- `_config.yml`：配置文件，你可以在里面配置博客用到的常量，如博客名，邮件等。
- `_includes`：文章各个部分的`html`文件，可以在布局中包含这些文件。
- `layouts`：存放模板。即，网页的布局，主页布局，文章布局。<是指，你包含哪些基本的内容到页面上。包含的内容就是includes里面的文件。>
- `_post`：存放博客文章。
- `index`：博客主页。
- `CNAME文件`：域名地址。
- `CSS`：存放博客所用的`CSS`。
- `JS`：存放博客所用的`JavaScript`。

## 写文章
在`markdown`开头加上：

---
layout: posttitle: "Welcome to Jekyll!"
date: 2014-01-27 21:57:11
categories: Blog
---
文件命名格式: 时间加标题比如：`2015-08-15-HowToBuildBlog.md`
ok，你可以写文章了，放入`_post`文件夹即可。

##选择及修改主题
1. 可以选择已经开发好的主题：[`jekyll`主题集](http://jekyllthemes.org/)
2. 可以自己编写修改<此处需要懂一些前段的知识>

## 使用独立域名
可以通过[GoDaddy](http://www.jianshu.com/p/8f843034c7ec)进行购买，支持支付宝。**在网上可以搜到优惠码。（省钱大法）**
在本地网站文件夹中添加一个文件[CNAME](https://help.github.com/articles/adding-or-removing-a-custom-domain-for-your-github-pages-site/)，写入域名。
国内需要翻墙，需要域名解析，可以选择[Dnspod
                   ](https://www.dnspod.cn/console/dashboard)。添加两条[A记录](https://wapbaike.baidu.com/item/A%E8%AE%B0%E5%BD%95?fr=aladdin&ref=wise&ssid=0&from=1099b&uid=0&pu=usm@0,sz@1320_2001,ta@iphone_1_8.4_3_600&bd_page_type=1&baiduid=C154F2901996C296DF97D4D3F11262E0&tj=Xv_1_0_10_title)。

##添加评论
使用[Disqus](https://disqus.com/)。
- 新建一个名为`Comments`的`html`文件，复制代码，保存到文件夹`_includes`中。然后再`_layouts`的`post`文件中加入。
- 要还想添加一些`feature`就去折腾吧。比如分享、文章目录、代码高亮、标签云、搜索等等。我想到现在，学习这些内容，对你已经很简单了。

## 最后的话
- 多看官方文档
- 遇见问题，先看他人如何解决。




> 参考与资料
> [`jekyll`官方文档](http://jekyllrb.com/docs/home/)
> [`Jekyll`主题](http://jekyllthemes.org/)
> [`Jekyll`主题`github`](https://github.com/mattvh/jekyllthemes)
> [`Bootstrap`前端`html`及`css`学习网站](http://www.bootcss.com/)

