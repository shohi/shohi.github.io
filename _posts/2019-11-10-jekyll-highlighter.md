---
layout: post
title: jekyll中使用highlighter
category: tech
guid: 8AFED111-B103-48FA-9645-136A7B074213
tags: [jekyll]

---
{% include JB/setup %}

`Jekyll`从版本`3`开始将`rouge`做为默认的highlighter. `rouge`是一个纯ruby写的lighlighter, 本文将简单介绍如何在jekyll中使用`rouge`和`GFM`(github-flavored markdown), 以及对`github pages`进行本地调试. 文中内容主要来自[链接1](#add-syntax-hl).

### 1. `_config.yaml`
将个人的`github pages` clone到本地，并找到目录下的 `_config.yaml`. 将markdown的解析引擎设为`kramdown`同时指定其处理的markdown格式为`GFM`, 另外将highlighter设为`rouge`. 

```yaml
# _config.yaml
markdown: kramdown
highlighter: rouge
kramdown:
  input: GFM
```

这样设置之后, `jekyll`就能正确将`GFM`风格的`markdown`转成`hmtl`了，并对代码相应的代码块采用`rouge`的高亮处理(即对转换后`html`的特定元素添加高亮`class`).

但是为了让高亮产生效果，必须在默认模板中添加`rouge`导出的`css`(<span style="color:red">**这一点至关重要**</span>).

```bash
# create a css file for the highlighting style
rougify help style
rougify style github > /path/to/css/hl.rouge.github.css

# add above css file in `default.html` 
# <link rel="stylesheet" href="{{ ASSET_PATH }}/css/hl.rouge.github.css">
```

### 2. 本地调试
本地调试jekyll blog, 需要安装以下gem.

```bash
# github-pages contains jekyll related gems
gem install github-pages

gem install rouge
```

然后在blog根目录,运行

```
$> jekyll serve

   Server address: http://127.0.0.1:4000
   Server running... press ctrl-c to stop.
```

成功启动之后, 访问<a href="http://127.0.0.1:4000" target="_blank">`http://127.0.0.1:4000`</a>即可实时预览blog.

### 3. Tips

- 生成UUID - `uuidgen`
- 参看github pages所支持的jekyll版本 - [`https://pages.github.com/versions/`](https://pages.github.com/versions/)
    
### 4. 参考链接

1. <span id="add-syntax-hl">[Add Syntax Highlighting to your Jekyll site with Rouge](https://bnhr.xyz/2017/03/25/add-syntax-highlighting-to-your-jekyll-site-with-rouge.html)</span>
2. {: id="jekyll-now"}[Jekyll Now](https://github.com/barryclark/jekyll-now)
3. {: id="GFM"}[GFM - GitHub Flavored Markdown Spec](https://github.github.com/gfm/)

### 5. 备注

- 以上配置在`jekyll 3.8.5`上测试通过.
