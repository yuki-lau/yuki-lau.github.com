---
layout: post
title: 使用Jekyll和GitHub Pages搭建博客
tags: [jekyll, github pages]
---

###摘要：

> 关于如何使用GitHub Pages构建自己的博客或者项目页面，无论是GitHub官方或者网上其他的资料都有很多，但大多面向Linux/Mac OS，所以在此我仅将Windows 7环境下的构建方式以及注意点进行记录。

本科的时候写过很多技术博客，当然内容简单，也是为了受用于自己，三心二意，换过百度空间、校内网（如果是微博的follow方式，估计早就掉粉惨烈了）、博客园。

作为写代码出身的姑娘，一直也想建立自己的博客，有自己的完全统治权。没借口的就是自己懒，或者说怕麻烦，一想到买域名服务器、再学Wordpress就搁置了。前段时间有在我博客园留言的朋友说可以搞个GitHub Pages，于是趁着正式开学前，就“一狠心”折腾了2天把这件事情落实啦！也算是给自己一个新学期好状态的鼓励^_^~
###目录：

* What is GitHub Pages?
* What is Jekyll?
* Use Jekyll to Build GitHub Pages
	* 在本地安装Jekyll环境
	* 在Github建立网站代码库并添加Jekyll模板
	* 提交到Github，等待发布
* Other

###What is GitHub Pages?

<a href="http://pages.github.com/" target="_blank">GitHub Pages</a>是GitHub提供给用户用于构建用户个人、项目网站的一种服务，使用GitHub Pages构建的网站可以免费的放在GitHub的服务器上，这样就省去了我们自己管理服务器的麻烦。而且你的网站代码需要以GitHub公有库（私有可能会收费，我没试）的方式放在GitHub上，这种方式不仅简单，而且保证了你的博客内容也是可以进行版本管理的。

建立GitHub Pages有两种方式：

1. Automatic Page Generator：项目->settings->GitHub Pages->Automatic Page Generator，在编辑框内输入你需要展示的内容，选择模板，发布，即可。

2. 手动创建：这种方式更加灵活，表现也更为丰富，本文介绍这种方法。

###What is Jekyll?

<a href="http://jekyllrb.com/" target="_blank">Jekyll</a>（发音/'dʒiːk əl/，"杰克尔"）是一个静态站点生成器，它会根据网页源码生成静态文件。它提供了模板、变量、插件等功能，所以实际上可以用来编写整个网站。

Github Pages支持Jekyll，因此，你只需要先在本地编写符合Jekyll规范的网站源码，然后上传到github，github不仅可以管理你的网站源代码，并且会通过jekyll对源码进行编译，生成静态的HTML代码放到你的Pages上，至此，其他人就可以通过浏览器访问你的站点了。以后，如果你需要更新自己的站点内容，就只需要在本地修改源码，然后commit、push就可以了。

当然，如果你愿意自己全部手写HTML代码上传到github，也是可以的，但是很麻烦不是么？所以还是使用jekyll吧。

###Use Jekyll to Build GitHub Pages

使用Jekyll来建立Github Pages主要分为以下几步：

0. 在本地安装Git（略）
1. 在本地安装Jekyll环境
2. 在Github建立网站代码库并添加Jekyll模板
3. 提交到Github，等待发布

####1. 在本地安装Jekyll环境

其实在本地安装Jekyll环境并不是必须的，因为提交到Github的是网页源码，并非jekyll编译好的html代码。但是这就好比你作为一个网站开发者，是否需要在本地安装Apache等web服务器一样，首先需要在本地方便的验证结果，再提交，是一个再基本不过的习惯了。

<a href="http://jekyllrb.com/docs/installation/" target="_blank">Jekyll官网</a>上有关于jekyll安装的详细方法，当然仅限于Linux, Unix, Mac OS X。这里我主要介绍Windows下的安装方式。

#####1) 下载并安装<a href="http://rubyinstaller.org/downloads/" target="_blank">RubyInstaller for Windows</a>。
版本可以选择2.0或者1.9.3都可以。

双击安装，选择一个没有空格的路径（推荐C:\Ruby200），选中“Add Ruby executables to your PATH”前的框将ruby添加到环境变量中。

安装完成后，在 Windows 命令行窗口中执行以下命令，检查ruby是否已经加到PATH中： ruby --version

#####2) 安装<a href="http://rubyinstaller.org/downloads/" target="_blank">DevKit</a>。
请根据主页上的描述下载对应的DevKit版本，下载后直接解压到没有空格的路径（例如E:\DevKit)，然后在Windows的命令行窗口中执行以下命令：

<pre>
E: 
cd E:\DevKit 
ruby dk.rb init 
ruby dk.rb install
</pre>
#####3) 安装Jekyll和相关的包。
安装完成Ruby和DevKit 后继续安装jekyll，执行如下命令安装：

<pre>
gem install jekyll
</pre>
等待安装完成即可，这里由于需要下载较多的依赖，因此等待时间稍微有点长。

####2. 在Github建立网站代码库并添加Jekyll模板

#####1) 建立代码库

登录到Github，选择New repository，Project Name命名为：USERNAME.github.com，其中USERNAME就是你的用户名，比如我的用户名是“yuki-lau”，那么我建立的Project Name就是“yuki-lau.github.com”。

#####2) 添加Jekyll模板
我们可以按照Jekyll的目录规则、代码规则从头编写网站源码，当然更方便的是实用已经编写好的Jekyll模板。
关于Jekyll模板有很多资源，比如：

* jekyll-bootsrap：这套模板使用bootstrap来构建前端样式，并且添加了rakefile文件，方便添加博客、页面的方式；
* jekyllthemes.org：这里面有很多jekyll模板，都可以根据说明和开源协议来使用。
* github：在github上搜索jekyll theme，也会有一大堆模板涌现。

我们先不考虑这些模板为我们提供了哪些额外的工具（如jekyll-bootsrap的rakefile），只考虑模板样式和基本页面，我们所需要做的工作就是将模板代码clone到我们本地，再push到我们刚刚新建的USERNAME.github.com库中。这里以jekyll-bootstrap为例：

<pre>
# Clone jekyll-bootsrap 到本地USERNAME.github.com文件夹
git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com 
cd USERNAME.github.com
# 将USERNAME.github.com文件夹的git源更改为我们刚刚建立的代码库，并push jekyll模板代码
git remote set-url https://github.com/USERNAME/USERNAME.github.com.git
git push origin master
</pre>

#####3) 生成网站
cd到USERNAME.github.com文件夹内运行命令：

<pre>
jekyll serve
</pre>
成功后打开浏览器输入地址：http://localhost:4000，即可浏览本地生成的页面。

#####4) 注意

如果在jekyll serve这一步出现诸如编码等错误，则请尝试以下步骤：

1.由于jekyll默认安装1.4.3版本，在windows会出现一些问题，因此更换为1.4.2版本：

<pre>
gem uninstall jekyll
gem install jekyll --version "=1.4.2"
</pre>

2.提示“invalid byte sequence in GBK”，此问题是因字符编码错误引起的，修改方法如下：
找到“C:\Ruby200\lib\ruby\gems\2.0.0\gems\jekyll-1.4.2\lib\jekyll\convertible.rb”
在第38行左右，取消掉原有的两句

<pre>
#self.content = File.read_with_options(File.join(base, name),  
#                                      merged_file_read_opts(opts))  
</pre>
添加一句

<pre>
self.content = File.read_with_options(File.join(base, name),:encoding=>"utf-8")
</pre>

3.如果还是build失败，可以尝试<a href="#" target="_blank">Tang Ru Chen 博客中的方法 2、所有文档使用utf-8无BOM格式</a>

最后，不要忘记修改_config文件中的production_url和BASE_PATH为你自己的github pages地址。

####3. 提交到Github，等待发布
在本地预览网站成功后，就可以将代码push到USERNAME.github.com master了。

push完成后，我们会在项目的settings->GitHub Pages看到如下提示：

<img src="/images/201402/use-jekyll-and-github-pages-to-build-your-blog-site-under-windows-1.jpg" />

如果出现了build failed，则需要确定在本地是否jekyll serve通过，若本地通过了但Github没通过，则请仔细检查编码问题。过几分钟后，再次刷新，若出现如下提示，则表明你的站点发布成功啦！

<img src="/images/201402/use-jekyll-and-github-pages-to-build-your-blog-site-under-windows-2.jpg" />

###Other

如果你使用了jekyll-bootsrap，那么可以参考它的官网继续学习添加post、页面等功能；

如果你使用了其他基本的jekyll模板，那么可以参考jekyll官网学习jekyll目录结构，如何<a href="http://jekyllrb.com/docs/posts/" target="_blank">writing post</a>；

……

至此，在Windows环境下搭建Jekyll环境，并使用GitHub Pages搭建个人博客的方式就全部介绍完了。虽然算是Quick Start这种比较浅的方式，但是可以在后续时候中再满满完善嘛，毕竟现在这个博客是你的了，你拥有完全控制权，加些功能什么的不还是要看你愿不愿意折腾嘛。

