---
title:  "参考我做一个blog"
date:   2018-12-19 07:08:11 -0600
categories: jekyll
---

## 预览效果
### 使用Jekyll
**我们需要用docker来运行，因为要保持简单的环境**


#### clone项目到本机
[https://github.com/chinafzy/chinafzy.github.io](https://github.com/chinafzy/chinafzy.github.io)

### 创建jekyll的docker容器
```bash
export P=4003
export U=${U:-$(basename $PWD)}
docker run -it \
    --name=blog$P$U \
    -p "$P":"$P" \
    -v $PWD:/srv/jekyll \
    jekyll/jekyll \
    /bin/bash -c \
        "bundle; jekyll serve -P $P -w -V -I"
```
+ P=4003，这个指定了你当前使用的端口，可以修改这个为其它值

#### 打开浏览器预览
[http://localhost:4003](http://localhost:4003/)


### 使用Github

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
      # url: mailto:your.name@email.com
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
