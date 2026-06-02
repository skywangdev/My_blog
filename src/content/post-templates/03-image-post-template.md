---
title: "图片文章模板：带封面和插图的文章"
author: "Sky Wang"
pubDatetime: 2026-06-03T12:00:00Z
featured: false
draft: true
tags:
  - 图片
  - 记录
description: "演示如何在文章里使用本地图片、远程图片和图片说明。"

# 可选：社交分享图。推荐使用 src/assets/images 里的图片。
# ogImage: "@/assets/images/astropaper-og.jpg"
---

这篇模板适合写游记、作品记录、读书摘录、产品截图分析等带图片的文章。

## Table of contents

## 使用 src/assets 里的图片

推荐把图片放在：

```txt
src/assets/images/
```

然后在 Markdown 里这样引用：

```md
![图片描述](@/assets/images/astropaper-og.jpg)
```

实际效果：

![AstroPaper 示例图](@/assets/images/astropaper-og.jpg)

## 使用 public 里的图片

如果图片放在：

```txt
public/example.jpg
```

文章里就这样写：

```md
![图片描述](/example.jpg)
```

`public` 里的图片不会被 Astro 优化，但路径最直观。

## 使用远程图片

远程图片可以直接写 URL：

```md
![远程图片描述](https://example.com/image.jpg)
```

## 带说明文字的图片

如果想加图片说明，可以用 HTML 的 `<figure>`：

<figure>
  <img
    src="https://images.unsplash.com/photo-1518770660439-4636190af475?auto=format&fit=crop&w=1200&q=80"
    alt="电路板特写"
  />
  <figcaption class="text-center">
    示例图片：适合写技术、设备、桌面或项目记录。
  </figcaption>
</figure>

## 写作建议

图片文章最好不要只堆图。你可以按这个节奏写：

1. 这张图是什么。
2. 它为什么值得记录。
3. 我从里面看到了什么。
4. 下一步准备怎么做。
