---
layout: post
title:  "Tour of Heroes in HTTP2"
date:   2016-05-30 22:42:56 -0400
categories: jekyll update
---
## Intro
As part of [my talk][youtube] at [ng-conf][ngconf] earlier this year I talked about the benefits of using Http2 over Http/1.

Seeing Torgeir's tweet reminded me that I should write a follow up article about it. 
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Had fun tweaking performance today. Cut article load by a factor of 5 through bundling and other tricks :-) <a href="https://t.co/RWr9AVCUsh">https://t.co/RWr9AVCUsh</a></p>&mdash; Torgeir Helgevold (@helgevold) <a href="https://twitter.com/helgevold/status/737053087043575808">May 29, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## Baseline
Starting with John Papa's [Tour of Heroes][tour] repo and decided to use it as a baseline.  As you can see from the screenshot it takes about `1.09s` to load. 

![Live Server Version]({{ site.url }}/assets/LiveServer.png)

## Http/1.1
Now that we have a baseline we can move on to serving the exact same content up via a node + express server.
{% highlight javascript %}
var express = require('express');
var app = express();

app.use(express.static('.'));

app.listen(3000, function () {
    console.log('Example app listening on port 3000!');
});
{% endhighlight %}

Switching to node + express shaved the time down a little bit to `981ms`.
This is due mostly to the fact that live server implements hot-reloading, a fantastic feature for development but it does add a little overhead in terms of assets and speed. 
![Node Http/1.1 Version]({{ site.url }}/assets/NodeExpressHttp1.png)

## Http2
Switching to Http2 is actually pretty straight forward.  First you have to update to Express 5.  Then you can make the following changes to your server javascript.  
Credit for this part goes to [tunniclm][github]. 

{% highlight javascript %}
var express = require('express');
var fs = require('fs');
var app = express();
var http2 = require('http2');

express.request.__proto__ = http2.IncomingMessage.prototype;
express.response.__proto__ = http2.ServerResponse.prototype;

app.use(express.static('.'));

http2.createServer({
        key: fs.readFileSync('./certs/server.key'),
        cert: fs.readFileSync('./certs/server.crt')
    }, app)
    .listen(3000, (err) => {
    if (err) {
        throw new Error(err);
    }
console.log(`Listening on port 3000!`);
});
{% endhighlight %}

Now that we switched over to Http2 we can see our final speed of `821ms`
![Node Http2 Version]({{ site.url }}/assets/NodeExpressHttp2.png)

[ngconf]: https://www.ng-conf.org/
[youtube]: https://www.youtube.com/watch?v=jxt8qe6DSOw
[tour]: https://github.com/johnpapa/angular2-tour-of-heroes
[github]: https://github.com/expressjs/express/issues/2761#issuecomment-216912022