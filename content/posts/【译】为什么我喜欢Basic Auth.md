---
title: "【译】为什么我喜欢Basic Auth"
date: "2017-09-25"
draft: false
tags: ["code"]
keywords: ["编程", "Basic Auth"]
category: "技术"
---

**原文:**[Why I Love Basic Auth](http://www.rdegges.com/why-i-love-basic-auth/)

我发现了近些年一个让我比较不爽的趋势，越来越多的API服务支持了OAuth，却慢慢地放弃了HTTP基本验证（HTTP Basic Authentication 也叫Basic Auth）的支持。

我作为一个：

+ 使用了多年的REST API服务
+ 开发过很多的REST API服务
+ 曾经创建并运营了一家提供REST API服务的公司
+ 目前在一家大型面向开发者提供REST API的公司工作

的人，我不由自主地感觉到这是一件坏事情。

因为OAuth（现在非常流行）无论对API服务的开发者和使用者来说都是巨大的痛点。

OAuth复杂，难理解，广泛滥用，缺乏统一的实现。几乎可以肯定的说，它有它的用处，但是大多数情况下感觉是负担。

而Basic Auth简单，很好理解，并且从90年代开始每一种语言和框架都支持。

## Basic Auth是如何工作的

谈谈Basic Auth：

+ 它拥有明确的[规范](http://tools.ietf.org/html/rfc2617)。
+ 从1996年开始广泛使用。
+ 超级简单。

以下是一个简短的示例，展示了它是如何工作的。

+ 你是个开发者。
+ 你拥有API钥匙对：一个API Key ID 和一个API Key Secret。两者都是随机生成的字符串（一般是[uuid](http://en.wikipedia.org/wiki/Universally_unique_identifier)）
+ 想要进行API服务的验证时，所有你需要做的只是将你的验证信息塞进[HTTP Authorization Header](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)里。

这是使用命令[cURL](http://curl.haxx.se/)来进行Basic Auth验证的例子：

```bash
$ curl —user ‘xxx:yyy’ https://api.someapi.com/v1/blah
```

下面用Python来展示它的工作过程：

```python
>>> from requests import get
>>>
>>> api_key_id = ‘xxx’
>>> api_key_secret = ‘yyy’
>>>
>>> rep = get(‘https://api.someapi.com/v1/blah', auth=(api_key_id, api_key_secret))
>>> print resp.json()
```

如果觉得这些还不够，下面是Node.js版本，也能很好的展示：

```javascript
var request = require(‘request’);

var api_key_id = ‘xxx’;
var api_key_secret = ‘yyy’;

request({
    url: ‘https://api.someapi.com/v1/blah',
    auth: {
        ‘user’: api_key_id,
        ‘pass’: api_key_secret
    }
}, function(err, rest, body) {
   console.log(body);
});
```

是不是很简单？

Basic Auth你所需要做的只是指定你的凭证（在较高的抽象级别）。仅此而已。

**NOTE:** 是的，我知道在技术层面上比这复杂。它使用了`base64`编码，这是HTTP Authorization的头格式，等等。但是因为这些对于本文不重要，为避免歧义我就不说了。

Basic Auth对开发者非常友好，因为它简单，直观，易用。

假设你正在集成一项API服务，所有你需要做的只是创建一个API钥匙对，然后开始创建请求。

如果你意外的泄露了你的钥匙对怎么办？这会带来风险嘛？好吧......这时候你只需要创建一个新的API钥匙对，添加进你应用的代码中，旧的钥匙对就没用了。

正确的使用时，Basic Auth对于加强REST API的安全性是一个很好的选择。

## 简单

我最喜欢Basic Auth的就是简单。

不像OAuth一样，你不需要一个中间步骤来获取`access token`，你只需要你的钥匙对。

对于开发效率来说，这是一个巨大的差异。

你不需要花大把的时间来弄明白你需要被授予什么范围，什么权限，设置重定向页面、web 服务，所有的这些只是将事情弄的复杂而已。你只需要把你的API钥匙对放进HTTP Authorization头和BAM里就行。

因为它非常简单，所以你能快速地完成大量的工作和测试。

同样，你不再需要担心像下面的这样的事：

+ token过期。
+ 提供者更改他们的实现。几乎每一个OAuth提供者都好多次地更改他们的实现，搞挂成千上万开发者的应用。
+ 获取你数据的流程非常复杂。OAuth2有4种不同的授权方式，每一种都有不同的目的和不同的使用方式。

每一次我使用像[Twilio](https://www.twilio.com/)这样的服务时，都再次被提醒要是使用Basic Auth该多方便啊。能做到下面的这些真的很爽：

+ 使用命令行生成和销毁API keys。
+ 不论我目前在开发什么都能简单的把API keys塞进去。
+ 能使用命令行或者像[Runscope](https://www.runscope.com/)这样的API工具来进行测试。

我只是喜欢。

## 安全性

Basic Auth获得了不安全的坏名声，但是这不一定是正确的。

有非常多的事情你可以去做来使API服务（使用Basic Auth）尽可能地安全：

+ 所有的请求都使用HTTPs。如果你没有使用SSL，那么不管你使用什么验证协议，都是不安全的。除非你使用HTTPs，否则你的所有证书在网络上都是明文传输的，可怕的事情。
+ 使你的开发者们能生成尽可能多的密钥对。这样他们就能很方便地一个应用或服务使用一个密钥对。
+ 让你的开发者在他们需要的时候能够撤销密钥对。例如：一个开发者意外地在Github上泄露了密钥对，他就能够撤销原来的密钥对，确保其他人也无法使用。
+ 使用[uuids](http://en.wikipedia.org/wiki/Universally_unique_identifier)来随机生成密钥对。这确保了密钥对不会被猜中。

此外，开发者使用Basic Auth的时候，需要记住下面的事情：

+ 安全的保存你的API密钥对。如果你使用的服务支持Basic Auth，确保不做像把密钥对保存在Github repo中这样的事情。
+ 不使用不是跑在HTTPs下的API服务—迟早会出事的。
+ 每个应用使用不同的API密钥对。这样的话，如果你不小心泄露了一个密钥对，你只需要撤销那一个，更新基于那个密钥对的代码库就行了。从长远看来，这可以让你的生活过的更轻松。

如果你正寻找一个很好地处理了API密钥的例子，看看[AWS](http://aws.amazon.com/)吧。

## 普遍的支持

Basic Auth另一个很棒的一点是它只有一种实现，这意味着如何进行请求或者服务端组件都没有任何模糊不清的地方，它总是一样的。

它是这么工作的：

+ 你获得API Key ID和API Key Secret，然后把他们塞进一个使用冒号分隔的字符串中，比如：`’xxx:yyy’`。
+ 然后在前面加上单词`’Basic ‘`，就变成了：`’Basic xxx:yyy’`。
+ 然后将API key的一部分进行`base64`转码，最终得到：`’Basic eHh4Onl5eQ==‘`。
+ 最后，在创建HTTP请求的时候将这个值设为HTTP Authorization头部。

当你的web服务器接收到请求之后：

+ `base64`解码头部值。
+ 根据冒号分割字符串。
+ 左边部分是API Key ID，右边部分是API Key Secret。
+ 服务器进行验证，然后要么通过请求，要么返回`HTTP 401 UNAUTHORIZED`。

Basic Auth没有任何歧义。这就意味着每一种编程语言都有一流的支持，并且对开发者和使用者都很容易找到可信赖的库。

## 我的期望

我的期望就是，在OAuth继续流行的同时，开发者尽最大的可能继续支持Basic Auth。

这不仅仅让我作为开发者更加的简单，也更加的有意思。

对任何相信他们开发人员的API服务来说，支持Basic Auth绝对没错。

此外，甚至对于那些不相信他们的开发者的服务—也就是像Google，Facebook，Fitbit等第三方服务—支持Basic Auth依然是不错的想法。最起码，可以不用进入OAuth的圈子就能授予他们的用户更实用的获取他们自己的数据的权力。

**NOTE：**我意识到这会是个多少会引起争议的话题。我不再继续写关于这篇文章的东西，会直接丢掉他们，因为他们的范围已经超出太多了。后面我计划写更多文章来讨论关于OAuth和Basic Auth的各式各样的坑，为我的观点提供更深入的技术原因。

*PS*: 看到这篇文章，感触颇深。OAuth和Basic Auth应该是各有用处。但是原文作者吐槽的也不无道理。在使用了几次国内的OAuth服务之后，真是心好累。应用审核，调试，并且各种OAuth的认证过程又有着细微的不同，比如微博、QQ、人人等，基本都有区别。调试更是坑了。不管如何，读完这篇文章，对于Basic Auth还是能增进不少了解的。