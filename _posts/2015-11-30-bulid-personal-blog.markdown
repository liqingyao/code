---
layout: post
title:  "Jekyll 个人静态博客建站及优化"
date:   2015-11-20 15:34:32
categories: 其他
comments: true
---
写博客要追溯到高中，用百度空间记录生活和想法，坚持了四年多，直到大二时被盗号，申诉多次没有追回帐号，就没再写过。研二重整旗鼓，买了域名和服务，在 WordPress 上搭了个人博客，想模仿牛人写些技术博客。最近在面试时，面试官提到了我的博客，建议我用 GitHub 和 Jekyll 建站试试，于是我就开始捯饬着搬迁和建立工作。

发现自己经历了[阮一峰](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)说的写博客的三个阶段:

>第一阶段，刚接触 Blog，觉得很新鲜，试着选择一个免费空间来写。
>
>第二阶段，发现免费空间限制太多，就自己购买域名和空间，搭建独立博客。
>
>第三阶段，觉得独立博客的管理太麻烦，最好在保留控制权的前提下，让别人来管，自己只负责写文章。

从零开始搭博客挺好玩，也可以趁此机会学一些前端。按照[教程](http://beiyuu.com/github-pages/)一步步走完后，博客初见规模。要做到更好还要考虑性能优化、网站分析和其他功能，有很多可以挖掘的，完全看个人喜好和发挥。当然最重要还是内容。弄完后才知道一篇华丽丽的牛人博客，其实很花心思！相关的建站教程很多，我就结合自己的建站过程整理下思路，有兴趣的也可以直接在 Github 上 clone。

###**确定环境和架构**

**1. [GitHub Pages](https://pages.github.com/)**

用 GitHub 托管博客的好处是方便查看和共享、轻量级、干净简单、还可以绑定域名。配合 Jekyll / Hexo / Ghost 等博客系统站内生成网页，直接在本地 repo 更新、发布博客，可以更专注于码字。

首先去官网看一下文档：

- [基础教程](https://help.github.com/categories/github-pages-basics/)主要看一下 Pages 的用法、怎么用 Jekyll 和配置域名

- [进阶版](https://help.github.com/categories/github-pages-features/)了解下 Pages 还有订阅、表情、重定向、Sitemap 等功能和插件

**2. [Jekyll](http://jekyllrb.com/)**

对[博客系统的讨论](http://www.zhihu.com/question/21981094)很多，

- Hexo：基于 Node.js，因为文章生成速度快、操作简单、命令精简而受欢迎；本地直接生成网页上传 Github

- Jekyll：基于 Ruby；Github 原生，上传到 Github 后自动转换成网页；据说 Geek 更喜欢 Jekyll， 所以就选了它

接着去 Jekyll 官网看一下文档：

- 搞清楚[文件目录框架](http://jekyllrb.com/docs/structure/)和[语法元素](http://jekyllrb.com/docs/frontmatter/)的用法

- 了解 Jekyll 还可以安装许多插件

**3. 站点结构**

GitHub 支持[两种形式的 Pages](https://help.github.com/articles/user-organization-and-project-pages/)：

- User & Organization Pages：

  为每个用户准备的主页，通过建立名为 `<username>.github.io` 的 repo 激活，建立后可通过 `http(s)://<username>.github.io` 访问；`username` 是 Github 用户名，每个用户只能建一个，内容在 `master` 分支修改和提交。
- Project Pages：

  为 Github 项目准备的描述页面，博客也可使用这种形式配置模板、发布内容；用户可以建多个 Project Pages，新建 `gh-pages` 分支并在里面修改和提交，可通过 `http(s)://<username>.github.io/<projectname>` 访问。

GitHub 支持两种形式自定义域名：

- Apex domains(一级域名)：在 DNS 提供商也就是 Godady 上修改 A 记录指向 GitHub 的 IP

- Subdomains(二级域名)：在 Godaddy 上申请一个 CNAME 记录

我参考了[Wileam](http://http://www.wileam.com)组织结构，一级域名作为网站导航，博客、project等子项目分别建在二级域名上，方便管理和分类、可以灵活用不同主题，也防止 [Github 不再支持直接指向 IP 地址的 A Host 导致没法查看文章](http://myweb.jowai.info/bind-subdomain-on-godaddy-for-github-pages/)。

架构如下：

<p align="center">
  <img width="420px" height="180px" src="/images/structure.png"/>
</p>

分别建立 repo:

- 一级域名用 `User & Organization Pages`，直接在 `master` 分支 commit

- 二级域名用 `Project Pages`，新建并在`gh-pages`分支 commit

###**绑定域名**

**1. 购买域名**

在 [godaddy](https://www.godaddy.com/) 上挑一个没被注册的域名，付钱搞定。一般第一次付款的折扣力度比较大，可以一下多买几年。

**2. [绑定一级域名](https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/)**

配置 Godaddy：

  1. 配置 A 记录：每个域名只有一个 A 记录，修改 A 记录指向 IP 地址 `192.30.252.153` 或者 `192.30.252.154`

  2. 配置 www 子域名：新建或修改 www 的 CNAME 记录，指向 A 记录，这样有无 www 的两个网站就能相互跳转

配置本地 repo：向根目录添加或修改 CNAME 文件，内容为自己的一级域名 `domain`, 此时浏览器输入一级域名就能实现跳转

**3. [绑定二级域名](http://myweb.jowai.info/bind-subdomain-on-godaddy-for-github-pages/)**

配置 Godaddy Zone File:

  1. 添加一条 Record，选择 CNAME(Alias)

  2. 填写 Host 为二级域名，并指向自己的一级域名

配置本地 repo:

  1. 向根目录添加或修改 CNAME 文件，内容为自己的二级域名 `<subdomain>`

  2. 修改根目录 `_config.yml` 配置文件中的 `url: http://<subdomain>.<domain>`

###**运行起来**

**1. [安装 Jekyll](http://jekyllrb.com/docs/installation/)**

- [安装 Ruby / Rubygems ](https://ruby.taobao.org/)

- `$ gem install jekyll`

**2. 选个喜欢的模板**

- 主页：在 [Free CSS](http://www.free-css.com/free-css-templates) 上找单页模板，下载到一级域名对应 repo 的 `master` 分支

- 博客：在 [Jekyll Themes](http://jekyllthemes.org/) 上直接下载，或者用 GitHub `clone` 到二级域名对应 repo 的 `gh-pages`分支

**3. 预览**

- 本地预览：`$ jekyll serve`，浏览器输入 `http://localhost:4000` 本地预览模板效果

- 提交看效果：`commit`，`push`，页面生效后就能浏览看，看看你的模板在移动端的 Responsive 能力好不好

**4. 发布第一篇博文**

- 在 `_posts` 下新建名为 `YYYY-MM-DD-artical-title` 的 MarkDown 文件

- [MarkDown 语法说明](http://sspai.com/25137)

###**其他自定义及优化**

**1. 修改模板**

- 小修小改：背景图片、字体、主题颜色、添加 favicon

- 添加模块： About、Categories

- 有用的网站：[修改大全](http://blog.javachen.com/2013/08/31/my-jekyll-config.html)； [Google 字体](https://www.google.com/fonts)； [Font Awesome 图标字体库](http://fontawesome.dashgame.com/)；[色码转换器](http://www.ifreesite.com/color/color-code-converter.htm)； [色彩对照表](http://rgb.phpddt.com/)

**2. 增加功能**

- [Disqus 评论模块](http://blog.masr.in/geek/add_comment_to_jekyll.html)：Jekyll 不支持数据库，评论模块需要另装，[Disqus](https://disqus.com/) 和 国内的[多说](http://duoshuo.com/)都不错

- [页内分享](http://codingtips.kanishkkunal.in/share-buttons-jekyll/)：除了自己写链接代码外，分享到国内社交平台还可以用[百度分享](http://share.baidu.com/code/)

- [Google Analytics](https://www.google.com/analytics/)：关心博客访问量和流量分析的话，可以安装 Google Analystics 或者[百度统计](http://tongji.baidu.com/web/welcome/login)

- [代码高亮](http://zyzhang.github.io/blog/2012/08/31/highlight-with-Jekyll-and-Pygments/)：MarkDown 原生的代码块没有高亮，可以用 Pygments 选择喜欢的主题，生成 CSS 样式，引用到自己的模板中

**3. [SEO 网站优化](http://jekyll.tips/tutorials/seo/)**

 SEO（Search Engine Optimize) 是专为 Google 或 百度小爬虫能有效找到博客的优化方法。

- 优化博客标题和描述：用形如 `page.title` 或 `page.description` 这样的标签，而不是 Hard Code，方便修改

- 优化 URL 结构：用 Permalink  改写博客 URL，最好是以标题关键词的形式

- 自定义 404.html：设置 404 错误页面，把失效或者无效的博客地址引导到主页

- 添加 Sitemap.xml： Sitemap 包含站点结构和博客标题、最近修改时间等内容信息，方便搜索引擎找到

- [其他尚未使用的方法](http://vdaubry.github.io/2014/10/21/SEO-for-your-Jekyll-blog/)

最后，感谢我的两个模板 [TPW](https://github.com/mojombo/tpw) 和 [Zetsu](https://github.com/nandomoreirame/zetsu) 让我省了很多力 :D
