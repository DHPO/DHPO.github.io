---
title: Typescript Spider 1：基础篇
date: 2018-03-24 15:15:43
tags:
  - nodejs
  - spider
  - typescript
---

## 前言

数据挖掘课的大作业快要发布了，在这之前不如先熟悉一下爬虫该怎么写，顺便也体验一下typescript.

根据“柿子得挑软的捏”原则，我就先拿cnBeta练练手(不用登录，不用设置user-agent)。

一个最简单的爬虫包括三个部分：

- 对给定的网址发送网络请求，并且获得回应
- 对于返回的页面(当然最好是json了，不过大多数情况还是只有html)进行解析，获得需要的数据
- 对于上一步中获取的数据，持久化到数据库

下面逐一展开。

## 网络请求

### 发送请求

这里使用的网络请求库是[`superagent`](https://www.npmjs.com/package/superagent)：

```bash
npm install superagent --save
npm install @types/superagent --save-dev
```

`superagent`的API很简单，如果不需要什么设置的话，可以这么写：

```typescript
superagent.get(url)
    .end((err, res) => {
    /* 处理response */
    });
```

或者干脆使用`await`（返回值是response，类型为`Promise<superagent.Response>`）：

```typescript
const remoteGet = async (url: string) => {
    return await superagent.get(url);
};
```

这里把它包装成另一个函数，如果之后要设置cookie之类的东西的话只要改这一个地方就行了。

接下来是用这个函数获取cnbeta的主页：

```typescript
const response = await remoteGet("https://www.cnbeta.com");
```

`response.text`就是页面的内容，接下来就是拿去解析。

### 简单的速率控制：sleep

为了防止爬虫爬得比香港记者还快，被网站批判一番，我们需要对爬取的速率做一些控制。

速率控制最简单的方式是完成一个请求以后sleep一段时间。不过JavaScript中没有原生的sleep函数，我们需要自己实现一个。

一个很tricky的方法是使用`setTimeout`做一个`async`函数：

```typescript
const sleep = (ms: number) => {
  return new Promise((resolve) => setTimeout(resolve, ms));
};
```

当`setTimeout`时间到了时，调用`resolve`，`Promise`返回。

不过在使用时要记得加上`await`：

```typescript
await sleep(1000);
```



## 页面解析

页面解析使用的库是[`cheerio`](https://github.com/cheeriojs/cheerio).它的特点是语法和jQuery十分相似，和scrapy里的xpath有很大差别。

首先是安装这个库：

```bash
npm install cheerio --save
npm install @types/cheerio --save-dev
```

使用的第一步是加载需要分析的字符串(也就是`response.text`)：

```typescript
const $ = cheerio.load(response.text);
```

这里使用`$`的原因是让它看上去和jQuery更加相似。

接下来就像使用jQuery一样，使用选择器进行匹配。不过要注意，除非使用`text()`、`attr()`等方法可以转化为`string`外，直接使用选择器得到的数据类型是`Cheerio`，使用`each()`方法遍历所有匹配到的节点时，得到的数据类型是`CheerioElement`。`CheerioElement`可以被进一步分析、选择。获取cnbeta首页的新闻的标题、链接和内容的代码如下：

```typescript
const $ = cheerio.load(response.text);
$(".item").each((index, element) => {
    const title: string = $(element).find("a").first().text().trim();
    const url: string = $(element).find("a").first().attr("href");
    const content: string = $(element).find("p").first().text().trim();
});
```

因为对这个库比较陌生，所以我显式地指定了变量的类型而不使用推断，防止得到预期之外的结果。

而对于发布时间的匹配则需要用到正则表达式，因为在网页上的时间格式是“发表于2018-03-24 12:00”：

```typescript
const rawTime: string = ($(element).find(".status>li").first().text()
        .match(/\d{4}-\d{2}-\d{2} \d{2}:\d{2}/g) || [""])[0];
const time: Date = new Date(rawTime);
```

正则表达式是js的内容，这里就不多说了。

现在已经把需要的内容都解析出来，接下来是把它们持久化到数据库中。

## 持久化

这里使用的数据库是MongoDB，连接数据库的库是[`mongoose`](http://mongoosejs.com/docs/index.html)：

```bash
npm install mongoose --save
npm install @types/mongoose --save-dev
```

### 建立连接

建立数据库连接的方式如下：

```typescript
import * as mongoose from "mongoose";

export const mongodb = async () => {
  await mongoose.connect("mongodb://localhost:27017/cnbeta");
  const db = mongoose.connection;
  db.on("error", (err) => {console.log(err); });
  return db;
};
```

我觉得这个库比较tricky的一点是，数据库连接似乎是一个全局状态，建立连接以后就不用管了，之后无论是保存还是查找都不需要把这个连接作为参数传入。

### 设置Schema

我对这个库底层的设计不太懂，所以直接贴代码了：

```typescript
import {Document, Model, model, Schema} from "mongoose";

interface INews extends Document {
  title: string;
  url: string;
  time: Date;
  content: string;
}

const NewsSchema = new Schema({
  content: String,
  time: Date,
  title: String,
  url: String,
});

export const News: Model<INews> = model<INews>("News", NewsSchema);
```

这里有一个问题：为什么一个`Schema`要定义两次？其实答案很简单：因为这是typescript，编译成JavaScript以后`INews`这种接口定义不会传递到JavaScript中，所以需要用`NewsSchema`用字典的方式将信息告诉mongoose。或者这么说：`INews`是给typescript编译器看的，`NewsSchema`是给mongoose看的。

经过一通操作，最终导出一个变量`News`，它包含了CRUD的方法。

### 保存与查找

保存只需要两步：

1. 设置属性值
2. 调用`save`方法

代码如下：

```typescript
const news = new News({title, url, time, content});
news.save((err) => {if (err) {console.log(err); }});
```

这里有个问题我不是很懂：`News`的类型是`Model<INews>`，是一个变量而不是一个类，为什么可以使用`new`？

在保存之前应该查一下数据库里面是不是已经有这条新闻了，如果有了就不要再保存了：

```typescript
const news = new News({title, url, time, content});
if (await News.findOne({title}).exec() === null) {
    news.save((err) => {if (err) {console.log(err); }});
}
```

这里的`News`才比较像是一般的变量的用法。

## 总结

到目前为止一个最简单的爬虫就完成了。把上面的函数整合到一起运行后，就能爬到许多数据了：

```
> db.news.find()
{ "_id" : ObjectId("5ab5c0300aae12190b0cb80d"), "title" : "[图]HTC U12+高清渲染图再曝光：额头下巴进一步收窄", "url" : "https://www.cnbeta.com/articles/tech/710023.htm", "time" : ISODate("2018-03-24T02:52:00Z"), "content" : "日前，爆料大神@evleaks表示HTC内部正在研发代号为“Imagine”的旗舰新机，在上市之后可能名为HTC U12+；今天外媒Techno Buffalo和Concept Creator带来了HTC U12 Plus的高清渲染图。根据内部消息称，在台湾地区仅会销售U12+，而不会销售常规的U12标准版本。而且从图片上来判断，两者可能会采用不同的处理器来区分市场。", "__v" : 0 }
{ "_id" : ObjectId("5ab5c0300aae12190b0cb80e"), "title" : "外媒披露《碟中谍6》独家剧照：阿汤哥高空扒飞机", "url" : "http://hot.cnbeta.com/articles/movie/710021", "time" : ISODate("2018-03-24T02:52:00Z"), "content" : "外媒Empire今天披露了一张《碟中谍6》独家剧照，该媒体表示这也是电影中最值得期待的镜头之一，一起来欣赏一下。在照片中阿汤哥悬身于正在爬升的直升机起落架上，徒手扒飞机的场景相当刺激。在此前公布的第一部预告中我们曾经看到过这个镜头，搭配食用效果更佳。", "__v" : 0 }
{ "_id" : ObjectId("5ab5c0300aae12190b0cb80f"), "title" : "《人民的名义》续集筹备中 豪掷4亿元再拍60集", "url" : "http://hot.cnbeta.com/articles/movie/710019", "time" : ISODate("2018-03-24T02:46:00Z"), "content" : "编剧周梅森接受采访称，去年大热的电视剧《人民的名义》续集《人民的财产》正在创作中，投资额高达4亿。据周梅森透露，《人民的财产》仍然是第一部大家所熟识的反腐题材，不过这次将重点放在民众关注的金融领域反腐和国企腐败。", "__v" : 0 }
......
```

接下来的提升方向有很多：

- 设置headers、模拟登录
- 模拟翻页
- 使用ip池
- 爬取结果的可视化展示
- 更加高级的速率控制方式

另外对于mongoose，还只是会用而不明白其原理。

还是要学习一个！