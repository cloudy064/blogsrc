---
title: 云064教你用hexo搭建博客
date: 2016-06-20 12:16:20
categories: blog
tags: [blog, hexo]
---

## 前言
现在搭建一个博客所需要的成本越来越低了，相比起利用`WordPress`,甚至是自己开发博客框架，利用`hexo`+`github`的方式来进行搭建博客省时省力，并且以`Markdown`的方式进行书写，简化了书写时本就不需要费脑考虑的排版之类的问题，完全将博客内容从界面排版中剥离出来。当然这种方式也有不能够随心所欲控制界面的每个元素的限制，不过对于像我这种低端用户来说已经绰绰有余了。之前的`wordpress`搭建的博客，由于不满于它的写作方式，所以整整一年都没有动笔更新过。所以今天就教大家如何利用`hexo`和`github`搭建一个完全免费的博客。这里只介绍`windows`，至于`linux`和`macos`简单很多，这里我就忽略了。
<!--more-->

## 安装node.js
因为`hexo`是跑在`node.js`上的，所以说第一步要做的就是去[node.js官网](https://nodejs.org/en/)下载安装程序。安装完成后，在命令行输入`npm`，如果有输出的话就表示不需要配置了；如果提示找不到这个命令，就需要手动把`nodejs`的安装目录加入到环境变量`PATH`中。

## 安装hexo
在命令行输入`npm install -g hexo-cli`，将`hexo`安装到全局目录中，完成。

## 创建github仓库
如果没有`github`账户先去注册一个，如果你被`Greate Fucking Wall`墙了，那我也没办法了。然后在主页点击`new repository`，然后将名字填写成你的`github名字.github.io`（一定是这个名字），然后点击`Create repository`就行了。如下图所示：
{% asset_img newrepository.png 创建仓库 %}

## 安装并配置git
去[git官网](https://git-scm.com/download)下载`git`工具，所有安装选项都默认就行了。安装完成后，我们需要配置我们的`github`的用户名和密码：
```cmd
git config --global user.name cloudy064
git config --global user.email cloudy@gmail.com
git config --global user.password **********
```

做好了这些之后我们还需要一个`SSH key`，不明白的人自己去百度是干嘛的吧，这里不解释。在命令行继续输入：
```cmd
ssh-keygen -t rsa -C"cloudy064@gmail.com"
```

然后一路回车，之后就可以在用户目录下（我这边是`C:\Users\cloudy`目录）找到`.ssh`文件夹，该文件夹下主要看一个文件`id_rsa.pub`，待会儿会用到。

打开`github`页面，在右上角下拉菜单中点击`Setting`，进入到设置页面，然后点击`SSH and GPG keys`，看到下面这个界面：
{% asset_img generatesshkey.png 生成SSH key %}

你需要点击右上角的`New SSH key`，然后在下面的输入框填写相关信息，`Title`随便填，只是一个标题而已，而`key`则需要把刚刚`id_rsa.pub`中的内容拷贝过来，然后点击`Add SSH key`即可。

到目前为止`git`就配置完成了。

## 准备博客目录
打开你的命令行工具，跳转到你想要存放博客文件的目录下，然后一次输入下面几个命令行：
```cmd
npm install hexo --save
hexo init
```

然后还需要安装`Hexo`插件，让他变得更完善，再依次输入下面的命令行：
```cmd
npm install hexo-generator-index --save
npm install hexo-generator-archive --save
npm install hexo-generator-category --save
npm install hexo-generator-tag --save
npm install hexo-generator-feed@1 --save
npm install hexo-generator-sitemap@1 --save

npm install hexo-server --save

npm install hexo-deployer-git --save
npm install hexo-deployer-heroku --save
npm install hexo-deployer-rsync --save
npm install hexo-deployer-openshift --save

npm install hexo-renderer-marked@0.2 --save
npm install hexo-renderer-stylus@0.2 --save
```

这样一个初步的`hexo`博客就搭建完成了。

## 配置hexo
但是我们的博客发布到哪里呢？打开你的`github`页面，进入到你刚刚创建的仓库中，点击右边的`Clone or download`，然后在下面弹出的对话框中，将输入框中的内容复制下来，等下有用。我这里的内容是`git@github.com:cloudy064/cloudy064.github.io.git`：
{% asset_img getrepositoryaddress.png 找到发布地址 %}

打开博客根目录下的`_config.yml`配置文件，这里面也很多比如标题、作者之类的信息需要你自己去填写。我主要说下发布这里，在最后找到`deploy:`，然后回车，空两个，输入下面的内容：
```
  repository: git@github.com:cloudy064/cloudy064.github.io.git
  type: git
  branch: master
```

这里就是告诉`hexo`，我写完了之后发布到哪个网址。

这样`hexo`也配置完成了。

## 发布博客
打开你的命令行工具，然后跳转到博客的目录下去，依次输入下面的命令行:
```cmd
hexo g
hexo d
```

然后再打开你的浏览器，输入你的博客地址，也就是仓库名称，我的是`cloudy064.github.io`，然后就可以看到`HelloWorld`这个熟悉的文章了。

## 创建博文
打开命令行工具，输入:
```
hexo n "first blog"
```

然后你就可以在`source\_post\`目录下找到`first blog.md`，你所要做的就是用`Markdown`语法创作你的博客，完成之后，按照上面发布博客的流程进行发布就可以了。

## 其他问题

### 添加图片
这里我用的是`hexo`自定义的标签添加图片的。打开`_config.yml`文件，找到`post_asset_folder`选项，把后面的`false`改成`true`。然后以后你用`hexo n`创建博文的时候，在同级目录下会创建一个同名的目录，这个目录里面就是存放图片的，你在博文里面要做的就是利用下面的语句来插入图片：
```
{% asset_img 图片名字 图片描述 %}
```

当然这种方式插入图片有一个弊端就是不好移植，因为是`hexo`专有语法。我用这种方法纯属是因为方便管理，不需要人工去维护图片的存放位置。

### 替换主题
比如我这里用的就是`Next`主题，你需要做的就是用下面的命令把这个主题`clone`到本地：
```cmd
git clone https://github.com/iissnan/hexo-theme-next
```

然后在博客目录下的`themes`文件夹中创建一个新的文件夹，我命名为`Next`，把刚刚`clone`下来的文件中的，除了`.git`和`.github`的文件拷贝到`Next`文件夹中。

然后修改博客根目录下的`_config.yml`，找到`theme`一行，把后面的名字修改为`Next`即可。

然后利用`hexo g`重新生成博客，再用`hexo d`发布上线，就可以看到实际效果了

### 测试博客效果
`hexo`提供了本地服务器的功能，你可以在用`hexo g`生成博客之后，通过输入`hexo s`，然后在浏览器输入`localhost:4000`来访问上线之后的效果。比如像上面替换主题这种大修改，这种本地测试的方式会很有用。