---
title: 在Cloud flare Workers上部署WordPress的完整指南
published: 2025-08-06
Updated: 2025-08-06 1:17:24
description: ''
image: './IMG_20250302_114646.jpg'
tags: [guide]
category: 'WordPress'
draft: false 
lang: ''
---

:::important

本指南目前仅作理论依据参考，缺乏实质性操作，在部分位置上很可能有些许偏差，请见谅

:::

____

## 理解部署方式

Cloudflare Workers 是边缘计算平台，**无法直接运行 WordPress 的 PHP 代码**。但我们可以通过以下两种方式部署：

### 推荐方法：反向代理配置

将 Cloudflare Workers 作为现有 WordPress 站点的反向代理，提供加速和安全功能。

### 替代方法：无头 WordPress

使用 WordPress 作为内容管理后端，通过 Workers 提供 API 驱动的无头架构。

---

## 反向代理方案部署步骤

### 前提条件

1. 已部署的 WordPress 站点（可在任何主机/VPS）

2. Cloudflare 账户

3. 域名已添加到 Cloudflare

### 配置步骤

#### 1. 创建新的 Cloudflare Worker

```javascript
// worker.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // 替换为你的 WordPress 源站 URL
  const originUrl = 'https://your-wordpress-site.com'
  
  // 获取请求 URL
  const url = new URL(request.url)
  
  // 构建目标 URL
  const targetUrl = new URL(url.pathname + url.search, originUrl)
  
  // 复制原始请求头
  const headers = new Headers(request.headers)
  headers.set('Host', targetUrl.hostname)
  
  // 转发请求到 WordPress 源站
  const response = await fetch(targetUrl.toString(), {
    method: request.method,
    headers: headers,
    body: request.body,
    redirect: 'follow'
  })
  
  // 返回响应
  return response
}
```

#### 2. 部署 Worker

1. 登录 Cloudflare 仪表板

2. 进入 "Workers" 部分

3. 创建新服务

4. 粘贴上述代码

5. 保存并部署

#### 3. 配置自定义域名

1. 在 Worker 的 "触发器" 选项卡中

2. 添加路由：`*.yourdomain.com/*`

3. 关联你的域名

#### 4. WordPress 配置调整

在 WordPress 的 `wp-config.php` 中添加：

```php
// 确保 WordPress 识别 Cloudflare 的代理
if ($_SERVER['HTTP_CF_VISITOR']) {
    $cf_visitor = json_decode($_SERVER['HTTP_CF_VISITOR']);
    if ($cf_visitor->scheme == 'https') {
        $_SERVER['HTTPS'] = 'on';
    }
}

// 设置正确的站点 URL
define('WP_HOME', 'https://yourdomain.com');
define('WP_SITEURL', 'https://yourdomain.com');
```

#### 5. 配置缓存规则（可选）

在 Worker 中添加缓存逻辑：

```js
// 在 handleRequest 函数中添加
const cache = caches.default
const cacheKey = new Request(url.toString(), request)

// 检查缓存
let response = await cache.match(cacheKey)

if (!response) {
  // 从源站获取
  response = await fetch(targetUrl.toString(), {
    method: request.method,
    headers: headers,
    body: request.body,
    redirect: 'follow'
  })
  
  // 只缓存 GET 请求和成功响应
  if (request.method === 'GET' && response.status === 200) {
    const cacheResponse = response.clone()
    const cacheTtl = 60 * 60 * 24 // 1天缓存
    cacheResponse.headers.append('Cache-Control', `max-age=${cacheTtl}`)
    event.waitUntil(cache.put(cacheKey, cacheResponse))
  }
}

return response

```

## 无头 WordPress 方案（高级）

### 架构概览

```text
WordPress (后端) → REST API → Cloudflare Worker (前端) → 用户
```

### 配置步骤

1. **设置 WordPress REST API**
   
   - 安装必要的插件（如 JWT Authentication）
   
   - 配置 API 端点权限

2. **创建前端 Worker**

```js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const API_URL = 'https://your-wordpress-site.com/wp-json/wp/v2'
  const { pathname } = new URL(request.url)
  
  // 示例：获取最新文章
  if (pathname === '/posts') {
    const response = await fetch(`${API_URL}/posts?_embed`)
    const posts = await response.json()
    
    // 构建前端页面
    const html = `
      <!DOCTYPE html>
      <html>
      <head><title>无头 WordPress</title></head>
      <body>
        <h1>最新文章</h1>
        <ul>
          ${posts.map(post => `
            <li>
              <h2>${post.title.rendered}</h2>
              <div>${post.excerpt.rendered}</div>
            </li>
          `).join('')}
        </ul>
      </body>
      </html>
    `
    
    return new Response(html, {
      headers: { 'Content-Type': 'text/html' }
    })
  }
  
  // 处理其他路由...
}
```

---

## 最佳实践与注意事项

### 安全配置

```js
// 添加安全头
response = new Response(response.body, response)
response.headers.set('X-Content-Type-Options', 'nosniff')
response.headers.set('X-Frame-Options', 'DENY')
response.headers.set('Content-Security-Policy', "default-src 'self'")
response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
return response
```

### 性能优化

- 使用 Workers KV 存储频繁访问的数据

- 为静态资源添加长期缓存

- 开启 Brotli 压缩

### 限制与注意事项

1. Workers 免费计划限制：
   
   - 每天 100,000 请求
   
   - 每次请求最多 10ms CPU 时间

2. 不支持 PHP 或 MySQL

3. 动态功能（如评论）需通过 AJAX 处理

4. 文件上传仍需通过原始 WordPress 站点

### 替代方案

对于完整的 WordPress 托管，考虑：

- [Cloudflare Pages](https://pages.cloudflare.com/)（静态站点）

- [WordPress.com](https://wordpress.com/)

- 传统 VPS + Cloudflare CDN

---

通过以上配置，您可以在享受 Cloudflare 全球网络性能优势的同时，继续使用 WordPress 的强大功能。反向代理方案是最简单有效的部署方式，而无头架构适合需要高度定制化的高级用例。
