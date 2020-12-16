---
title: "Hugo Help"
date: 2020-12-16T10:32:27+08:00
draft: true
---

## 搭建 Hugo 博客项目

### 安装 hugo

安装命令：

```bash
brew install hugo
# or
port install hugo
```

查看版本：

```bash
$ hugo version
Hugo Static Site Generator v0.76.0/extended darwin/amd64 BuildDate: unknown
```

### 生成站点

```bash
$ hugo new site blog
Congratulations! Your new Hugo site is created in /Users/tc/Documents/workspace_2020/funnycode-org/blog.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```

### 添加主题

```bash
$ cd blog/
$ git init
Initialized empty Git repository in /Users/tc/Documents/workspace_2020/funnycode-org/blog/.git/
$ git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
Cloning into '/Users/tc/Documents/workspace_2020/funnycode-org/blog/themes/ananke'...
remote: Enumerating objects: 4, done.
remote: Counting objects: 100% (4/4), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 1877 (delta 0), reused 0 (delta 0), pack-reused 1873
Receiving objects: 100% (1877/1877), 4.36 MiB | 1.56 MiB/s, done.
Resolving deltas: 100% (1048/1048), done.
```

配置主题

```bash
echo 'theme = "ananke"' >> config.toml
```

### 增加第一个内容

```bash
$ hugo new posts/hugo-help.md
/Users/tc/Documents/workspace_2020/funnycode-org/blog/content/posts/hugo-help.md created
```

```bash
$ ls content/posts/
hugo-help.md
```

```bash
$ cat content/posts/hugo-help.md 
---
title: "Hugo Help"
date: 2020-12-16T10:32:27+08:00
draft: true
---
```

> draft 表示是否是草稿，如果 true，不会发布。当你写完一篇文章要发布的时候需要设置 ` draft: false`

### 启动 hugo 服务

```bash
$ hugo server -D
Start building sites … 

                   | EN  
-------------------+-----
  Pages            | 10  
  Paginator pages  |  0  
  Non-page files   |  0  
  Static files     |  6  
  Processed images |  0  
  Aliases          |  1  
  Sitemaps         |  1  
  Cleaned          |  0  

Built in 45 ms
Watching for changes in /Users/tc/Documents/workspace_2020/funnycode-org/blog/{archetypes,content,data,layouts,static,themes}
Watching for config changes in /Users/tc/Documents/workspace_2020/funnycode-org/blog/config.toml
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

通过地址 http://localhost:1313 访问

### 改造主题

#### 站点配置

项目路径下的 `config.toml` 文件

```bash
$ cat config.toml 
baseURL = "http://example.org/"
languageCode = "en-us"
title = "My New Hugo Site"
```

如果要挑选其它主题可以 [theme site](https://themes.gohugo.io/)

如果要[自定义主题](https://gohugo.io/hugo-modules/theme-components/)

## 参考文档

[quick-start](https://gohugo.io/getting-started/quick-start/)