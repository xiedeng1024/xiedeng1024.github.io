---
title: GitHub Pages
date: 2021-10-04 22:30:00
categories: GitHub-Pages
description: 快速维护个人站点
tags:
- github-pages
---

# 站点域名

  参考：[GitHub Pages 使用入门](https://docs.github.com/cn/pages/getting-started-with-github-pages)

```
用户站点：http(s)://<username>.github.io
项目站点：http(s)://<username>.github.io/<repository>
个人用户不考虑组织站点
```

站点域名取决于用户名   确定用户名 如不合适请调整



# 站点仓库

* 创建站点仓库 属性public 确保全网可浏览

  ```
  仓库名为<username>.github.io 则属于用户站点 为站点主仓库 访问http(s)://<username>.github.io
  其他仓库名 则属于项目站点 访问http(s)://<username>.github.io/<repository>
  ```

* 仓库根目录中，创建一个名为 `index.md` 激活pages 配置页

  ```仓库名称下，单击 **Settings（设置）**，在左侧边栏中，单击 **Pages（页面）**```



# 使用模板创建站点

* 搜索满足自己需求的模板 fork 即可

  ​    [jekyll-jacman](https://github.com/Simpleyyt/jekyll-jacman.git)

* fork 到自己仓库后 调整仓库名称 为需要的用户站点 或 项目站点 名称

* 调整**_config.yml** 为符合自己需求

* 新增文章 _posts 目录存放所有文章 参考文件名 创建即可

  文章前缀

  ```
  ---
  title: GitHub Pages
  date: 2021-10-04 22:30:00
  categories: GitHub-Pages
  description: 快速维护个人站点
  tags:
  - github-pages
  ---
  ```

* 访问站点 ```http(s)://<username>.github.io```

