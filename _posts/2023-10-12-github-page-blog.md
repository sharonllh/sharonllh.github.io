---
layout: post
title: 用Github Page和Jekyll搭建个人博客
date: 2023-10-12 +0800
tags: [Github Page, Jekyll, 博客]
categories: [GitHub]
---

如果你想要一个方便维护、无广告、支持markdown书写的个人博客，用Github Page和Jekyll来搭建是不错的选择。[Github Page](https://docs.github.com/en/pages/quickstart)是通过Github托管的网页，可以用来创建个人博客或开源项目的展示页面。[Jekyll](https://jekyllrb.com/)是一个静态站点生成器，可以很方便地根据html和markdown文件组织成一个美观的网站。本文介绍一种快速简单的搭建方法。

1. 在[Jekyll主题网站](http://jekyllthemes.org/themes/mundana-jekyll-theme/)或者Github上找一个合眼缘的博客项目，然后fork一下。
    > 在Github上搜索关键词`github blog`，查看那些repository名为`xxx.github.io`的，打开`https://xxx.github.io`就能看到对应的博客。

2. fork的时候repository name必须是`{你的Github用户名}.github.io`，并且这个repository必须是public的。

3. 到fork完成后的repository下，选择`Settings -> Code and automation -> Pages`，然后在`Build and deployment`下设置`Source`为`Deploy from a branch`，设置`Branch`为`main`。
![BuildAndDeployment](/assets/img/GithubPageBlog_BuildAndDeployment.png)

4. 等待一会儿，页面上会出现`Your site is live at https://{你的Github用户名}.github.io`的提示，表示你的博客已经部署好了，地址就是`https://{你的Github用户名}.github.io`。
![SiteReady](/assets/img/GithubPageBlog_SiteReady.png)
    >如果一直没出现这个提示的话，尝试随便push一个change到main branch。

怎么样，是不是很简单呢？当然啦，这个博客不一定完全符合你的要求，也许你想要进行一些修改和定制，接着往下看吧。

{:start="5"}
5. 参考[Jekyll安装文档]((https://jekyllrb.com/docs/installation/))，在本地安装Ruby和Jekyll。

    - 下载并安装[Ruby+Devkit](https://rubyinstaller.org/downloads/)，安装时使用默认选项。
      >安装路径不能包含空格。

    - 在自动打开的cmd窗口中输入3。
      ![InstallRuby](/assets/img/GithubPageBlog_InstallRuby.png)
      >如果没有自动打开cmd，就手动打开一个并输入`ridk install`。

    - 运行`gem install jekyll`，安装Jekyll。
      >如果安装速度很慢的话，可以考虑替换gem的source。先删除原来的source: `gem source -r https://rubygems.org/`, 再添加新的source：`gem source -a http://gems.ruby-china.com`。

6. 把`{你的Github用户名}.github.io`这个respository clone到本地，在项目路径下打开一个cmd，并运行`jekyll server`，运行成功后打开`http://localhost:4000`就能看到本地运行的博客了。
    >运行`jekyll server`时也许会报一些缺少plugin的错误，运行`gem install xxx`安装就行。

7. 接下来就随意修改，让博客变成你满意的样子吧。这里我就不多赘述了，相信聪明的你可以自己搞定的！

所有的文章都在`_posts`目录下，文件名是有格式要求的，必须是`YYYY-MM-DD-TITLE.md`。文章的开头需要添加front matter，如下所示，更多内容可以参考[Jekyll Posts](https://jekyllrb.com/docs/posts/)。

```
---
layout: post
title: 用Github Page和Jekyll搭建个人博客
date: 2023-10-12 +0800
tags: [Github Page, Jekyll, 博客]
categories: [GitHub]
---
```

每次新写一篇文章后，只需要把这个change push到main branch上，Github那边就会触发一次部署，你的博客就能自动更新了。Enjoy blogging!