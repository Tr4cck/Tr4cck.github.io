---
title: 「Blog」Hexo-Catus主题折腾心路历程
date: 2022-04-12 13:50:54
tags:
- Frontend
categories:
- TECHNOLOGY
---

## TL;DR

一开始它长成这样：
![](https://s2.loli.net/2022/04/12/v3WCrbG7O2REe4P.png)
这是我在现成配置文件中能做到的最好效果了, 但还是有种让人说不上来的奇怪:

- 首先是首页的文章做不到分类显示, 只有一个 Writing 躺在中间
- 其次是 categories 和 tags 的作用区分不明显, 两者谁用来代替对方似乎都挺合适
- 然后是页脚的右边部分似乎没有任何存在的必要, 它的意义和 header 的意义似乎有一些重叠
- 最后是搜索功能不直接, 不会有任何人想要先点一下鼠标左键, 等一会儿才能跳出一个空荡的搜索页面

## HOWTO

第一个问题实际上是需要在首页上再嵌入一个块, 并且能够进行分类过滤, 也就是能让我们在首页上看到不同的分类下的文章.
第一步是找到首页的位置, 我们根据这个显眼的 **Writing** 来做一些文章.

进入主题目录 `themes/catus/`, 找到 `index.ejs`, 将源码中的遍历方式从 posts 改为 categories:

```ejs
<% site.categories.map(function(category) { %>
  <% if (category.name == 'LIFE') { %>
    <% var field_sort = theme.posts_overview.sort_updated ? 'updated' : 'date' %>
    <% category.posts.sort(field_sort, 'desc').limit(theme.posts_overview.post_count || 5).map(function(post) { %>
      <li class="post-item">
        <%- partial('_partial/post/date', { post: post, class_name: 'meta' }) %>
        <span><%- partial('_partial/post/title', { post: post, index: true, class_name: '' }) %></span>
      </li>
    <% }) %>
  <% } %>
<% }) %>
```

同时我们也严格限制了 categories 的意义和数量, 把它和首页绑定, 并且让 tags 去承担分类的任务, 这样整个网站的分类逻辑就更加清晰了.

以后想加啥分类就再加一坨这个代码, 改一下过滤条件就好了. (似乎有点折磨, 能不能给 hexo 加点功能呢) 但是两个块没办法做到水平排列, 这是怎么会是呢? 直到我查到一个叫做 flex 布局的东西, 它是现代 css 用的比较多的布局方式. 那么我的解决思路是把两个块用一个 section 标签包起来, 并且给这个 section 一个名为 row 的 class, 然后每个小 section 也优化一下 class:

```styl
.row
  --bs-gutter-x: 1.5rem
  --bs-gutter-y: 0
  display: flex
  flex-wrap: wrap
  margin-top: calc(-1 * var(--bs-gutter-y))
  margin-right: calc(-.5 * var(--bs-gutter-x))
  margin-left: calc(-.5 * var(--bs-gutter-x))

.index-writing
  flex: 0 0 auto
  width: 58.33333333%
```

脚注好解决, 找到 footer.ejs, 把里面全部注释掉就好了.
