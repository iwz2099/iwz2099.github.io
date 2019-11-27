# Hugo+Github搭建个人博客

#### 介绍

之前博客一直使用hexo搭建,随着用golang越来越多，一直想把博客也迁移到hugo,hugo就不用多说了go语言编写的静态网站生成器,简单、易用、高效、易扩展、快速部署.

我网站目录结构如下:
```
├── archetypes #存放default.md，头文件格式
│   └── default.md  
├── config.toml #整个网站项目全局配置
├── content #content目录存放博客文章（markdown文件）
│   ├── about.md
│   └── posts
├── data #存放数据或配置,可以是json|toml|yaml
├── layouts #存放的是网站的模板文件
├── public #hugo编译后生成的静态文件
│   ├── 404.html
│   ├── CNAME
│   ├── about
│   ├── categories
│   ├── css
│   ├── images
│   ├── index.html
│   ├── index.xml
│   ├── js
│   ├── page
│   ├── posts
│   ├── robots.txt
│   ├── sitemap.xml
│   └── tags
├── resources
│   └── _gen
├── static #存放图片,css,jss静态资源
│   └── images
└── themes #存放网站主题
    └── LoveIt
```
#### 安装hugo
这里以mac环境为例:
```
brew install hugo
hugo new site wanzi
```

更多安装参考[官方安装文档](https://gohugo.io/getting-started/installing/)
#### 主题选择
```
cd themes
git clone https://github.com/dillonzq/LoveIt.git
cd ..
cp themes/LoveIt/exampleSite/config.toml ./
```

修改config.toml信息为你自己博客信息

#### 编写文章
```
hugo new content/posts/git-commands-base.md
```

#### 推送到github

```
cd wanzi/public
git init 
git remote add origin https://github.com/iwz2099/iwz2099.github.io.
echo "wanzi.im" > CNAME
git  add -A
git commit -m "initialization"
git push -u origin master
```
如果使用个人域名,只需要在github仓库下创建CNAME文件写入自己的域名即可，这样访问wanzi.im就可以访问自己博客了。
