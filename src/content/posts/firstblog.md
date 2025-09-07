---
title: 通过css实现不同主题色切换
published: 2025-09-07
description: '一种css实现不同主题方法的介绍'
image: ''
tags: [Css]
category: '通用'
draft: false 
---

## 基础篇

### CSS变量设置主题

```scss
:root[data-theme='light'] {
  --bg-color: #fff;
  --text-color: #333;
  --primary-color: #007bff;
  --card-bg: #f8f9fa;
}

:root[data-theme='dark'] {
  --bg-color: #1a1a1a;
  --text-color: #fff;
  --primary-color: #4dabf7;
  --card-bg: #2c2c2c;
}
```

### JS切换主题

```js
watch(selectedTheme, (newTheme) => {
  document.documentElement.setAttribute('data-theme', newTheme.toLowerCase());
}, { immediate: true });
```

## 进阶篇

### 动态修改变量

无需多套变量，只需动态修改：

```js
function getVar(name) {
  return getComputedStyle(document.documentElement).getPropertyValue(name);
}
function setVar(name, value) {
  document.documentElement.style.setProperty(name, value);
}
```

这样即可灵活切换和调整主题色