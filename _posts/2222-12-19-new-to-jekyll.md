---
title:  "参考我做一个blog"
date:   2222-12-19 07:08:11 -0600
categories: blog
---

## 快速预览blog

### —— 使用Jekyll
**我们需要用docker来运行，因为要保持简单的环境**

#### clone项目到本机
[https://github.com/chinafzy/chinafzy.github.io](https://github.com/chinafzy/chinafzy.github.io)

#### 创建jekyll的docker容器
先进入到目录，然后创建docker容器，
```bash
P=4000
P2=$(expr 31729 + $P)
U=${U:-$(basename $PWD)}
JEKYLL_VERSION=3.8
docker run -it \
    --name=blog$P$U \
    -p $P:$P \
    -p $P2:$P2 \
    -v $PWD:/srv/jekyll \
    jekyll/jekyll:$JEKYLL_VERSION \
    /bin/bash -c \
        "bundle; jekyll serve --port $P -w -I -V --livereload --livereload-port $P2 --future"
```
+ P=4000，这个指定了你当前使用的端口，可以修改这个为其它值

第一次运行时候，会需要比较多的时间去下载。

#### 打开浏览器预览
[http://localhost:4000](http://localhost:4000/)

#### jekyll参数说明（可忽略）
[https://jekyllrb.com/docs/configuration/options/](https://jekyllrb.com/docs/configuration/options/)

+ -P, --port：指定服务端口
+ -w, --watch：jekyll会监听文件的修改，立刻更新
+ -I, --incremental：配合上面的参数，jekyll在更新的时候只更新与修改过的文件相关的内容。  
这个参数会带来一个bug：新增页面不能增加到分类和标签下，需要通过命令`jekyll clean && jekyll build`来强制刷新。
+ --livereload：页面会实时监控blog的修改，发生变动时候会自动刷新页面
+ --livereload-port：配合上面的，实时监控的port


### —— 使用Github

#### Fork本项目到你的Github账号下
[https://github.com/chinafzy/chinafzy.github.io](https://github.com/chinafzy/chinafzy.github.io)

![fork my repository](/assets/img/new-blog/fork-git.png)

#### 修改repository的名称为USERNAME.github.io
`USERNAME`是你当前的Github账号名称。
![change it](/assets/img/new-blog/change-repository-name.png)

#### 访问url https://USERNAME.github.io

这个地址也可以在你的git仓库的settings中看见，如下图：

![page url](/assets/img/new-blog/enable-page.png)


## 修改配置
`config.yml`中，修改`Author`部分，使用个人的配置

```yml
# Site Author
author:
  name             : "fzy"
  avatar           : "/assets/img/mny.jpg"
  bio              : "资深宅男，专业程序员."
  location         : "BJ"
  email            : 19781971@qq.com
  links:
    - label: "Email"
      icon: "fas fa-fw fa-envelope-square"
      # url: mailto:chinafzy1978@gmail.com
    - label: "Website"
      icon: "fas fa-fw fa-link"
      # url: "https://your-website.com"
    - label: "Twitter"
      icon: "fab fa-fw fa-twitter-square"
      # url: "https://twitter.com/"
    - label: "Facebook"
      icon: "fab fa-fw fa-facebook-square"
      # url: "https://facebook.com/"
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/chinafzy"
    - label: "Instagram"
      icon: "fab fa-fw fa-instagram"
      # url: "https://instagram.com/"
```
123
