---
title: 部署 Hexo 博客到 Github Pages
date: 2020-04-11 03:30:48
tags:
---

Hexo 官方的文档地址：https://hexo.io/docs/github-pages.html 这个文档有一堆问题，评论区有说。

建议使用 Travis CI 来部署。

首先创建一个自己用户名的 repo `dimpurr.github.io` ，随意要不要 Initialize ，然后把 Hexo 的文件丢进去。 ~~`.gitignore` 可以选 Node 的，记得从里面去掉 `public` 一行的注释。~~ Hexo 自带 `.gitignore` 所以不用选。

如果和我一样还没有安装 Hexo 的话， clone repo 然后去新建 hexo 分支：

```bash
yarn global add hexo # npm install -g hexo-cli
cd dimpurr.github.io
git checkout -b hexo
hexo init tmp
mv ./tmp/* .
rm -rf tmp # rmdir tmp on windows
```

然后编辑 `_config.yml` 做一些基本设置。我觉得有必要改的：

```yml
language: zh
timezone: 'Shanghai'
```

记得 push 。

然后在 Github 账户上添加 Travis CI ： https://github.com/marketplace/travis-ci 选开源 Free 就好。我选了 `Only select repositories` ，然后添加这个项目。

创建新的 Github Token ：https://github.com/settings/tokens/new 理论上只需要 **repo** 部分权限。存好这个 Token ，关掉网页就再也看不到了。

在 Travis CI 控制面板编辑对应 repo 的环境变量，例如 https://travis-ci.com/github/dimpurr/dimpurr.github.io/settings ，把刚才的 Token 存进 `GH_TOKEN` 变量。

在 repo 里新建 `.travis.yml` 文件，内容如下：

```yml
sudo: false
language: node_js
node_js:
  - 10 # use nodejs v10 LTS
cache: npm
branches:
  only:
    - hexo # store source code of hexo in hexo-source branch
script:
  - hexo generate # generate static files
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  # added
  target_branch: master # generate static files to master
  on:
    branch: hexo
  local-dir: public
```

此时就已经可以访问了。该换主题换主题，该加插件加插件，然后开始写文章吧！

~~吐槽一个，安装了主题之后 yarn fetch 慢死了， Travis CI 又要每次重新 fecth ，还不如手动 hexo generate 呢 ……~~

### 绑定域名

在 Github 的 repo setting 里面设置自定义域名之后会在 master 生成一个 `CNAME` 文件，内容是你绑定的域名，显然这个是会被 Travis CI 覆盖掉的。

所以在 `hexo` 分支的 `source` 目录下手动创建这个文件，内容是一行你要绑定的域名。

然后到 DNS 配置中添加对应记录。如果是顶级域名的话：

```
@          A             192.30.252.153
@          A             192.30.252.154
www      CNAME           username.github.io.
```

如果是 HTTPS 的话， A 记录指向：

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

不是顶级域名不需要 A 记录。

### 添加 RSS 插件

https://github.com/hexojs/hexo-generator-feed

```bash
yarn add hexo-generator-feed --save
```

可用的配置：

```yml
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
  content:
  content_limit: 140
  content_limit_delim: ' '
  order_by: -date
  icon: icon.png
  autodiscovery: true
  template:
```