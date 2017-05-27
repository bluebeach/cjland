---
title: 【全栈之路】Node.js爬虫（一）
tags:
  - Node.js
date: 2016-11-22 02:01:08
categories: 全栈之路
---
### 使用nodejs爬取知乎的内容，然后在web中显示出来

> [知乎发现URL](https://www.zhihu.com/explore#daily-hot)
> 
> 统计今日最热的问题和回答

* 使用request发起http请求
* 使用了cheerio解析html
* 使用了mysql保存抓取的数据
* 使用了express作web展示

<!-- more -->
---

* 字段包括
    - 问题title
    - 热门回答url
    - 回答者姓名
    - 点赞数
    - 回答者badge
    - 回答者title
    - 回答内容摘要
    - 评论数

* 设计
    * fetcher.js: 负责下载网页
    * parser.js: 解析html返回需要保存的数据
    * zhihuDao.js: 数据表的操作

---

#### 1. 下载网页
<a href="#1">fetcher.js</a>

使用request库，很简单

#### 2. 解析网页
<a href="#2">parser.js</a>

打开[知乎的发现页面](https://www.zhihu.com/explore#daily-hot)，使用Chrome开发者工具仔细研究想要获取的数据对应的html元素在哪里。

![2.png](/images/2.png)

发现每一个热门的item都对应着一个class="explore-feed feed-item"的div，Good！

首先挑选出所有的item

```JavaScript
$('.explore-feed.feed-item').each
```

[jQuery遍历each()方法](http://www.w3school.com.cn/jquery/traversing_each.asp)

* 再根据[jQuery选择器语法](http://www.w3school.com.cn/jquery/jquery_ref_selectors.asp)，基本可以选择出想要的元素
    - ${a.question_link}: 获取class为question_link的a标签
    - $('.explore-feed.feed-item'): 获取class具有explore-feed且feed-item的元素
    - $('ul').attr('id'): 获取属性
    - $('.badge-summary > a').text(): 获取class为badge-summary的span标签下的a标签文本
    - item('.summary-wrapper > [title]'): 获取class为summary-wrapper下带有title属性的元素


#### 3. 保存数据
<a href="#3">zhihuDAO.js</a>

nodejs连接mysql

新建mysql表zhihu_explore用来存储爬取的数据(以后换成mongodb试试(づ｡◕‿‿◕｡)づ)

![3.png](/images/3.png)

```
insert: 
connection.query(sql, paramArray, function(err, result){})
```

* 什么时候需要关闭connection连接？
    - 在zhihuDAO.js中`exports.close`导出关闭mysql连接函数
    - 在parser.js中新增一个回调函数用来关闭mysql连接


#### 4. 抓取数据保存到数据库中
<a href="#4">main.js</a>

![4.png](/images/4.png)

#### 5. 启动express展示数据
![1.png](/images/1.png)

---

#### 附源码

<a name="1">
```JavaScript
// fetcher.js
var request = require('request')
var zhihuDAO = require('../scrapy/zhihuDAO')

exports.fetch = function (url, callback) {
    request(url, function (error, response, body) {
        if (!error && response.statusCode == 200) {
            callback(body, zhihuDAO.insert, zhihuDAO.close)
        }
    })
}
```
</a>

<a name="2">
```JavaScript
// parser.js
var cheerio = require('cheerio');

exports.parse = function (html, callback, final) {
    var $ = cheerio.load(html);
    $('.explore-feed.feed-item').each(function (index, element) {
        var item = cheerio.load(element);
        var question_title = item('a.question_link').text();
        var question_url = item('a.question_link').attr('href');
        var answer_name = item('.author-link').text();
        var support_count = item('.zm-item-vote-count.js-expand.js-vote-count').text();
        var answer_badge = item('.badge-summary > a').text();
        var answer_title = item('.summary-wrapper > [title]').attr('title');
        var answer_content_summary = item('.zh-summary.summary.clearfix').text();
        var answer_content_pic = item('.zh-summary.summary.clearfix > img').attr('src');
        answer_content_pic = answer_content_pic===undefined?'':answer_content_pic;
        var comment_count = item('.meta-item.toggle-comment.js-toggleCommentBox').text();

        var param = [removeN(question_title),
            removeN(question_url),
            removeN(answer_name),
            removeN(support_count),
            removeN(answer_badge),
            removeN(answer_title),
            removeN(answer_content_pic)+' '+removeN(answer_content_summary),
            removeN(comment_count)];
        callback(param)

    });
    final()
};

function removeN(str) {
    var res = '';
    if (str) {
        res= str.replace(/\\n/g, '');
    }else {
        res = str;
    }
    return res;
}
```
</a>

<a name="3">
```JavaScript
// zhihuDAO.js
var mysql = require('mysql')

var connection = mysql.createConnection({
    host:'xxx',
    user:'xxx',
    password:'xxx',
    database:'xxx'
});
connection.connect();

exports.insert = function (param) {

    connection.query('insert into zhihu_explore(qustion_title, answer_url, answer_name, support_count, answer_badge, answer_title, answer_content_summary, comment_count) values (?,?,?,?,?,?,?,?)', param, function(err, result) {
        if (err) {
            console.log('insert error - ', err.message)
            return ;
        }
    })
}

exports.close = function () {
    connection.end()
}
```
</a>

<a name="4">
```JavaScript
// main.js
var fetcher = require('../scrapy/fetcher');
var parser = require('../scrapy/parser');
fetcher.fetch('https://www.zhihu.com/explore', parser.parse);

// node main.js
```
</a>

