---
title: "项目"
description: "一些正在维护和打磨的小项目。"
---

<style>
  .project-page {
    --project-bg: var(--background);
    --project-fg: var(--foreground);
    --project-accent: var(--accent);
    --project-accent-fg: var(--accent-foreground);
    --project-muted: var(--muted);
    --project-muted-fg: var(--muted-foreground);
    --project-border: var(--border);
    width: min(100vw - 2rem, 980px);
    margin: 0 0 4rem 50%;
    transform: translateX(-50%);
    color: var(--project-fg);
  }

  .app-prose .project-page :where(h2, h3, p, a, strong, span, li) {
    color: inherit;
  }

  .app-prose .project-page a {
    text-decoration: none;
  }

  .project-hero {
    position: relative;
    display: grid;
    grid-template-columns: minmax(0, 1.05fr) minmax(18rem, 0.95fr);
    gap: 2.5rem;
    align-items: end;
    min-height: 31rem;
    overflow: hidden;
    border: 1px solid var(--project-border);
    border-radius: 8px;
    padding: 3.5rem;
    background:
      linear-gradient(135deg, color-mix(in srgb, var(--project-muted) 76%, transparent), transparent 58%),
      linear-gradient(180deg, var(--project-bg), color-mix(in srgb, var(--project-muted) 50%, var(--project-bg)));
    box-shadow: 0 24px 80px color-mix(in srgb, var(--project-fg) 12%, transparent);
  }

  .project-hero::before {
    content: "";
    position: absolute;
    inset: 2rem;
    border: 1px solid color-mix(in srgb, var(--project-border) 72%, transparent);
    border-radius: 6px;
    pointer-events: none;
  }

  .project-hero::after {
    content: "";
    position: absolute;
    right: 20%;
    bottom: -11rem;
    width: 34rem;
    height: 34rem;
    border: 1px solid color-mix(in srgb, var(--project-accent) 32%, transparent);
    border-radius: 999px;
    box-shadow:
      inset 0 0 0 5.2rem color-mix(in srgb, var(--project-accent) 4%, transparent),
      0 0 0 1rem color-mix(in srgb, var(--project-accent) 3%, transparent);
    pointer-events: none;
  }

  .project-hero-content,
  .project-preview {
    position: relative;
    z-index: 1;
  }

  .project-kicker {
    display: inline-flex;
    width: fit-content;
    align-items: center;
    gap: 0.7rem;
    margin: 0 0 1rem;
    color: var(--project-muted-fg);
    font-size: 0.78rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .project-kicker::before {
    content: "";
    display: block;
    width: 2.5rem;
    height: 2px;
    background: var(--project-accent);
  }

  .app-prose .project-title {
    max-width: 12ch;
    margin: 0;
    color: var(--project-fg);
    font-size: 3.15rem;
    line-height: 0.98;
    letter-spacing: 0;
  }

  .project-lead {
    max-width: 38rem;
    margin: 1.3rem 0 0;
    color: var(--project-muted-fg);
    font-size: 1.05rem;
    line-height: 1.9;
  }

  .project-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    margin-top: 1.8rem;
  }

  .project-button {
    display: inline-flex;
    min-height: 2.75rem;
    align-items: center;
    justify-content: center;
    border: 1px solid var(--project-border);
    border-radius: 6px;
    padding: 0.7rem 1.15rem;
    background: color-mix(in srgb, var(--project-bg) 84%, transparent);
    color: var(--project-fg);
    font-size: 0.92rem;
    font-weight: 700;
    transition:
      transform 160ms ease,
      border-color 160ms ease,
      background 160ms ease;
  }

  .project-button:hover {
    transform: translateY(-1px);
    border-color: var(--project-accent);
    background: color-mix(in srgb, var(--project-muted) 58%, var(--project-bg));
  }

  .project-button.primary {
    border-color: var(--project-accent);
    background: var(--project-accent);
    color: var(--project-accent-fg);
  }

  .project-preview {
    align-self: stretch;
    display: grid;
    align-content: end;
    min-height: 21rem;
    overflow: hidden;
    border: 1px solid var(--project-border);
    border-radius: 8px;
    padding: 1.25rem;
    background:
      linear-gradient(180deg, transparent 10%, color-mix(in srgb, var(--project-fg) 76%, transparent)),
      url("https://photo.skywangdev.com/home-image") center / cover;
    box-shadow: 0 18px 48px color-mix(in srgb, var(--project-fg) 14%, transparent);
  }

  .project-preview-label {
    width: fit-content;
    border: 1px solid color-mix(in srgb, var(--project-accent-fg) 30%, transparent);
    border-radius: 999px;
    padding: 0.24rem 0.62rem;
    color: var(--project-accent-fg);
    font-size: 0.72rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .project-preview h3 {
    margin: 0.9rem 0 0;
    color: var(--project-accent-fg);
    font-size: 1.45rem;
    line-height: 1.25;
    letter-spacing: 0;
  }

  .project-preview p {
    margin: 0.65rem 0 0;
    color: color-mix(in srgb, var(--project-accent-fg) 78%, transparent);
    font-size: 0.92rem;
    line-height: 1.7;
  }

  .project-strip {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    margin-top: 0.8rem;
    border: 1px solid var(--project-border);
    border-radius: 8px;
    overflow: hidden;
    background: color-mix(in srgb, var(--project-muted) 44%, var(--project-bg));
  }

  .project-stat {
    min-height: 8rem;
    padding: 1.3rem;
    border-right: 1px solid var(--project-border);
  }

  .project-stat:last-child {
    border-right: 0;
  }

  .project-stat strong {
    display: block;
    font-size: 1.45rem;
    line-height: 1.2;
  }

  .project-stat span {
    display: block;
    margin-top: 0.45rem;
    color: var(--project-muted-fg);
    font-size: 0.88rem;
    line-height: 1.6;
  }

  .project-section {
    margin-top: 0.8rem;
    border: 1px solid var(--project-border);
    border-radius: 8px;
    padding: 2.25rem;
    background: var(--project-bg);
  }

  .project-section-head {
    display: flex;
    align-items: end;
    justify-content: space-between;
    gap: 2rem;
    margin-bottom: 1.35rem;
  }

  .app-prose .project-section h2 {
    margin: 0;
    color: var(--project-fg);
    font-size: 1.55rem;
    line-height: 1.25;
    letter-spacing: 0;
  }

  .project-section-note {
    max-width: 22rem;
    margin: 0;
    color: var(--project-muted-fg);
    font-size: 0.9rem;
    line-height: 1.7;
    text-align: right;
  }

  .project-grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 0.75rem;
  }

  .project-card {
    min-height: 13.5rem;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    border: 1px solid var(--project-border);
    border-radius: 8px;
    padding: 1.25rem;
    color: var(--project-fg);
    background: color-mix(in srgb, var(--project-muted) 34%, var(--project-bg));
    transition:
      transform 160ms ease,
      border-color 160ms ease,
      background 160ms ease,
      box-shadow 160ms ease;
  }

  .project-card:hover {
    transform: translateY(-2px);
    border-color: var(--project-accent);
    background: color-mix(in srgb, var(--project-muted) 48%, var(--project-bg));
    box-shadow: 0 14px 36px color-mix(in srgb, var(--project-fg) 10%, transparent);
  }

  .project-card.featured {
    grid-column: span 2;
    min-height: 15rem;
    background:
      linear-gradient(90deg, color-mix(in srgb, var(--project-accent) 10%, transparent), transparent 48%),
      color-mix(in srgb, var(--project-muted) 42%, var(--project-bg));
  }

  .project-card.featured::before {
    content: "";
    width: 3rem;
    height: 2px;
    margin-bottom: 1rem;
    background: var(--project-accent);
  }

  .project-card-top {
    display: flex;
    align-items: start;
    justify-content: space-between;
    gap: 1rem;
  }

  .project-label {
    color: var(--project-muted-fg);
    font-size: 0.76rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .project-status {
    border: 1px solid var(--project-border);
    border-radius: 999px;
    padding: 0.22rem 0.55rem;
    color: var(--project-muted-fg);
    font-size: 0.72rem;
    white-space: nowrap;
  }

  .project-card h3 {
    margin: 1rem 0 0;
    color: var(--project-fg);
    font-size: 1.32rem;
    line-height: 1.3;
    letter-spacing: 0;
  }

  .project-card p {
    margin: 0.75rem 0 0;
    color: var(--project-muted-fg);
    font-size: 0.95rem;
    line-height: 1.75;
  }

  .project-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 0.45rem;
    margin-top: 1.25rem;
  }

  .project-chip {
    border: 1px solid var(--project-border);
    border-radius: 999px;
    padding: 0.22rem 0.58rem;
    color: var(--project-muted-fg);
    font-size: 0.74rem;
    line-height: 1.2;
  }

  .project-timeline {
    display: grid;
    gap: 0;
    border: 1px solid var(--project-border);
    border-radius: 8px;
    overflow: hidden;
  }

  .project-row {
    display: grid;
    grid-template-columns: 8rem 1fr auto;
    gap: 1rem;
    align-items: center;
    padding: 1rem 1.15rem;
    background: color-mix(in srgb, var(--project-muted) 28%, var(--project-bg));
  }

  .project-row + .project-row {
    border-top: 1px solid var(--project-border);
  }

  .project-row span:first-child {
    color: var(--project-muted-fg);
    font-size: 0.78rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .project-row strong {
    color: var(--project-fg);
    font-size: 0.98rem;
    line-height: 1.6;
  }

  .project-row a {
    color: var(--project-muted-fg);
    font-size: 0.85rem;
  }

  .project-row a:hover {
    color: var(--project-accent);
  }

  @media (max-width: 820px) {
    .project-hero {
      grid-template-columns: 1fr;
    }
  }

  @media (max-width: 760px) {
    .project-page {
      width: min(100vw - 1rem, 980px);
    }

    .project-hero {
      min-height: auto;
      padding: 2rem;
    }

    .project-hero::before {
      inset: 1rem;
    }

    .app-prose .project-title {
      font-size: 2.35rem;
    }

    .project-preview {
      min-height: 16rem;
    }

    .project-strip,
    .project-grid,
    .project-row {
      grid-template-columns: 1fr;
    }

    .project-stat,
    .project-stat:last-child {
      border-right: 0;
      border-bottom: 1px solid var(--project-border);
    }

    .project-stat:last-child {
      border-bottom: 0;
    }

    .project-section {
      padding: 1.5rem;
    }

    .project-section-head {
      display: block;
    }

    .project-section-note {
      margin-top: 0.55rem;
      text-align: left;
    }

    .project-card.featured {
      grid-column: auto;
    }
  }
</style>

<div class="project-page">
<section class="project-hero">
<div class="project-hero-content">
<p class="project-kicker">Sky Wang Projects</p>
<h2 class="project-title">把日常工具做得更清爽一点</h2>
<p class="project-lead">这里放一些我正在维护和打磨的项目：有照片站、博客、Serverless 小工具，也有一些围绕效率、内容和自动化的实验。</p>
<div class="project-actions">
<a class="project-button primary" href="https://photo.skywangdev.com/" target="_blank" rel="noreferrer">打开在线相册</a>
<a class="project-button" href="https://github.com/skywangdev" target="_blank" rel="noreferrer">查看 GitHub</a>
</div>
</div>
<a class="project-preview" href="https://photo.skywangdev.com/" target="_blank" rel="noreferrer">
<span class="project-preview-label">Live Project</span>
<h3>旅途与日常</h3>
<p>照片、城市、旅行与完整 EXIF 记录。</p>
</a>
</section>
<section class="project-strip" aria-label="项目概览">
<div class="project-stat">
<strong>Photo</strong>
<span>旅行、城市、日常与 EXIF 记录。</span>
</div>
<div class="project-stat">
<strong>Astro</strong>
<span>当前博客基于 AstroPaper 修改。</span>
</div>
<div class="project-stat">
<strong>Serverless</strong>
<span>轻量工具优先，部署和维护成本更低。</span>
</div>
</section>
<section class="project-section">
<div class="project-section-head">
<h2>Featured</h2>
<p class="project-section-note">主项目用更大的展示位，让访问者第一眼知道你在认真维护什么。</p>
</div>
<div class="project-grid">
<a class="project-card featured" href="https://photo.skywangdev.com/" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">Online Gallery</span>
<span class="project-status">Live</span>
</div>
<h3>旅途与日常</h3>
<p>用照片记录城市、旅行与日常片段，并保留完整 EXIF 信息，让每一个画面都能被重新找到。</p>
</div>
<div class="project-meta">
<span class="project-chip">Photo</span>
<span class="project-chip">EXIF</span>
<span class="project-chip">Canon / Sony / iPhone</span>
</div>
</a>
<a class="project-card" href="https://github.com/skywangdev/My_blog" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">Blog</span>
<span class="project-status">Astro</span>
</div>
<h3>My_blog</h3>
<p>这个中文博客本身也是一个长期项目：围绕写作、归档、搜索、中文阅读体验和轻量维护持续调整。</p>
</div>
<div class="project-meta">
<span class="project-chip">Astro</span>
<span class="project-chip">AstroPaper</span>
</div>
</a>
<a class="project-card" href="https://github.com/skywangdev/serverless-qrcode-hub" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">Utility</span>
<span class="project-status">JavaScript</span>
</div>
<h3>serverless-qrcode-hub</h3>
<p>用于生成长期可用的二维码，也可作为短链接服务使用，适合解决群聊二维码频繁变动的问题。</p>
</div>
<div class="project-meta">
<span class="project-chip">Serverless</span>
<span class="project-chip">QR Code</span>
<span class="project-chip">Short Link</span>
</div>
</a>
</div>
</section>
<section class="project-section">
<div class="project-section-head">
<h2>GitHub Lab</h2>
<p class="project-section-note">更偏实验和工具的小项目，保留简洁卡片，方便后续继续替换或扩展。</p>
</div>
<div class="project-grid">
<a class="project-card" href="https://github.com/skywangdev/avatar-gallery" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">Gallery</span>
<span class="project-status">TypeScript</span>
</div>
<h3>avatar-gallery</h3>
<p>头像和图片展示相关的小项目，适合继续扩展成个人视觉资产库。</p>
</div>
<div class="project-meta">
<span class="project-chip">TypeScript</span>
<span class="project-chip">Gallery</span>
</div>
</a>
<a class="project-card" href="https://github.com/skywangdev/bark-worker" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">Notification</span>
<span class="project-status">JavaScript</span>
</div>
<h3>bark-worker</h3>
<p>围绕 Bark 推送和 Worker 部署的轻量服务，偏向简单、直接、可长期运行的小工具。</p>
</div>
<div class="project-meta">
<span class="project-chip">Worker</span>
<span class="project-chip">Bark</span>
</div>
</a>
<a class="project-card" href="https://github.com/skywangdev/serverless-markdown-convertor" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">Convertor</span>
<span class="project-status">HTML</span>
</div>
<h3>serverless-markdown-convertor</h3>
<p>Markdown Conversion 工具，适合处理内容格式转换和轻量文本工作流。</p>
</div>
<div class="project-meta">
<span class="project-chip">Markdown</span>
<span class="project-chip">Serverless</span>
</div>
</a>
<a class="project-card" href="https://github.com/skywangdev/fuclaude-switcher" target="_blank" rel="noreferrer">
<div>
<div class="project-card-top">
<span class="project-label">AI Tool</span>
<span class="project-status">JavaScript</span>
</div>
<h3>fuclaude-switcher</h3>
<p>简洁优雅的 FuClaude 工具，属于个人 AI 工作流里的效率型实验。</p>
</div>
<div class="project-meta">
<span class="project-chip">AI</span>
<span class="project-chip">Tooling</span>
</div>
</a>
</div>
</section>
<section class="project-section">
<div class="project-section-head">
<h2>Direction</h2>
<p class="project-section-note">后续可以继续补充截图、技术栈、部署方式和复盘文章。</p>
</div>
<div class="project-timeline">
<div class="project-row">
<span>Now</span>
<strong>先把可访问的项目集中展示出来</strong>
<a href="https://github.com/skywangdev" target="_blank" rel="noreferrer">GitHub</a>
</div>
<div class="project-row">
<span>Next</span>
<strong>为重点项目补充截图、技术栈和架构说明</strong>
<a href="https://photo.skywangdev.com/" target="_blank" rel="noreferrer">Photo</a>
</div>
<div class="project-row">
<span>Later</span>
<strong>把成熟项目写成文章，沉淀成长期维护记录</strong>
<a href="/posts">Posts</a>
</div>
</div>
</section>
</div>
