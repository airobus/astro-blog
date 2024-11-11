---
author: robus
pubDatetime: 2024-11-10T15:22:00Z
modDatetime: 2024-11-10T09:12:47.400Z
title: 在 Astro 中添加 Twikoo 评论系统
slug: "112268"
featured: true
draft: false
tags:
  - astro
  - twikoo
description:
  Add the Twikoo comment system in Astro.
--- 

上干货，在 src/components 组件目录新建一个 Comment.astro 组件，内容如下：

```astro
---
const initTwikoo = `
  twikoo.init({
    envId: '你的Twikoo环境ID',
    el: '#tcomment',
  });
`;
---

<style is:global>
    .tk-avatar {
        align-items: center;
        display: flex;
    }
</style>

<div class="prose max-w-3xl mx-auto px-4">
    <div id="tcomment" class="comment-container"></div>
</div>

<script define:vars={{ initTwikoo }}>
  document.addEventListener('astro:page-load', () => {
    function loadTwikoo() {
      const commentsContainer = document.getElementById('tcomment');
      if (commentsContainer) {
        const script = document.createElement('script');
        script.src = 'https://cdn.staticfile.org/twikoo/1.6.32/twikoo.all.min.js';
        script.async = true;
        script.onload = () => {
          const initScript = document.createElement('script');
          initScript.innerHTML = initTwikoo;
          document.body.appendChild(initScript);
        };
        document.body.appendChild(script);
      }
    }
    loadTwikoo();
  });
</script>
```


接下来在需要添加评论的页面添加组件即可。

比如在 src/pages/posts/[slug]/index.astro 页面中添加：

```astro
---
  import Comment from '~/components/Comment.astro';
---

<!-- 引入评论组件 -->
<Comment />
```