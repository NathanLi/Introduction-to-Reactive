# Introduction-to-Reactive
## 响应式编程简介
([原文](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754))

　　你应该对响应式编程这个新事件有点好奇吧，尤其是与之相关的部分框架：Rx、Bacon.js、RAC等等。

　　在缺乏好的资源的情况下，学习响应式编程成为痛苦。我开始学的时候，做死地找各种教程。结果发现有用的只是极少部分，而且这少部分也只是表面上的东西，对于整个体系结构的理解也起不了多大的作用。直接去看那些库文档同样也理解不了。比如下面这个：

> **Rx.Observable.prototype.flatMapLatest(selector, [thisArg])**

> Projects each element of an observable sequence into a new sequence of observable sequences by incorporating the element's index and then transforms an observable sequence of observable sequences into an observable sequence producing values only from the most recent observable sequence.

　　我擦，这究竟是什么鬼！

　　我看过两本书，一本就是在那画图，另一本则在教怎么用响应式库。

　　学习最困难的地方在于**响应式思维**。咱们得用不同于传统的方法来思考，而且还要尽量不用传统编程中的状态变量。我在网上没有找到任何关于这方面的东西，而我认为一个实用的教程就在于教会你怎么用响应式思维来思考，这样才能引导你入门。我希望这可以帮助你。

## “什么是响应式编程？”

　　在网上的解释和定义大多是很烂的。[维基](https://en.wikipedia.org/wiki/Reactive_programming)中的定义又太泛而且过于理论。[Stackoverflow](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming)上的标准答案对于新手而言又不太适合。 [Reactive Manifesto](http://www.reactivemanifesto.org/)听起来像是你在秀给你产品经理看似的。微软的 [Rx术语](https://rx.codeplex.com/) "Rx = Observables + LINQ + Schedulers" 这种微软式的说法，咱们大部分人是理解不了的。像“反应”和“变化传播”与典型的 MV* 没啥不同，现在的语言都是这么干的。我的视图当然反应于我的模型。变化当然会传播，如果不传播的话，那界面上不是不会变化了么！

　　好了，不扯蛋了。

#### 响应式编程就是异步数据流编程。

　　在某种程度上，这并不是什么新东西。事件总线(Event buses)或咱们常见的单击事件就是一个异步事件流，你可以观察这个流，也可以基于这个流做一些自定义操作（原文：side effects，副作用，本文皆翻译为自定义操作）。响应式就是基于这种想法。你能够创建所有事物的数据流，而不仅仅只是单击和悬停事件数据流。 流廉价且无处不在，任何事物都可以当作一个流：变量、用户输入、属性、缓存、数据结构等等。比如，假设你的微博评论就是一个跟单击事件一样的数据流，你能够监听这个流，并做出响应。

　　**最重要的是，有一堆的函数能够创建（create）任何流，也能将任何流进行组合（combine）和过滤（filter）。** 这正是“函数式”的魔力所在。一个流能作为另一个流的输入(input)，甚至多个流也可以作为其它流的输入。你能_合并（merge）_两个流。你还能通过_过滤（filter）_一个流得到那些你感兴趣的事件。你能将一个流中的数据_映射（map）_到一个新的流中。

　　如果说流是响应式的中心，那咱们就来仔细地研究一下，先从咱们熟悉的“点击按钮”事件流开始。

　　![单击事件流](http://i.imgur.com/cL4MOsS.png)

　　一个流就是一个**将要发生的以时间为序的事件**序列。它能发射出三种不同的东西：一个数据值（data value）(某种类型的)，一个错误（error）或者一个“完成（completed）”的信号。比如说，当前按钮所在的窗口或视图关闭时，“单击”事件流也就“完成”了。

　　我们只能**异步**地捕获这些发出的事件：定义一个针对数据值的函数，在发出一个值时，该函数就会异步地执行；针对发出错误时的函数；还有针对发出‘完成’时的函数。有时你可以省略这最后两个函数，只专注于针对数据值的函数。“监听”流的行为叫做**订阅**。我们定义的这些函数就是观察者。这个流就是被观察的主体(subject)（或“可观察的(observable)”）。这正是[观察者设计模式](https://en.wikipedia.org/wiki/Observer_pattern)。

　　在本教程中，会有一部分地方用ASCII来画图：
```
--a---b-c---d---X---|->

a, b, c, d: 发出的值(value)
X : 是一个错误(error)
| : '完成'信号(completed)
---> : 时间线
```

　　这已经够熟悉了，再说下去你就觉得烦了，咱来整点新玩意：咱们从点击事件流创建（通过转换）出新的点击事件流。

　　首先，创建一个counter stream来记录一个按钮被点击了多少次。在所有的响应式库(Reactive libraries)中，有很多关于流的函数，比如`map`、`filter`、`scan`等等。在你调用这些函数时，比如`clickStream.map(f)`，它会基于clickStream返回一个**全新的流**，也就是说，这个新的流随便怎么玩，也不会修改原来的clickStream。这就是所谓的**不可变**特性，这个特性与响应式流的结合极为nice。这使得咱们可以使用链式函数，比如`clickStream.map(f).scan(g)`：

```
  clickStream: ---c----c--c----c------c-->
               vvvvv map(c 变成 1) vvvv
               ---1----1--1----1------1-->
               vvvvvvvvv scan(+) vvvvvvvvv
counterStream: ---1----2--3----4------5-->
```

　　这个`map(f)` 函数根据你提供的`f`函数，将clickStream中发出的每一个值进行替换（替换后的值放到一个新的流中）。在咱们这个例子里，咱们直接将每一次点击都映射成为数字1。这个`scan(g)`函数会聚集流上之前所有的值，并得到一个值`x = g(accumulated, current)`，在这里`g`只是一个简单的(+)函数。此时，每当点击事件发生的注意，`counterStream`这个流就会发出一个点击次数总数值，如上图的1、2、3、4、5就是在点击后发出的点击总数。
```
  注：
     scan可以这么理解：假设一个数组（流与数组不一样）
     id 为任意类型
     id x;
     for (id current in array) {
       x = g(x, current);
       发出(x);
     }
      
```

　　为了显示响应式的强大之处，咱们来假设你想要一个“双击”事件流。为了使事情更有趣，咱们在这个流中，将多次点击（两次或两次以上）都当作是“双击”。深呼吸，然后想想用传统的编程方式该怎么来实现这个需求。我敢打赌，你会用一些变量来保存各种状态和计算时间间隔，这想想就好复杂。

　　而在响应式中，这却很简单。实际上，实现这个逻辑只需要[4行代码](http://jsfiddle.net/staltz/4gGgs/27/)就可以了。但咱们先忽略掉代码。图是理解和构建流的最好的方法，无论是你初学者还是专家：

　　![Multiple clicks stream](http://i.imgur.com/HMGWNO5.png)

　　灰框里是将一个流转换成另一个流的函数。首先，我们先把那些点击间隔在250毫秒内的点击累积到一个列表中（简单来说，也就是`buffer(stream.throttle(250ms))`做的事情。先别急着理解这些代码的细节，这里只是响应式的一个小示例而已），这就个返回了一个列表流（即a stream of lists），然后咱们再针对这个流使用`map()`将列表转换成为代表列表长度的整数。最后，咱们使用`filter(x >= 2)`函数来过滤掉那些整数。就是这样，经过3步操作，得到了咱们想要的流。咱们可以订阅(监听)这个流来做咱们想做的事情

　　我希望你会喜欢这个优雅地处理方式。这个例子仅仅是冰山一角，你可以用将相同的操作应用在不同的流上。比如API response流；另一方面，还有好多其它的函数可用。

## “为什么我应该考虑采用RP？”

　　响应式编程提高了代码的抽象层次，这样你就可以专注于你的业务逻辑的事件定义，而不是尝尝捣鼓那些大量的实现细节。RP的代码可能会更简洁、清晰。

　　对于现代web应用和移动应用这种众多UI事件与数据事件高度互动应用程序，好处更加明显。10年前，与web页面的交互基本上就是在后台提交一个表单，然后在前端进行简单的渲染。而现在的应用则更具实时性：修改一个单一表单字段可以自动触发保存到后端；“赞”某些内容则可以实时反映到其它相关联的用户那里，等等。

　　如今的应用有丰富的各式各样的实时事件，给用户一种高度互动的体验。我们需要工具来妥善处理这些事情，响应式编程是其中一个答案。

## RP思维实践

　　咱来整点真的。在这个真的例子中一步一步来教你怎么用RP来思考。这不是一堆的小例子，各种概念也会解释清楚。在教程的最后，咱们将会编出真正可用的代码，而且还理解咱们所做的每一件事。

　　我选择 **JavaScript** 和 **[RxJS](https://github.com/Reactive-Extensions/RxJS)** 作为本次教程的工具，原因是：JavaScript 是目前最广泛熟悉的语言，而 [Rx* 类库](http://www.reactivex.io) 是很多语言和平台所广泛采用的类库 ([.NET](https://rx.codeplex.com/), [Java](https://github.com/Netflix/RxJava), [Scala](https://github.com/Netflix/RxJava/tree/master/language-adaptors/rxjava-scala), [Clojure](https://github.com/Netflix/RxJava/tree/master/language-adaptors/rxjava-clojure),  [JavaScript](https://github.com/Reactive-Extensions/RxJS), [Ruby](https://github.com/Reactive-Extensions/Rx.rb), [Python](https://github.com/Reactive-Extensions/RxPy), [C++](https://github.com/Reactive-Extensions/RxCpp), [Objective-C/Cocoa](https://github.com/ReactiveCocoa/ReactiveCocoa), [Groovy](https://github.com/Netflix/RxJava/tree/master/language-adaptors/rxjava-groovy), 等等)。所以，基本上无论你的编程语言是什么，你都可以从本教程中受益。

## 实现一个关注推荐表 "Who to follow"

　　在 Twitter 中的关注推荐表是这样的：

　　![Twitter Who to follow suggestions box](http://i.imgur.com/eAlNb0j.png)

　　我们只关注模仿其核心功能：

　　* 启动时，从API读取帐户数据，并且显示3个推荐的帐户
　　* 点击 "Refresh"，读取另外3个帐户数据并放到推荐列表中
　　* 点击 'x' 按钮，清除按钮所在行的帐户数据，并显示另一个帐户
　　* 每一行显示帐户的头像和他们的页面链接

　　那些次要的功能和按钮咱就不管了。Twitter 最近关闭了其未授权公共API，所以咱们做一个关注 Github 用户的UI得了。这里是[获取 Github 用户的 API](https://developer.github.com/v3/users/#get-all-users)。

　　如果你想提前看看的话，这里有完整的代码 http://jsfiddle.net/staltz/8jFJH/48/。

## 请求和响应

　　**你怎么用 Rx 来处理这里问题？** 好了，开始，(基本上) _所有东西 都能当作是 流_。这是 Rx 的口头禅。咱们先从最简单的功能开始：“启动时，用 API 读取3个帐户数据”。 这没有什么特别的地方，也就是几个简单的步骤：(1)发出一个请求(request)，(2)获得到一个响应(response)，(3)渲染得到的响应数据。OK，我们继续，咱们把请求当作一个流。这有点小题大做了，但我们得从基础做起，对吧？

　　在启动时，我们只需要发送一个请求，所以我们将其建模为数据流，这个流只会发射一个值。咱们知道，接下来还会有很多请求，但现在只有一个。

```
--a------|->

这里，a 是一个字符串 'https://api.github.com/users'
```

　　这是我们想要请求的URLs流，当一个请求事件发生时，它发告诉咱们两件事情：when and what。“when” 是说，当发出一个事件时就表示应该开始执行那个请求。“what” 指的是这个请求发出的值：一个包含URL的字符串。

　　在 Rx* 中创建只包含一个值的流是非常简单的。在官方术语中，流是“可观察的”，也就是说它可被观察，但如果用“observable”来命名的话，则显得有点蠢了，所以我还是把它叫做 _stream_：

```javascript
var requestStream = Rx.Observable.just('https://api.github.com/users');
```

　　现在，这只是一个字符串的流，还没有做其它操作，在该值被发出时，咱们需要以某种方式做点什么事情。这可以通过[订阅(subscribing)](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypesubscribeobserver--onnext-onerror-oncompleted)这个流来完成。

```javascript
requestStream.subscribe(function(requestUrl) {
  // execute the request
  jQuery.getJSON(requestUrl, function(responseData) {
    // ...
  });
}
```

　　注意，在这里我们使用了 [jQuery 的 Ajax 回调](http://devdocs.io/jquery/jquery.getjson) 来处理这个异步的请求操作。但是先等一下，Rx 就是处理 **异步** 数据流的。这个请求的response不是会包含一些数据么，那咱们是不是也可以将这个response包装成一个流呢？从概念上来看可行，那咱们来试试看： 

```javascript
requestStream.subscribe(function(requestUrl) {
  // execute the request
  var responseStream = Rx.Observable.create(function (observer) {
    jQuery.getJSON(requestUrl)
    .done(function(response) { observer.onNext(response); })
    .fail(function(jqXHR, status, error) { observer.onError(error); })
    .always(function() { observer.onCompleted(); });
  });
  
  responseStream.subscribe(function(response) {
    // do something with the response
  });
}
```

　　[Rx.Observable.create()](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservablecreatesubscribe) 所做的就是创建一个你自己的流，在有数据事件(`onNext()`)或错误(`onError()`)时，这个流会通知其每一个观察者（或“订阅者”）。我们所做的只是对 jQuery Ajax Promise(注：[JS Promise 模式](https://www.promisejs.org/)) 的封装而已。**打断一下，这也就是说 Promise 是可观察的？**

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

![Amazed](http://www.myfacewhen.net/uploads/3324-amazed-face.gif)

　　没错！

　　在 Rx 中用 `var stream = Rx.Observable.fromPromise(promise)` 用可以将一个 Promise 转换成一个可观察的流，够简单吧。虽然 Observable 与 [Promises/A+](http://promises-aplus.github.io/promises-spec/) 不兼容，但从概念上来说并没有什么冲突。简单点说，一个 Promise 就是只发射一个值的 Observable。Rx流比 promises 多的就是能够返回多个值。 

　　这也就是说 Observables 至少也有 Promises 这么强大，如果你相信 Promises 的能力的话，那你也应该留意一下 Rx Observables。

　　现在回到刚刚那个例子，你有注意到 `subscribe()` 么，它就是用来回调的。`responseStream`的创建是依赖于`requestStream`的，创建这个流也还是很简单的吧。

　　接下来介绍 [map(f)](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypemapselector-thisarg) 函数，它是针对流A中的每一个值，运用 `f()` 产生一个值（即映射），并将这个产生的值由流B发射出来。如果将其用在咱们的请求和响应流上的话，咱们可以将请求URLs 映射成为响应 Promises（伪装为 streams）。

```javascript
var responseMetastream = requestStream
  .map(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });
```

　　这里创建了一个名为 “_metastream_” 的玩意儿：流中流（a stream of streams）。 别恐慌，metastream 也就是一个流，这个流发射出的值也是一个流。你可以把它当作[指针](https://en.wikipedia.org/wiki/Pointer_(computer_programming))：每一个发射出的值都是一个 _指针_ ，它指向另一个流。在这个例子中，每个请求URL被映射为一个指针，指向一个包含有response的promise流。

![Response metastream](http://i.imgur.com/HHnmlac.png)

　　response metastream除了使事情更复杂之外，看起来没什么其它用呀。咱们只是想要一个简单的response流，每次会发射出一个JSON对象的，而不是这种发射 'Promise'对象的。先来看看 [Flatmap](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypeflatmapselector-resultselector)：这是 `map()`的一个变种，能够 “整合(flattens)” metastream，经过整合后，“trunk”流发射出的值都来自于“branch”流。Flatmap 不是 “修复版”，metastream也不是一个bug，在 Rx 中，它们是处理异步responses很有用的工具。

```javascript
var responseStream = requestStream
  .flatMap(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });
```

![Response stream](http://i.imgur.com/Hi3zNzJ.png)

　　Nice。response 流是根据 request流定义的，**如果**我们接下来的 request流还有事件的话，咱们的 response流也将会产生相应的响应事件：

```
requestStream:  --a-----b--c------------|->
responseStream: -----A--------B-----C---|->

(小写字母是request, 大写字母是相应的response)
```

　　现在有了一个 response流，咱们可以根据咱们接受到的数据来渲染：

```javascript
responseStream.subscribe(function(response) {
  // render `response` to the DOM however you wish
});
```

　　到目前为止，代码如下：

```javascript
var requestStream = Rx.Observable.just('https://api.github.com/users');

var responseStream = requestStream
  .flatMap(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });

responseStream.subscribe(function(response) {
  // render `response` to the DOM however you wish
});
```

## 刷新按钮

　　这个 reponse 中的JSON里包含有 100 个用户数据。而这个API只能指定请求的页数（page offset），而不能指定请求每页的大小（page size），而咱们只需要3个就可以了，所以会有97个用户数据浪费掉。现在先忽略这个问题，待会再看怎么缓存reponses。

　　刷新按钮每点击一次，请求流都应该发射一个新的URL，然后我们就可以得到一个新的response。这需要做两件事：1、刷新按钮的点击事件流（所有事物都可以当作一个流）；2、更改请求流依赖于刷新按钮的点击事件流。RxJS有工具能根据事件监听器创建 Observables。

```javascript
var refreshButton = document.querySelector('.refresh');
var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');
```

　　刷新点击事件不会关联任何 API URL，我们需要将其映射到一个实际的URL上。现在咱们更改请求流的实现逻辑：对刷新点击流运用 map 函数，映射成为随机页面的API。

```javascript
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```
　　
　　这个请求在启动时什么也不会做，它只有在刷新按钮点击时才会被触发。请求会在这两种行为下发生：刷新按钮点击或打开网页。
　　
　　加上本例子最开始那个请求流，现在有两个请求流了。为了区分这两个流，分别给它们取个不同的名字：

```javascript
var requestOnRefreshStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
  
var startupRequestStream = Rx.Observable.just('https://api.github.com/users');
```

　　怎么将这两个流“合并(merge)”为一个流呢？ [merge()](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypemergemaxconcurrent--other) 函数就是专为来干这事的。用文字图来解释一下它做了些什么：

```
stream A: ---a--------e-----o----->
stream B: -----B---C-----D-------->
          vvvvvvvvv merge vvvvvvvvv
          ---a-B---C--e--D--o----->
```

　　现在合并两个流很简单了：

```javascript
var requestOnRefreshStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
  
var startupRequestStream = Rx.Observable.just('https://api.github.com/users');

var requestStream = Rx.Observable.merge(
  requestOnRefreshStream, startupRequestStream
);
```

　　还有一种替代的简洁方式，没有临时中间流：

```javascript
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  })
  .merge(Rx.Observable.just('https://api.github.com/users'));
```

　　更简单，更具有可讲性的写法：
```javascript
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  })
  .startWith('https://api.github.com/users');
```
　　
　　[startWith()](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypestartwithscheduler-args) 函数的功能如其字面意思。无论你的输入流是怎么样的，`startWith(x)` 输出流在开始的时候都会发射出 `x`。我这可不是在 [DRY (重复劳动)](https://en.wikipedia.org/wiki/Don't_repeat_yourself)，而是在对比各API（指这里的 startWith 与 merge 没什么区别）。可以将 `startWith()` 紧接在 `refreshClickStream` 后面，本质上这是在启动时 “模仿” 刷新按钮点击。

```javascript
var requestStream = refreshClickStream.startWith('startup click')
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```

　　Nice。将启动请求流是刷新按钮点击请求流合并，只加了一个函数：`startWith()` 而已。

## 推荐里的3个模型流

　　直到现在，我们也只是在 response 流里的 `subscribe()`里的渲染步骤中接触了一下 _推荐_
UI元素。此时刷新按钮带来了问题：当你点击`refresh`按钮后，当前这3个推荐却没有清空。新的推荐会在得到 response 显示，但为了 UI 看起来更自然，咱们需要在刷新按钮点击后清空当前的推荐。

```javascript
refreshClickStream.subscribe(function() {
  // clear the 3 suggestion DOM elements 
});
```
　　
　　别，兄弟别这么干。这么做可不好，这会导致有 **两个** 订阅者操作推荐 DOM 元素（另一个是 `responseStream.subscribe()`），这违反了 [关注点分离](https://en.wikipedia.org/wiki/Separation_of_concerns) 原则。你可曾记得：

&nbsp;
&nbsp;
&nbsp;
&nbsp;

![Mantra](http://i.imgur.com/AIimQ8C.jpg)

　　所以咱们将一个推荐作为一个流，它发射出的值就是包含推荐数据的 JSON 对象。我们为这3个推荐分别单独创建一个流，第一个推荐流：

```javascript
var suggestion1Stream = responseStream
  .map(function(listUsers) {
    // get one random user from the list
    return listUsers[Math.floor(Math.random()*listUsers.length)];
  });
```

　　另外两个，`suggestion2Stream` 和 `suggestion3Stream` 直接从 `suggestion1Stream` 复制过来就可以了。

　　去掉 response 流的 subscrbie() 函数调用，咱们这么来渲染：　　

```javascript
suggestion1Stream.subscribe(function(suggestion) {
  // render the 1st suggestion to the DOM
});
```

　　回到 “点击刷新，清除推荐数据”，我们将刷新按钮点击事件映射为一个 `null` 推荐数据，并将其合并到 `suggestion1Stream` 流中：

```javascript
var suggestion1Stream = responseStream
  .map(function(listUsers) {
    // get one random user from the list
    return listUsers[Math.floor(Math.random()*listUsers.length)];
  })
  .merge(
    refreshClickStream.map(function(){ return null; })
  );
```

　　当渲染的时候，我们将 `null` 当作是 “没有数据”，隐藏相应的 UI 元素。

```javascript
suggestion1Stream.subscribe(function(suggestion) {
  if (suggestion === null) {
    // hide the first suggestion DOM element
  }
  else {
    // show the first suggestion DOM element
    // and render the data
  }
});
```

　　图如下：

```
refreshClickStream: ----------o--------o---->
     requestStream: -r--------r--------r---->
    responseStream: ----R---------R------R-->   
 suggestion1Stream: ----s-----N---s----N-s-->
 suggestion2Stream: ----q-----N---q----N-q-->
 suggestion3Stream: ----t-----N---t----N-t-->
```

　　这里 `N` 代表的是 `null`。

　　同样，我们可以在启动的时候渲染 “空” 的推荐数据。只要给推荐流加上 `startWith(null)` 就可以了：

```javascript
var suggestion1Stream = responseStream
  .map(function(listUsers) {
    // get one random user from the list
    return listUsers[Math.floor(Math.random()*listUsers.length)];
  })
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
```

　　结果：

```
refreshClickStream: ----------o---------o---->
     requestStream: -r--------r---------r---->
    responseStream: ----R----------R------R-->   
 suggestion1Stream: -N--s-----N----s----N-s-->
 suggestion2Stream: -N--q-----N----q----N-q-->
 suggestion3Stream: -N--t-----N----t----N-t-->
```

## 清除一个推荐 和 缓存responses

　　还有一个功能要实现。每个推荐的后面都有一个 'x' 按钮能够清除当前行推荐数据，然后再读取另一个推荐数据并显示。你首先的想法可能是任何一个清除按钮点击后，发一个新的请求：

```javascript
var close1Button = document.querySelector('.close1');
var close1ClickStream = Rx.Observable.fromEvent(close1Button, 'click');
// and the same for close2Button and close3Button

var requestStream = refreshClickStream.startWith('startup click')
  .merge(close1ClickStream) // we added this
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```

　　这行不通。它们清除所有的推荐数据，然后再重新载入，而不是只是当前这一个有影响。有两种不同的方式来解决这个问题，我们通过复用之前的 responses 来解决这个问题。这个 API 的 response 每页有100个用户数据，而我们只用了3个，所以还有很多数据可以用来当作刷新数据，没有必要再发请求。

　　同样，咱们以流的方式来思考。当 'close1' 的一次点击事件发生时，我们就从 `responseStream` _最后发射_ 的 response 数据中随机取一条用户数据：

```
    requestStream: --r--------------->
   responseStream: ------R----------->
close1ClickStream: ------------c----->
suggestion1Stream: ------s-----s----->
```

　　在 Rx* 中有一个组合函数： [combineLatest](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypecombinelatestargs-resultselector) ，看上去正是我们想要的。两个流 A 和 B作为其输入，无论哪个流发射了值（两个流都至少要发射一次值才会触发），`combineLatest` 都会将两个流最近分别发射的值 `a` 和 `b` 组合起来，再输出一个值  `c = f(x,y)`， 这里的 `f` 是你定义的函数。如下图所示：

```
stream A: --a-----------e--------i-------->
stream B: -----b----c--------d-------q---->
          vvvvvvvv combineLatest(f) vvvvvvv
          ----AB---AC--EC---ED--ID--IQ---->

这里的 f 是一个 小写转大写 函数
```

　　我们针对 `close1ClickStream` 和 `responseStream` 两个流使用 combineLatest()，每当 close1 按钮一点击，我们都能获取到最后的 response，然后产生一个新的值给 `suggestion1Stream`。而且，combineLatest() 是对称的：只要 `responseStream` 一发射新的 response，它都会与 `close 1`按钮最后的点击组合触发，并产生一个新的推荐数据。这样，咱们就只需对前面 `suggestion1Stream` 的代码简单改造一下就可以了：

```javascript
var suggestion1Stream = close1ClickStream
  .combineLatest(responseStream,             
    function(click, listUsers) {
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
  )
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
```

　　还有一个问题没解决。combineLatest() 使用了两个资源的最后的数据，如果其中一个没有发射过的话，combineLatest() 返回的输出流就不会发射数据。如果你看了上面的文字图的话，你会发现当第一个流发射数据 `a`时，输出流并没有发射数据。当第二个输入流发射数据 `b` 时，输出流才产生一个值。

　　也有两种不同的文案来解决这个问题，我们依然采用最简单的那种，在启动时模拟 'close 1' 按钮被点击：

```javascript
var suggestion1Stream = close1ClickStream.startWith('startup click') // we added this
  .combineLatest(responseStream,             
    function(click, listUsers) {l
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
  )
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
```

## 结束

　　做完了。完整代码如下：

```javascript
var refreshButton = document.querySelector('.refresh');
var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');

var closeButton1 = document.querySelector('.close1');
var close1ClickStream = Rx.Observable.fromEvent(closeButton1, 'click');
// and the same logic for close2 and close3

var requestStream = refreshClickStream.startWith('startup click')
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });

var responseStream = requestStream
  .flatMap(function (requestUrl) {
    return Rx.Observable.fromPromise($.ajax({url: requestUrl}));
  });

var suggestion1Stream = close1ClickStream.startWith('startup click')
  .combineLatest(responseStream,             
    function(click, listUsers) {
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
  )
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
// and the same logic for suggestion2Stream and suggestion3Stream

suggestion1Stream.subscribe(function(suggestion) {
  if (suggestion === null) {
    // hide the first suggestion DOM element
  }
  else {
    // show the first suggestion DOM element
    // and render the data
  }
});
```

**整个儿能跑的例子：http://jsfiddle.net/staltz/8jFJH/48/**

　　例子虽小，但五脏俱全：用中心分离想法管理多个事件，甚至还有缓存。函数式风格代码更像是声明式代码：我们不是指定要执行的一串指令，而是通过定义流之间的关系来 **描述事情是什么**。比如，我们用 Rx 告诉计算机 _`suggestion1Stream` **是** 'close 1'按钮点击流与最新的 response 中的一个用户数据的组合，除了程序启动时或刷新事件发生时是 `null`_。
　　
　　同时，这里没有多少类似于 `if`、`for`或`while`之类的控制元素，也没有多少常见的回调函数。你甚至可以通过在 `subscribe()` 之前使用 `filter()` 来摆脱 `if` 和 `else`（不举例了，留给你当作练习）。在 Rx 中，有很多用来操作流的函数，如  `map`、 `filter`、 `scan`、 `merge`、 `combineLatest`、’ `startWith`和更多的控制流的事件驱动函数。这个函数集能让你以少量的代码实现更强大的功能。
