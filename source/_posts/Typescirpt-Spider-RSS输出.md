---
title: 'Typescirpt Spider 2: RSS输出'
date: 2018-03-24 22:45:22
tags:
  - nodejs
  - spider
  - typescript
---

## 前言

数据抓取完成后，一种不错的展示方式是生成一个RSS源（虽然RSS已经凉了）。

现在大概要干这么几件事：

- 使用express框架跑一个服务器
- 按照RSS的格式输出XML，这最好使用模版实现
- 之前爬的是文章简介，现在要爬全文
- 爬得多了需要控制速度

## Express

### 安装

[`express`](http://expressjs.com/)是目前最流行的nodejs服务器框架。首先安装它：

```bash
npm install express --save
npm install @types/express --save-dev
```

### HelloWorld

```typescript
import * as express from "express";

const app = express();

app.get("/", (req: Request, res: Response) => {
    res.send("Welcome to CnBeta Spider");
  });
  
app.listen(8080);
```

express的书写范式和SpingMVC还是挺相似的，使用`get`、`post`等方法匹配协议，使用第一个参数`url`匹配链接，然后传入一个handler，从request中读取请求数据，将结果写入response中。

更复杂的操作（中间件等）在这里就不多述了。

现在，理论上我们可以按照格式把XML写到response中实现RSS。不过，还是用模板比较好。

## Pug

### 安装

[pug](https://github.com/pugjs/pug)是一种基于Nodejs的模版引擎（原来叫Jade，我之前只听说过Jade）。首先是安装：

```bash
npm install pug --save
npm install @types/pug --save-dev
```

### 语法

pug的语法是基于缩进的。每一行的第一个单词表示标签名，之后的是标签的数据（不是元数据）。如果下一行的缩进比这一行更多，那么下一行的标签就是这一行的儿子。

pug也可以在中间引入变量，也有逻辑控制和遍历。这些在模板渲染时根据传入的数据填写这些变量。

最终的模板是这样的：

```jade
doctype xml
rss(version="1.0")
  channel
    title cnbeta-rss
    link https://www.cnbeta.com
    description 中文业界资讯站
    each n in news
      item
        title #{n.title}
        link #{n.url}
        description #{n.content}
        pubDate #{n.time}
```

每一个`item`就是一条新闻。因为新闻有很多条，所以使用`each ... in ...`遍历所有的`news`（这里的`news`是所有新闻的数组）。

### 与express的集成

首先需要在express中设置渲染引擎和模板的位置：

```typescript
app.set("view engine", "pug");
app.set("views", __dirname + "/Views");
```

然后在handler中使用`render`方法渲染，并传入参数`news`（模板在`/Views`目录下，文件名为`rss.pug`）：

```typescript
app.get("/rss", async (req: Request, res: Response) => {
    try {
      const news = await News.find({}).sort({time: -1}).exec();
      res.render("rss", {news});
    } catch (err) {
      console.log(err);
      res.send("Error");
    }
  });
```

使用Firefox访问`http://localhost:8080/rss`就可以看到渲染结果了：

{% asset_img 20180324232019.png %}

express可以把渲染好的页面存起来，避免重复渲染相同的页面。要启用这个功能，需要加一行设置：

```typescript
app.enable("view cache");
```



## npm script

现在还有一个问题：模板作为源文件应该放在`/src`目录下，但是编译为js后却是在`/out`目录下寻找。所以在build的时候需要把模板复制到`/out`目录下。另外我们也希望在build完成后直接运行`main.js`。

要达到这两个目的，可以修改`package.json`，使用npm script实现：

```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "tsc && cp src/Views/* out/Views/*",
    "postbuild": "node out/main.js"
  }
```

其中`build`和`postbuild`是新加的。`build`完成了对`tsc`的调用和对模板的复制；`postbuild`会在`build`之后自动执行，其目的是运行`main.js`。之后在终端中输入`npm run build`即可执行这两个脚本。

## 全文爬取

之前的新闻的`content`是直接从首页中获取的，只有内容的概述。现在我们希望获得新闻的全文。

由于之前我们已经获得了新闻的链接，因此我们可以直接访问新闻的页面，提取出正文的内容：

```typescript
const getContent = async (url: string) => {
    const response = await remoteGet(url);
    const $ = cheerio.load(response.text);
    let result: string = "<p>" + $(".article-summary>p").html() || "" + "</p>";
    $("#artibody>p").each((index, element) => {
        result += "<p>" + $(element).html() || "" + "</p>";
    });
    return result;
};
```

提取的方式是找到所有的`<p>`标签，取出标签中的内容，再加上`<p>`标签后拼接在一起。

`parse`函数中获取`content`的语句也要做相应的修改。

## 爬取速度控制：`forEach`中的`await`

cnbeta每页有25条新闻，如果同时发起25次`getContent`容易被服务器封禁ip，因此需要对访问速度进行限制。

首先想到的方法是，在`remoteGet`方法中加一句`await sleep(1000)`，这样就可以保证每秒发起一次请求。

不过测试的结果和预期并不一样：实际情况是**过了1s后同时发起25个请求**，而不是每隔1s发起一个请求。

翻了文档以后发现，`Cheerio`的`each`并不会等到callback函数执行完毕才运行下一个（也就是说并不会`await callback`），这样实际上是生成了25个等待的线程，和预期产生了不同的结果。

现在js的`for (... on ...)`语句是可以等到循环体执行完毕后再运行下一个的（也就是会`await`循环体）。所以现在的思路是使用`for (... on ...)`遍历所有的`element`：

```typescript
const items: CheerioElement[] = $(".item").get();
    for (const element of items) {
        const title: string = $(element).find("a").first().text().trim();
        if (title === "" || await News.findOne({title}).exec() !== null) {
            continue;
        }
        const url: string = $(element).find("a").first().attr("href");
        let content: string = "";
        try {
            content = await getContent(url);
        } catch (err) {
            console.log(err);
            content = $(element).find("p").first().text().trim();
        }
        const rawTime: string = ($(element).find(".status>li").first().text()
            .match(/\d{4}-\d{2}-\d{2} \d{2}:\d{2}/g) || [""])[0];
        const time: Date = new Date(rawTime);

        const news = new News({title, url, time, content});
        news.save((err) => {if (err) {console.log(err); }});
    }
```

`$(".item").get()`返回所有的`CheerioElement`的集合。

这里我遇到了一个typescript的坑：编译时报了一个错误：

```bash
src/spider.ts(11,11): error TS2322: Type 'string[]' is not assignable to type 'CheerioElement[]'.
  Type 'string' is not assignable to type 'CheerioElement'.
```

意思是，编译器认为`$(".item").get()`的返回值类型是`string[]`，与预期类型不服。但是翻看`Cheerio`的类型声明，可以看到：

```typescript
get(): string[];
get(): CheerioElement[];
get(index: number): CheerioElement;
```

也就是说这个函数有三种重载类型，编译器取的是第一种，而我需要的是第二种。

理论上，在我钦定了返回值的类型后，编译器是有足够的信息判断适用哪一种，但是typescript的做法是取匹配到的第一个。这一点在官方文档中也提到了：

> *Don’t* put more general overloads before more specific overloads:

```typescript
/* WRONG */
declare function fn(x: any): any;
declare function fn(x: HTMLElement): number;
declare function fn(x: HTMLDivElement): string;

var myElem: HTMLDivElement;
var x = fn(myElem); // x: any, wat?
```

> *Do* sort overloads by putting the more general signatures after more specific signatures:

```typescript
/* OK */
declare function fn(x: HTMLDivElement): string;
declare function fn(x: HTMLElement): number;
declare function fn(x: any): any;

var myElem: HTMLDivElement;
var x = fn(myElem); // x: string, :)
```

解决方案有两个：

1. 修改类型声明，将我需要的移到第一个
2. 修改代码，适用第一个函数

如果需要将代码共享给他人使用，那么只能使用方案二，因为你无法要求他人改库的类型声明。

不过这只是我的个人玩具，为了方便还是选择了方案一。

## 总结

目前比较好地实现了对新闻的爬取和解析工作，不过还没有很好地支持RSS标准中反人类的时间格式规范。

接下来还是要尝试爬一些不同的网站，比如说做个知乎日报的RSS。