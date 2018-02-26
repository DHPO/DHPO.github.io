---
title: base64翻译工具
date: 2018-02-26 21:35:56
tags:
  - base64
  - javascript
---

起因是这个问题：（已经被续了）
{% asset_img TIM截图20180226201251.png 问题截图 %}

现在把回答补充一下贴在下面：

把[有道划词翻译](https://greasyfork.org/zh-CN/scripts/14197-ac-%E6%9C%89%E9%81%93%E5%8F%96%E8%AF%8D-%E7%BF%BB%E8%AF%91)改成了base64翻译工具：
{% asset_img TIM截图20180226205600.png  %}

注释掉256-273行，并在下面插入：
```javascript
var retContent = {basic: {explains: [decodeURIComponent(escape(atob(word)))]}};
popup(mx, my, JSON.stringify(retContent));
```
**思路**：将`translate`函数替换为base64翻译，并保持原来的格式，从而复用其GUI

效果：
{% asset_img TIM截图20180226205825.png  %}
（考虑到知友的建议，换了一张效果图）
{% asset_img TIM截图20180226210144.png  %}
