---
title: 'Hugo、twikoo、umami博客搭建'
date: 2024-05-05T23:22:47+08:00
tags: ["Hugo","twikoo","umami","blog","博客"]
---

## Hugo

> [Hugo官网](https://gohugo.io)

```bash
# 安装
brew install hugo
# 创建项目
hugo new site blog --format=yaml
```

创建git仓库

```
cd blog
git init
```

PaperMod主题

```
git submodule add https://github.com/adityatelange/hugo-PaperMod/ themes/PaperMod
```

修改配置文件`hugo.yaml`

```yaml
theme: PaperMod
```

启动测试

```bash
hugo server
```

## PaperMod主题

> [Github仓库](https://github.com/adityatelange/hugo-PaperMod/)

这个主题有许多可配置的选项，具体参考`hugo.yaml`配置文件或者[主题Wiki](https://github.com/adityatelange/hugo-PaperMod/wiki)

```yaml
baseURL: "https://example.com"
title: PaperMod
deFaultContentLanguage: zh
languageCode: zh
paginate: 5

theme: PaperMod


enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false
enableEmoji: true

minify:
  disableXML: true
  minifyOutput: true

taxonomies:
  category: categories
  tag: tags

params:
  env: production # to enable google analytics, opengraph, twitter-cards and schema.
  title: PaperMod
  description: "ExampleSite description"
  keywords: [PaperMod]
  author: Raison Chan
  # author: ["Me", "You"] # multiple authors
  images: ["<link or path of image for opengraph, twitter-cards>"]
  DateFormat: "January 2, 2006"
  defaultTheme: auto # dark, light
  disableThemeToggle: false

  ShowReadingTime: true
  ShowShareButtons: false
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowWordCount: true
  ShowRssButtonInSectionTermList: true
  UseHugoToc: true
  disableSpecial1stPost: false
  disableScrollToTop: false
  comments: true
  hidemeta: false
  hideSummary: false
  showtoc: true
  tocopen: false

  assets:
    # disableHLJS: true # to disable highlight.js
    # disableFingerprinting: true
    favicon: "<link / abs url>"
    favicon16x16: "<link / abs url>"
    favicon32x32: "<link / abs url>"
    apple_touch_icon: "<link / abs url>"
    safari_pinned_tab: "<link / abs url>"

  label:
    text: "無別"
    icon: /apple-touch-icon.png
    iconHeight: 35

  # profile-mode
  profileMode:
    enabled: false # needs to be explicitly set
    # title: ExampleSite
    # subtitle: "This is subtitle"
    # imageUrl: "<img location>"
    # imageWidth: 120
    # imageHeight: 120
    # imageTitle: my image
    # buttons:
    #   - name: Posts
    #     url: posts
    #   - name: Tags
    #     url: tags

  # home-info mode
  homeInfoParams:
    Title: "Hi there \U0001F44B"
    Content: Welcome to my blog

  socialIcons:
    - name: KoFi
      title: Buy me a Ko-Fi :)
      url: "#"

  cover:
    hidden: true # hide everywhere but not in structured data
    hiddenInList: true # hide on list pages and home
    hiddenInSingle: true # hide on single page

  # editPost:
  #   URL: "https://github.com/<path_to_repo>/content"
  #   Text: "编辑" # edit text
  #   appendFilePath: false # to append file path to Edit link

  # for search
  # https://fusejs.io/api/options.html
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    limit: 10 # refer: https://www.fusejs.io/api/methods.html#search
    keys: ["title", "permalink", "summary", "content"]
menu:
  main:
    - identifier: posts
      name: 文章
      url: posts
      weight: 1
    - identifier: archives
      name: 归档
      url: archives
      weight: 5
    - identifier: categories
      name: 分类
      url: /categories/
      weight: 10
    - identifier: tags
      name: 标签
      url: /tags/
      weight: 20
    # - identifier: search
    #   name: 搜索
    #   url: /search
    #   weight: 30
    - identifier: friendlink
      name: 友链
      url: /friend-links
      weight: 30
    - identifier: about
      name: Me
      url: /about
      weight: 40
# Read: https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs#using-hugos-syntax-highlighter-chroma
pygmentsUseClasses: true
markup:
  highlight:
    noClasses: false
    # anchorLineNos: true
    # codeFences: true
    # guessSyntax: true
    # lineNos: true
    # style: monokai
```

### 分类页面

```
cd content
mkdir categories
```

修改分类标题

```
cd categories
touch _index.md
```

```markdown
---
title: "分类"
---
```

### 归档页面

创建`content/archives.md`

```markdown
---
title: "归档"
layout: "archives"
# url: "/archives"
summary: "archives"
ShowRssButtonInSectionTermList: false
---
```

## Twikoo评论插件

> [twikoo官网](https://twikoo.js.org)

有多种部署方案，本人采用docker

`docker-compose.yml`

```yaml
version: '3'
services:
  twikoo:
    image: imaegoo/twikoo
    container_name: twikoo
    restart: unless-stopped
    ports:
      - 3001:8080
    environment:
      TWIKOO_THROTTLE: 1000
    volumes:
      - ./data:/app/data
```

采用cloudflare的内网穿透暴露到公网，并绑定域名

### 添加评论插件

创建`layouts/partials/comments.html`

```html
<!-- Twikoo -->
<div>
    <details class="comments_details">
        <summary style="cursor: pointer; margin: 50px 0 20px 0;width: 130px;">
            <span style="font-size: 20px;color: var(--content);">评论</span>
        </summary>
        <div id="tcomment"></div>
    </details>
    <script src="https://cdn.staticfile.org/twikoo/{{ .Site.Params.twikoo.version }}/twikoo.all.min.js">
    </script>
    <script>
        twikoo.init({
            envId: {{ .Site.Params.twikoo.id }},
        el: "#tcomment",
            lang: 'zh-CN',
            region: {{ .Site.Params.twikoo.region }},
        path: window.TWIKOO_MAGIC_PATH||window.location.pathname,
        })
    </script>
</div>
```

修改`hugo.yaml`配置文件

```yaml
params:
  comments: true
  twikoo:
    version: 1.6.32
    id: "地址"
```

## Umami

> [umami官网](https://umami.is)

docker部署

```
git clone https://github.com/umami-software/umami
docker compose up -d
```

### 添加分析脚本

创建`layouts/partials/extend_footer.html`

```html
<!--添加umami脚本-->
```

