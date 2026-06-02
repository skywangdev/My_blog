---
title: "代码笔记模板：一个小问题的解决过程"
author: "Sky Wang"
pubDatetime: 2026-06-03T12:00:00Z
featured: false
draft: true
tags:
  - 技术
  - 代码笔记
description: "记录一个技术问题的现象、原因、解决方案和最终代码。"
---

这篇模板适合写技术笔记。结构可以很固定：问题是什么、为什么发生、怎么解决、最后验证结果。

## Table of contents

## 问题

我遇到的问题是：

> 这里用一句话描述问题，比如：构建时某个字体文件加载失败。

复现方式：

```bash
pnpm build
```

错误信息可以直接贴关键部分，不要整屏都贴：

```txt
Error: Cannot find module "example"
```

## 原因

这里写你的判断过程。

比如：

- 配置文件里路径写错了。
- 依赖没有安装。
- 某个库只支持 `.ttf`，不支持 `.woff2`。

## 解决方案

代码块可以带语言名：

```ts
const message = "你好，AstroPaper";

function sayHello(name: string) {
  return `${message}，${name}`;
}

console.log(sayHello("Sky"));
```

如果要标注文件名，可以这样写：

```ts file="src/example.ts"
export function formatTitle(title: string) {
  return title.trim();
}
```

如果要展示改动，可以用 diff：

```diff
- const lang = "en";
+ const lang = "zh-cn";
```

AstroPaper 还支持 Shiki 的代码标记。比如高亮某一行：

```ts
const site = {
  title: "我的博客",
  lang: "zh-cn", // [!code highlight]
};
```

## 验证

最后写你怎么确认它成功了：

```bash
pnpm build
```

结果：

```txt
Result: 0 errors, 0 warnings
```

## 小结

这次问题的核心是：这里写一句总结。
