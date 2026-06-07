---
title: "关于"
description: "关于 Sky Wang 和这个中文博客。"
---

<style>
  .about-page {
    --about-bg: var(--background);
    --about-fg: var(--foreground);
    --about-accent: var(--accent);
    --about-muted: var(--muted);
    --about-muted-fg: var(--muted-foreground);
    --about-border: var(--border);
    width: min(100vw - 2rem, 980px);
    margin: 0 0 4rem 50%;
    transform: translateX(-50%);
    color: var(--about-fg);
  }

  .app-prose .about-page :where(h2, h3, p, a, strong, span, li) {
    color: inherit;
  }

  .app-prose .about-page a {
    text-decoration: none;
  }

  .about-hero {
    position: relative;
    min-height: 31rem;
    display: grid;
    align-content: end;
    overflow: hidden;
    border: 1px solid var(--about-border);
    border-radius: 8px;
    padding: 3.5rem;
    background:
      linear-gradient(135deg, color-mix(in srgb, var(--about-muted) 72%, transparent), transparent 58%),
      linear-gradient(180deg, var(--about-bg), color-mix(in srgb, var(--about-muted) 48%, var(--about-bg)));
    box-shadow: 0 24px 80px color-mix(in srgb, var(--about-fg) 12%, transparent);
  }

  .about-hero::before {
    content: "";
    position: absolute;
    inset: 2rem;
    border: 1px solid color-mix(in srgb, var(--about-border) 72%, transparent);
    border-radius: 6px;
    pointer-events: none;
  }

  .about-hero::after {
    content: "";
    position: absolute;
    right: -8rem;
    bottom: 4rem;
    width: 32rem;
    height: 32rem;
    border: 1px solid color-mix(in srgb, var(--about-accent) 36%, transparent);
    border-radius: 999px;
    box-shadow:
      inset 0 0 0 5rem color-mix(in srgb, var(--about-accent) 4%, transparent),
      0 0 0 1rem color-mix(in srgb, var(--about-accent) 3%, transparent);
    pointer-events: none;
  }

  .about-hero-content {
    position: relative;
    z-index: 1;
    max-width: 42rem;
  }

  .about-kicker {
    display: inline-flex;
    align-items: center;
    gap: 0.7rem;
    margin: 0 0 1rem;
    color: var(--about-muted-fg);
    font-size: 0.78rem;
    font-weight: 700;
    letter-spacing: 0;
    text-transform: uppercase;
  }

  .about-kicker::before {
    content: "";
    display: block;
    width: 2.5rem;
    height: 2px;
    background: var(--about-accent);
  }

  .app-prose .about-title {
    max-width: 11ch;
    margin: 0;
    color: var(--about-fg);
    font-size: 3.25rem;
    line-height: 0.98;
    letter-spacing: 0;
  }

  .about-lead {
    max-width: 38rem;
    margin: 1.3rem 0 0;
    color: var(--about-muted-fg);
    font-size: 1.05rem;
    line-height: 1.9;
  }

  .about-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    margin-top: 1.8rem;
  }

  .about-button {
    display: inline-flex;
    min-height: 2.75rem;
    align-items: center;
    justify-content: center;
    border: 1px solid var(--about-border);
    border-radius: 6px;
    padding: 0.7rem 1.15rem;
    background: color-mix(in srgb, var(--about-bg) 82%, transparent);
    color: var(--about-fg);
    font-size: 0.92rem;
    font-weight: 700;
    transition:
      transform 160ms ease,
      border-color 160ms ease,
      background 160ms ease;
  }

  .about-button:hover {
    transform: translateY(-1px);
    border-color: var(--about-accent);
    background: color-mix(in srgb, var(--about-muted) 58%, var(--about-bg));
  }

  .about-button.primary {
    border-color: var(--about-accent);
    background: var(--about-accent);
    color: var(--accent-foreground);
  }

  .about-strip {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    margin-top: 0.8rem;
    border: 1px solid var(--about-border);
    border-radius: 8px;
    overflow: hidden;
    background: color-mix(in srgb, var(--about-muted) 44%, var(--about-bg));
  }

  .about-stat {
    min-height: 8rem;
    padding: 1.3rem;
    border-right: 1px solid var(--about-border);
  }

  .about-stat:last-child {
    border-right: 0;
  }

  .about-stat strong {
    display: block;
    font-size: 1.45rem;
    line-height: 1.2;
  }

  .about-stat span {
    display: block;
    margin-top: 0.45rem;
    color: var(--about-muted-fg);
    font-size: 0.88rem;
    line-height: 1.6;
  }

  .about-section {
    margin-top: 0.8rem;
    border: 1px solid var(--about-border);
    border-radius: 8px;
    padding: 2.25rem;
    background: var(--about-bg);
  }

  .about-section-head {
    display: flex;
    align-items: end;
    justify-content: space-between;
    gap: 2rem;
    margin-bottom: 1.35rem;
  }

  .app-prose .about-section h2 {
    margin: 0;
    color: var(--about-fg);
    font-size: 1.55rem;
    line-height: 1.25;
    letter-spacing: 0;
  }

  .about-section-note {
    max-width: 22rem;
    margin: 0;
    color: var(--about-muted-fg);
    font-size: 0.9rem;
    line-height: 1.7;
    text-align: right;
  }

  .about-grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 0.75rem;
  }

  .about-card {
    min-height: 13rem;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    border: 1px solid var(--about-border);
    border-radius: 8px;
    padding: 1.25rem;
    background: color-mix(in srgb, var(--about-muted) 36%, var(--about-bg));
  }

  .about-card.wide {
    grid-column: span 2;
  }

  .about-card-top {
    display: flex;
    align-items: start;
    justify-content: space-between;
    gap: 1rem;
  }

  .about-label {
    color: var(--about-muted-fg);
    font-size: 0.76rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .about-dot {
    width: 0.62rem;
    height: 0.62rem;
    margin-top: 0.25rem;
    border-radius: 999px;
    background: var(--about-accent);
    box-shadow: 0 0 0 0.35rem color-mix(in srgb, var(--about-accent) 14%, transparent);
  }

  .about-card h3 {
    margin: 1rem 0 0;
    color: var(--about-fg);
    font-size: 1.25rem;
    line-height: 1.35;
    letter-spacing: 0;
  }

  .about-card p {
    margin: 0.8rem 0 0;
    color: var(--about-muted-fg);
    font-size: 0.96rem;
    line-height: 1.8;
  }

  .about-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.45rem;
    margin-top: 1.25rem;
  }

  .about-tag {
    border: 1px solid var(--about-border);
    border-radius: 999px;
    padding: 0.22rem 0.58rem;
    color: var(--about-muted-fg);
    font-size: 0.74rem;
    line-height: 1.2;
  }

  .about-list {
    display: grid;
    gap: 0;
    border: 1px solid var(--about-border);
    border-radius: 8px;
    overflow: hidden;
  }

  .about-row {
    display: grid;
    grid-template-columns: 8rem 1fr;
    gap: 1rem;
    padding: 1rem 1.15rem;
    background: color-mix(in srgb, var(--about-muted) 28%, var(--about-bg));
  }

  .about-row + .about-row {
    border-top: 1px solid var(--about-border);
  }

  .about-row span {
    color: var(--about-muted-fg);
    font-size: 0.78rem;
    font-weight: 700;
    text-transform: uppercase;
  }

  .about-row strong {
    color: var(--about-fg);
    font-size: 0.98rem;
    line-height: 1.6;
  }

  @media (max-width: 760px) {
    .about-page {
      width: min(100vw - 1rem, 980px);
    }

    .about-hero {
      min-height: 29rem;
      padding: 2rem;
    }

    .about-hero::before {
      inset: 1rem;
    }

    .app-prose .about-title {
      font-size: 2.4rem;
    }

    .about-strip,
    .about-grid,
    .about-row {
      grid-template-columns: 1fr;
    }

    .about-stat,
    .about-stat:last-child {
      border-right: 0;
      border-bottom: 1px solid var(--about-border);
    }

    .about-stat:last-child {
      border-bottom: 0;
    }

    .about-section {
      padding: 1.5rem;
    }

    .about-section-head {
      display: block;
    }

    .about-section-note {
      margin-top: 0.55rem;
      text-align: left;
    }

    .about-card.wide {
      grid-column: auto;
    }
  }
</style>

<div class="about-page">
<section class="about-hero">
<div class="about-hero-content">
<p class="about-kicker">About Sky Wang</p>
<h2 class="about-title">记录、实践，然后继续向前</h2>
<p class="about-lead">你好，我是 Sky Wang。这个博客是我的长期记录，用来整理 Linux、服务器、一些问题、实践过程，也放一些摄影、阅读和日常思考。写博客对我来说不是展示成果，而是把踩过的坑、验证过的方法和正在形成的判断留下来。</p>
<div class="about-actions">
<a class="about-button primary" href="/posts">阅读文章</a>
<a class="about-button" href="/projects">查看项目</a>
<a class="about-button" href="https://photo.skywangdev.com/" target="_blank" rel="noreferrer">在线相册</a>
</div>
</div>
</section>
<section class="about-strip" aria-label="博客概览">
<div class="about-stat">
<strong>Ops</strong>
<span>Linux、网络、服务器和排障记录。</span>
</div>
<div class="about-stat">
<strong>Tools</strong>
<span>把工具真正接进日常工作流。</span>
</div>
<div class="about-stat">
<strong>Photo</strong>
<span>用照片保存旅途、城市和日常片段。</span>
</div>
</section>
<section class="about-section">
<div class="about-section-head">
<h2>这个博客写什么</h2>
<p class="about-section-note">偏实用，少一点噪音，多一点能复用的经验。</p>
</div>
<div class="about-grid">
<article class="about-card">
<div>
<div class="about-card-top">
<span class="about-label">Systems</span>
<span class="about-dot" aria-hidden="true"></span>
</div>
<h3>把复杂系统拆到能操作</h3>
<p>服务器、网络、驱动、自动化部署这些主题，我更关心真实环境里的判断顺序：先确认什么，怎么缩小范围，最后如何留下可复查的记录。</p>
</div>
<div class="about-tags">
<span class="about-tag">Linux</span>
<span class="about-tag">Network</span>
<span class="about-tag">Server</span>
</div>
</article>
<article class="about-card">
<div>
<div class="about-card-top">
<span class="about-label">Workflow</span>
<span class="about-dot" aria-hidden="true"></span>
</div>
<h3>把工具变成稳定习惯</h3>
<p>AI、脚本、服务和小工具都只是手段。真正值得记录的是它们如何减少重复劳动，如何让一个流程更清晰、更可维护。</p>
</div>
<div class="about-tags">
<span class="about-tag">AI</span>
<span class="about-tag">Automation</span>
<span class="about-tag">Tools</span>
</div>
</article>
<article class="about-card wide">
<div>
<div class="about-card-top">
<span class="about-label">Daily</span>
<span class="about-dot" aria-hidden="true"></span>
</div>
<h3>也保留一点慢下来的空间</h3>
<p>除了技术，我也会记录照片、阅读和生活里的细节。它们不一定高效，但能提醒我：持续维护一个系统，和持续维护自己的注意力，其实是一件事。</p>
</div>
<div class="about-tags">
<span class="about-tag">Photo</span>
<span class="about-tag">Reading</span>
<span class="about-tag">Notes</span>
</div>
</article>
</div>
</section>
<section class="about-section">
<div class="about-section-head">
<h2>本站设计取向</h2>
<p class="about-section-note">保留轻量、快速和中文阅读优先。 By :SkyWang</p>
</div>
<div class="about-list">
<div class="about-row">
<span>Theme</span>
<strong>使用同博客配色，并跟随全站明暗主题变量。</strong>
</div>
<div class="about-row">
<span>Typography</span>
<strong>通过舒适的排版与留白，让技术内容更易于阅读。</strong>
</div>
<div class="about-row">
<span>Structure</span>
<strong>文章、标签、归档、搜索、RSS 和项目页都尽量保持简单直达。</strong>
</div>
</div>
</section>
</div>
