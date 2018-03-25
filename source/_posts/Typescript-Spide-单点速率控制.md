---
title: 'Typescript Spide 4: 单点速率控制——Promise链'
date: 2018-03-25 21:02:42
tags:
  - nodejs
  - spider
  - typescript
---

## 前言

之前提到速率控制的方式太僵硬，相当于是把整个程序写成完全串行的形式。而比较理想的控制方式是`remoteGet`函数自身保证请求发送的间隔，调用者只需要静静地`await`就行了。

## 启发

我偶然间看到了这篇文章：[JavaScript Promise：简介](https://developers.google.com/web/fundamentals/primers/promises?hl=zh-cn)，其中讲到了一个场景：并行地加载一部小说的各个章节，但是要按照顺序将它们显示出来。文中的解决方案是（中文注释是我加的）：

```javascript
getJSON('story.json').then(function(story) {
  addHtmlToPage(story.heading);

  // Map our array of chapter urls to
  // an array of chapter json promises.
  // This makes sure they all download parallel.
  return story.chapterUrls.map(getJSON) /* 生成好多个Promise */
    .reduce(function(sequence, chapterPromise) {
      // Use reduce to chain the promises together,
      // adding content to the page for each chapter
      return sequence.then(function() { /* 把Promise串起来 */
        // Wait for everything in the sequence so far,
        // then wait for this chapter to arrive.
        return chapterPromise;
      }).then(function(chapter) {
        addHtmlToPage(chapter.html);
      });
    }, Promise.resolve());
}).then(function() {
  addTextToPage("All done");
}).catch(function(err) {
  // catch any error that happened along the way
  addTextToPage("Argh, broken: " + err.message);
}).then(function() {
  document.querySelector('.spinner').style.display = 'none';
})
```

这种实现思路是：先用map生成一系列的Promise，让这些Promise先跑起来，保证下载的并行性；之后再形成(`getJson`)->(`addHtmlToPage`)->(`getJson`)->(`addHtmlToPage`)->...的序列，保证输出的有序性。

这启示了我这几件事：

- Promise的运行和`await`并不一定要在一起
- Promise可以形成很长的链（可以在中间插入几个sleep）
- Promise可以嵌套

## 实现

所有的改动都在`remoteGet`函数上：

```typescript
let sequence = Promise.resolve();

export const remoteGet = async (url: string) => {
    return new Promise<superagent.Response>((resolve, reject) => {
        sequence = sequence.then(() => {
            return sleep(config.spider.interval * 1000);
        }).then(async () => {
            console.log(`GET ${url}`);
            const response = await superagent.get(url);
            resolve(response);
        }).catch((err) => {
            reject(err);
        });
    });
};
```

解释一下：

- `sequence`是一个Promise链，每次调用`remoteGet`时往链上加上一个`sleep`和之前发送请求的函数(`superagent.get(url)`)
- 当对`remoteGet`进行`await`的时候，首先等待外层的Promise的执行。外层的Promise什么时候resolve呢？要等到sequence运行到第10行。
- 根据Promise链的特性，要sequence运行到第10行，首先要运行完sequence链之前的内容，包括之前所有的`sleep`
- 为了防止中途有些链接出错而导致整个链终止，在后面需要加上`catch`

虽然`remoteGet`早早地返回了Promise，但是这个Promise实际上要等到sequence执行到第10行才会resolve，`await remoteGet`的函数才会继续执行下去。实际上`await`Promise的resolve和返回Promise是分离的；而不像之前那样，在`remoteGet`中阻塞一段时间后才返回一个Promise，Promise返回后不久就会resolve。

这种设计解决了这么几个问题：

- 网络请求的阻塞不影响页面解析和数据库访问
- 爬虫的其它部分不再需要考虑限速问题，不需要写成串行的形式，因此又可以使用`Cheerio`库中的`each`了

## 优化

上面的写法虽然好用，但还是有一个问题：sequence容易变得很长。这对调度器似乎不怎么友好。

另外，把`Cheerio`改回到`each`的形式后，`await remoteGet`不会阻塞`parse`函数，因此爬虫会一次性爬下很多页的链接，全都加到sequence上。

因此可以考虑让`parse`等到所有的`await remoteGet`都返回以后再返回，也就是等到一页的新闻爬取完成后再加载下一页。

一个比较naive的实现是加上一个计数器：`each`的第一步是将计数器+1，下载、解析完写到数据库中以后再-1。那么计数器就代表正在处理的新闻的数量，当计数器为0时说明都处理完了：

```typescript
const $ = cheerio.load(response.text);
let count = 0;
$("li.clear").each(async (index, element) => {
    count += 1;
    const title: string = $(element).find("a").first().text().trim();
    if (title === "" || await News.findOne({title}).exec() !== null) {
        return;
    }
    const url: string = "https://m.cnbeta.com" + 			  $(element).find("a").first().attr("href");

    let detail: string[] = [];
    try {
    	detail = await getContent(url);
    } catch (err) {
        console.log(err);
        count -= 1;
        return;
    }
    const content = detail[0];
    const rawTime = detail[1];
    const time: Date = new Date(rawTime);

	const news = new News({title, url, time, content});
    news.save((err) => {if (err) {console.log(err); }});
    count -= 1;
});
while (count > 0) {
    await sleep(1000);
}
```

对element的遍历可以使用`each`了，也避免了之前提到的typescript的bug。

这样就保证sequence的长度不会太长（最多就是每页新闻数*2）.