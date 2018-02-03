---
title: '常见配置文件语言: INI, XML, JSON与YAML'
date: 2018-02-03 16:27:41
tags: 
  - YAML
  - JSON
---

昨天搭建了Hexo博客，发现无论是Hexo本身，还是Next主题，配置文件使用的都是YAML语言。

为了更好地调教我的博客，也为了提高自己的知识水平，我决定学习一个YAML的基础语法，并和我之前遇到过的XML、JSON、INI等语言一起做个总结。

# 前言

配置文件的主要作用是给程序提供运行时的参数。理想的配置文件语言应当满足：

- 便于人类阅读理解，最好让非专业人士能猜出大概的语法
- 具有强大的表达能力，除了键值对外还能支持数组、引用等表达方式；能表达层级关系
- 便于书写，包括纯手打和使用IDE
- 存储空间小
- ...

上述性质往往难以同时满足，例如：语法简单的语言表达能力就不会很强，而抽象符号较多的语言在节约空间和提升书写效率的同时也更难以学习。对这些性质的取舍一定程度上决定了语言的使用场景。

# INI

INI是这些语言中语法最简单的，也是我最早接触的。著名梯子GoAgent就是使用INI文件进行配置的，配置文件部分摘录如下：

```ini
[listen]
ip = 127.0.0.1
port = 8087
username =
password =
visible = 0
debuginfo = 0
; to be continued
```

## key-value

INI的键值对语法为：`key = value`。

需要注意的是key在section内是通常是不能重复的，其行为是不确定的，可能会导致前者被覆盖，或者直接报错；但key在不同的section中是可以重复的。

## section

INI使用section进行简单分割。section名写在方括号中，例如：

```ini
[ServerA]
enable = 0
port = 8080

[ServerB]
enable = 1
port = 8081
```

可以看到section可以在一定程度上起到数组的作用

## 注释

INI的单行注释为

```ini
; comment
```

## 总结

作为配置语言，INI的最大优势是简洁易懂，即使是完全没接触过编程的人也能懂个大概；但随之而来的问题是语言表达能力的不足，无法表现数组、层级关系等。因此INI适合用在配置内容简单、面向大众的场景中。

# XML

我在用java写后端时首次用到了XML来配置SpringMVC和Maven，而XML还是一种数据交换格式，常见于RSS等。我对XML的第一印象是笨重，它大概长这个样子：

```xml
<mail-list>
  <mail important=false>
    <title>Mail 1</title>
    <author>Jimmy</author>
    <content>Hello, world</content>
  </mail>
  <!-- comment -->
  <mail important=true>
    <title>Mail 2</title>
    <author>Xkz</author>
    <content>foobar</content>
  </mail>
</mail-list>
```

## 基本语法

XML的key-value形式是`<key>value</key>`,在XML中通常将`<key>`称为tag，每个tag都有对应的tag将其闭合。tag之间可以嵌套。在上面的示例中，一个mail-list的value为两封mail，而每个mail又包括title、author、content等内容。这样的层层嵌套形成了一种树状结构。在`value`中的内容通常称为data。

tag也可以带有属性。在上面的示例中，mail带有一个important的属性，来表示其是否是重要邮件。在tag中的属性通常称为metadata，也就是“数据的数据”。

XML的注释需要写在`<!--`和`-->`中，可以是多行的。

## data vs metadata

一个属性写在data中还是metadata中没有统一的标准。例如上述mail中也可以把mail的author作为metadata：

```xml
<mail author=jimmy important=false>
  <title>Mail 1</title>
  <content>Hello, world</content>
</mail>
```

一般而言，data为基础的数据，而metadata为附加的数据。对于模棱两可的情况，你们自身也要判断。

## 总结

和INI相比，XML最大的特点是拥有层级结构，可以表达复杂的层级关系。

但XML的一个明显缺陷是key的重复性：每个key都会出现两次。这种设计不仅不能使其含义更加直观，反而会使非专业人士困扰；不仅如此，key的重复还使得书写需要依赖IDE提高效率；此外，作为数据交换语言其冗余也太大了。

因此，XML适合表达复杂的层级关系，例如文件树；而在一般情形下不适合使用，其数据交换的功能也逐渐被JSON替代。

# JSON

JSON是我在写前端的时候遇到的，如果有JavaScript基础的话JSON是十分易懂的，可以看作JavaScript的子集(JS-SON，不过这不是官方解释)。除了作为配置文件，JSON还是最常用的数据交换格式。JSON大概长这个样子：

```json
{
  "mailList":[
    {
      "title": "Mail 1",
      "author": "Jimmy",
      "content": "Hello, World"
    },
    {
      "title": "Mail 2",
      "author": "xkz",
      "content": "foobar"
    }
  ]
}
```

## 基本语法

JSON中有两种数据结构：Object（键值对、字典、对象）和Array（数组）。

Object中的数据为键值对，其基本形式是`{"key": "value"}`，key需要有双引号（这样key中就可以有特殊字符了）。键值对之间用逗号分割。

Array为集合，其基本形式是`[element1, element2]`，element可以是数值、字符串等基本类型，也可以是Array和Object。

Object和Array可以嵌套，Object可以作为Array的元素（mailList中的mail），Array也可以是Object中的某个value（mailList）。

## 注释

JSON是没有注释的。这也许是因为JSON是为了数据交换设计的，要尽量减少冗余吧。

不过也有个投机取巧的方法：建立一个不用的key，把注释内容写在value中。

```json
{
  "title" : "Mail3",
  "comment": "some comment"
}
```



## 总结

在我看来JSON的特点是抽象而简洁。抽象是相对于INI而言语法不那么直观，简洁是相对于XML而言冗余度很低，是很好的数据交换格式。

JSON无注释的特点使其不太适合作为配置文件使用，而适合作为数据交换格式使用。因此有人干脆使用js作配置文件用（webpack）。使用js除了能使用注释外，还有import其他文件、使用if语句、使用引用等高端功能。不过js只适用于有js执行环境的地方，大概也就是nodejs了。

# YAML

YAML是目前比较流行的配置文件格式，我最近在Hexo和Docker-Compose上遇到。YAML初看比较像Python，因为它用缩进来体现层级关系。不过它在逻辑上与JSON十分类似，并且具有一些JSON不具有的特性。YAML大概长这样（转自[YAML语言教程](http://www.ruanyifeng.com/blog/2016/07/yaml.html)）：

```yaml
users:
  - jimmy
  - xkz

# 锚点
defaults: &defaults
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  <<: *defaults # 引用

test:
  database: myapp_test
  <<: *defaults # 引用
```



## 基本语法

YAML使用缩进表示层级关系，同一层级的缩进必须保持一致。缩进只能使用空格不能使用tab，不过空多少格没有规定。

YAML中的数组表示方式为在元素前加上“`-`”（和Markdown很像）；键值对的表示方式和JSON差不多，不过YAML中的key和字符串不用加引号。

YAML中的数组和对象也可以嵌套，其逻辑上与JSON是一致的。

YAML可以使用注释，语法是在`#comment`（与Python类似）。

## 引用

在一些复杂工程的配置文件中往往会出现内容的重复，例如：对若干服务器配置同样的数据库连接，数据库连接的配置本身包括ip、用户名、密码等项目，而这些服务器都需要使用这个数据库连接配置。在JSON中只能把配置在所有服务器的配置中复制：

```json
{
  "servers":[
    {
      "ip": "192.168.2.1",
      "...": "...",
      "db":{
        "ip": "192.168.1.1",
        "username" : "jimmy",
        "..." : "..."
      }
    },
    {
      "ip": "192.168.2.2",
      "...": "...",
      "db":{
        "ip": "192.168.1.1",
        "username" : "jimmy",
        "..." : "..."
      }
    }
  ]
}
```

这样不仅造成了配置文件的冗余，还在修改时造成麻烦，需要对所有服务器进行改动。因此YAML中引入了引用机制：所有服务器引用同一份数据库配置，不仅能节约空间，也能便于改动。同样语义的YAML文件如下：

```yaml
db: &database
  ip: 192.168.1.1
  username: jimmy
  # ...
  
servers:
  - ip: 192.168.2.1
    db: *database
  - ip: 192.168.2.2
    db: *database
```

其中`&database`的含义为在此处设下锚点，而第8和第10行中`*database`为取这个锚点的值，也就是db对象。语法和C有些类似。

## 总结

YAML可以说是JSON的加强版，主要体现在加入了注释和引用机制，同时省去了让人眼花的方括号和花括号。使用缩进而不是括号表达层级关系可以让我们能用肉眼看出层级关系，而不需要使用编辑器或是IDE寻找配对的括号，同时省去了不必要的空行，提高了可读性。YAML与JSON相比在加强了表达能力的同时提高了可读性，可以说是在复杂项目中最佳的配置语言了。

# 总结

上面的分析似乎在吹YAML的同时把其他语言批判了一番，但是配置文件的选择还是需要考虑实际使用场景，不能见的风是的雨。对于一些简单场景INI就能很好地解决，没有必要使用YAML；对于前端工程，程序员很熟悉JavaScript的那一套，在JSON能凑合用的情况下就没有必要再学习一种语言了，对于更复杂的情况用js或许是更好的选择；而没有特定语言倾向的复杂工程，或许是YAML最能胜任的。