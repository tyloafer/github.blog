---
title: Hexo中引入Mermaid流程图
tags: 
    - Hexo
    - Next
    - Mermaid
    - 流程图
categories:
    - Hexo
date: 2018-04-21 15:40
---

# 起因

流程图是个很清晰的展示自己思路的很好的工具，我在我的电脑上用Typora写的时候，自身带了mermaid流程图，但是上传到github上就无法解析了，Google若干后依旧没有效果，但是去Hexo官网逛逛的时候，无意中发现Hexo带有mermaid的插件，所以，对于不了解的技术，还是要多看多读官方文档 [!哀伤脸] 。

下面的内容摘自github并根据我自己的主题 *NEXT* 做了点修改， 下面的内容也是以 *NEXT* 主题为例，各位也可以直接看原版内容[https://github.com/webappdevelp/hexo-filter-mermaid-diagrams](https://github.com/webappdevelp/hexo-filter-mermaid-diagrams)

<!-- more -->

# 安装插件

1. yarn 安装

   ~~~
   yarn add hexo-filter-mermaid-diagrams
   ~~~

2. npm 安装

   ~~~
   npm install hexo-filter-mermaid-diagrams
   ~~~

# 编辑配置文件

配置文件*_config.yml*，应该分别在根目录下和themes/next/下分别有一个，我们这里编辑的是根目录下的配置文件

在 *_config.yml*  的最后加上以下内容

~~~
# mermaid chart
mermaid: ## mermaid url https://github.com/knsv/mermaid
  enable: true  # default true
  version: "7.1.2" # default v7.1.2
  options:  # find more api options from https://github.com/knsv/mermaid/blob/master/src/mermaidAPI.js
    #startOnload: true  // default true
~~~

# 引入相关的js文件

找到主题里面的页脚文件，也即  `themes/next/layout/_partials/footer.swig` ，在文件最后加上以下内容

~~~
{% if (theme.mermaid.enable)  %}
  <script src='https://unpkg.com/mermaid@<%= theme.mermaid.version %>/dist/mermaid.min.js'></script>
  <script>
    if (window.mermaid) {
      mermaid.initialize({theme: 'forest'});
    }
  </script>
{% endif %}
~~~

至此，重启hexo server应该就可以看到效果了。