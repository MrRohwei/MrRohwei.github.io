---
layout:     post
title:      "在github上搭建个人博客并在线更新"
date:       2020-09-24 00:00:00
author:     "looveh"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Web
    - 闲谈
    - github
    - 博客

---
该文章是在网上找到的， 保存起来。文章是基于你已经有了仓库了:{username.github.com}
#### 挑选主题

在[这里](http://jekyllthemes.org/)你可以找到很多这种主题。挑一款自己喜欢的吧。

点进去一款主题，可以点击Demo预览。选好之后download即可。

#### 修改主题

我们将下载下来的主题解压。可以见到大概如下文件(不一定完全相同)。

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566639989343.png)

如果你对这些`css`代码和`html`有一定了解，那么你可以展开思维任意魔改了。

如果你是个新手，那么就跟随我来进行简单的修改，将博客变为自己的吧。

打开`config.yml`，将里面的信息修改为自己的。

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566640171263.png)

然后进入==_posts==文件夹，将里面自带的文章删除(==删之前注意查看这些文章的起名格式和里面的书写格式，以后写文章会用到==)。当然你想留着也可以。

对于不同的主题，这里要改的内容不尽相同。以后慢慢修改就行了。

#### 上传主题

将主题传到你github上面的仓库名为==username.github.com==去（外层不要包文件夹，就传你下载解压出来的主题最外层里面的文件和文件夹就行），然后就可以去username.github.com去看自己的博客。

#### 更新博客

首先安利一个makrdown编辑器。

[小书匠markdown编辑器](http://markdown.xiaoshujiang.com/)

介绍一下他的功能。
1.编写markdown并在线预览(以及大多数编辑器能干的)
2.将图片和文章储存在自己绑定的开源仓库中。

## 绑定仓库[#](https://www.cnblogs.com/wxyww/p/xiaoshujiang.html#4062492404)

该编辑器与其他的不同点在于他可以绑定仓库。那么我们如果和刚才自己的博客仓库绑定起来，然后在小书匠上面编辑不就可以在线更新了么。。

首先我们点击左上角的**小书匠**按钮

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566641687182.png)

选择绑定

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566641703482.png)

数据储存选择githubgithub。

然后他告诉我们需要一个takentoken

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566641766253.png)

按照他给的链接去申请，需要的权限在刚才小书匠的的申请页面有写，按要求勾选即可(宽心的我一般都是全选啦)。

然后我们复制出来这个takentoken。(注意这个申请之后只能查看一次，建议找个地方保存好。)填写到小书匠里面对应的takentoken框里。然后那个仓库名称就填你的仓库名称(username.github.iousername.github.io)

然后一路确定。

点击中间回到编辑页面。你会看到左下角多了一栏。

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566642085818.png)

这就是你仓库的文件目录了。

然后我们只要将文章保存到_posts_posts文件夹中就达到了更新博客的目的了。

## 更新[#](https://www.cnblogs.com/wxyww/p/xiaoshujiang.html#334016315)

然后点击左上角新建。保存。

目录一定要选择_posts_posts，名字按照原来自带文章的格式填写。

![enter description here](https://gitee.com/wxyww/picture/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1566642286799.png)

然后写就完了。

快去看看你的博客有没有更新吧

