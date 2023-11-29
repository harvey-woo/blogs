如何正确处理promise、await产生的错误
=============================

最近在一些小童鞋的群上，讨论了一些很普遍的错误处理的话题。
发现大部分初级甚至中级前端，都不知道如何系统的处理一个错误。普遍出现
到处try...catch的问题。甚至有些公知，还推出各种奇怪的库来处理异步问题。
本文就以一个很常见的例子，说明我是如何处理错误的。

假设一个场景：
  - 根据token获取用户信息
  - 根据用户信息的 ```.vipRank``` vip等级推送广告
  - 上述每个过程都需要请求api

我们有两个实体

```typescript
interface User {
  name: string
  vipRank: number
  // ... 省略
}
interface Ad {
  // ... 省略
}
```

我们很自然会把这个过程拆成三个函数

```typescript
async function getUserInfo() {
  return (await axios.get('user/get-info')).data;
}

async function getUserAds(rank: number) {
  return (await axios.get('user/recommand-ads', { rank })).data;
}

async function main() {
  const info = await getUserInfo();
  const ads = await getUserAds(info.vipRank);
  console.log(ads);
}

// 如果你看不懂async函数，以下代码是等价的

function getUserInfo() {
  // 注意，必须return才是等价的，很多初级程序不返回，高级程序员（包括我）经常粗心漏了
  return axios.get('user/get-info').then(({data}) => data);
}

function getUserAds(rank: number) {
  // 记得return
  return axios.get('user/recommand-ads', { rank }).then(({data}) => data);
}

function main() {
  // 记得return
  return getUserInfo().then((info) => {
    const ads = getAds(info.vipRank);
    console.log(ads);
  });
}

```


## 那么问题来了，我们应该在什么地方进行捕获错误

这个问题我们先放下，我们先剖析以下在不同地方捕获，会出现什么情况

假如在 ```getUserInfo``` 和 ```getUserAds``` 捕获

```typescript

async function getUserInfo() {
  try {
    return (await axios.get('user/get-info')).data;
  } catch(e) {
    console.error(e);
  }
}

async function getUserAds(rank: number) {
  try {
    return (await axios.get('user/recommand-ads', { rank })).data;
  } catch(e) {
    console.error(e);
  }
}

// 同样，promise的例子

function getUserInfo() {
  // 再次强调，必须return才是等价的
  return axios.get('user/get-info')
    .then(({data}) => data)
    .catch((err) => {
      console.error(e);
    });
}

// getUserAds 我就不写了，自己脑补吧

```

在不改main函数的情况，如果一切请求正常的情况，用户会拿到 Ad 实体的数组

但是如果用户在如下步骤请求错误了：

### ```getUserInfo``` 出现错误

假设用户token异常，或者token过期，服务的返回 ```401 Unauthorized```

```typescript
// 下面代码不再提供promise，均以await演示


async function getUserInfo() {
  try {
    return (await axios.get('user/get-info')).data;
  } catch(e) {
    console.error(e);
    // ⬇ 其实这里相当于少了一句
    // return undefined;
  }
}

async function main() {
  const info = await getUserInfo();
  // ↑ 由于 info 是 undefind
  // 我们这里会收获一个 
  // Uncaught TypeError: Cannot read properties of undefined (reading 'vipRank')
  // 并且导致程序被中断
  const ads = await getUserAds(info.vipRank);

  // 程序到达不了这个位置，也到不到getUserAds
  console.log(ads);
}

```

可以看出，我们意外收获了一个

```Uncaught TypeError: Cannot read properties of undefined (reading 'vipRank')```

这不是我们想要的，也是我经常说的 “错误转移”，什么是错误转移，就是原本不是这一个错误
但是由于错误的捕获，导致其他错误冒出来

当然，有些同学说，我们可以继续增加try，避免意外的错误

```typescript

async function main() {
  try {
    const info = await getUserInfo();
    const ads = await getUserAds(info.vipRank);
    console.log(ads);
  } catch(e) {
    console.error(e);
  }
}

```

但是这样倒推下去会存在三个问题

- 假设我们的main函数不是程序入口，我们需要一直添加try，直到到真正的入口为止
- 会显示多个console.error，取决于我们catch多少次，而且很多错误根本的显示不是预期的
- 如果调用栈十分深，我们根本不知道原本的错误是什么


## 正确做法，Let it throw

正确的写法，其实一开始已经是对的了，就是都不catch

```typescript
// 下面代码不再提供promise，均以await演示


async function getUserInfo() {
  // 为了方便理解，我把代码拆成两行
  const res = await axios.get('user/get-info');
  // ↑ 如果上述逻辑错误，自动抛出，程序会被中断，往后的所有代码均不会被执行

  // 程序到达不了这个位置
  return res.data;
}

async function main() {
  const info = await getUserInfo();
  // ↑ 由于程序已经中断，后续代码根本不会执行

  // 程序到达不了这个位置
  const ads = await getUserAds(info.vipRank);
  console.log(ads);
}

```

我们会获得一个由于请求错误导致的 ```401 Unauthorized``` 错误
这个错误，才是我们需要的，而且不会出现上述问题

那么有些同学问，那么我应该怎么让用户知道用户没登陆导致错误：

我们应该利用一些系统钩子，或者框架，库给我提供的生命周期函数来处理

```typescript

// 错误处理器
function errHandler(err: unknown) {
  console.error(error);
  alert(error); // 当然，你可以用ui库提供的toast，或者message等
}

// 所有没有被catch的promise都会落入这个事件
window.addEventListener('unhandledrejection', (evt) => {
  const error = evt.reason;
  
  // 可以将错误转发到error
  window.reportError(error);
  
  // 或者交给错误处理器处理
  errHandler(error)
});


window.addEventListener('error', (error) => {
  // 交给错误处理器处理
  errHandler(error)
})

```

- [unhandledrejection 事件](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/unhandledrejection_event)
- [error 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event)
- [reportError api](https://developer.mozilla.org/en-US/docs/Web/API/reportError)

当然，如果我们浏览器版本比较低，不支持部分api，我们可以借助框架提供的一些钩子来实现
以下以vue技术栈为例，我们可以使用：

- [Vue errorHandler](https://vuejs.org/api/application.html#app-config-errorhandler)
- [Vue Router onError](https://v3.router.vuejs.org/api/#router-onerror)

总体原则是，
> 我们应该在最贴近堆栈顶层进行处理


## 错误处理器

这里引入错误处理器的概念
一般来说一个成熟的错误处理器，包含四个以上步骤
1. 转换错误，因为错误类型其实是```unknown```的，**你永远不知道一个错误是什么东西**。
  可能是字符串，可是```Error```对象，可是```undefined```。
  另外，如果遇到一些流程上认为的错误，但是系统实际没有错误的情况,
  比如 ```new DOMException('user abort', 'AbortError')```
  就直接return，不继续后续的错误流程

2. log错误，一般是转换错误后，将错误打印到控制台，方便开发人员查看。
  当然还要涉及到怎么打印方便，比如我们打印webgl错误的时候，经常会伴随shader错误。
  那么我需要有一段更好的打印方法来显示具体哪行代码错误，例如下面的代码

3. 上报错误：交给sentry或者一些日志系统进行上报，自动报障。当然，后续还有很多流程。比如根据内网sourcemap分析错误堆栈，自动录入错误工单系统，对接issue tracker等

4. 安抚用户：说白了最后一步可能就是得让用户知道到底什么情况了。
  比如 ```401 Unauthorized``` ，应该弹出登录框，或者转跳登录页。
  如果不是401或者其他特殊的错误，
  如果系统有国际化，应该在这个步骤进行文案的国际化

```typescript
// 处理一些内嵌语言的控制台显示效果
function getScriptErrorLinesLogMessage(
  script: string, errorLine: number, showLines = 5
) {
  const lines = script.split('\n');
  const lineIdx = errorLine - 1;
  // 标记控制台的 字符串 css 注入点标记 %c，注意避开前后的空白字符
  lines[lineIdx] = lines[lineIdx].replace(/(^\s*)/, '$1%c').replace(/(\s*$)/, '%c$1');

  const start = Math.max(0, errorLine - showLines);
  const end = Math.min(lines.length, errorLine + showLines);

  return lines.slice(start, end).map((line, idx) => {
    const lineNumber = idx + start + 1;
    return `${lineNumber === errorLine ? '➡️\t' : '\t'}${lineNumber}\t${line}`;
  }).join('\n');
}
```


## 修饰错误或者业务上的捕获错误

有些场景下，我们是需要修饰或者捕获业务上的错误

假如我增加一些场景的

  - 根据token获取用户信息
  - (+) 如果401，则代表用户未登录
  - 根据用户信息的 ```.vipRank``` vip等级推送广告
  - (+) 未登录用户需要额外的api获取默认推荐广告
  - 上述每个过程都需要请求api


正确的做法，我们就可以大方的使用 ```catch```，让流程落到正确的情况，因为

> 对于当前场景来说，用户没有登录也是一种正常情况

但是要注意的是，我们应该只处理 “没有登录” 这种错误，
而其他错误，我们应该继续抛出，因为我们永远不知道，运行时
我们的错误，是因为没有登录，还是因为我们写错代码导致的，或者其他情况
如果是后者，我们应该遵循 ```let it throw``` 原则，对不可控的错误，进行抛出

```typescript
// rank 改为非必填
function getUserAds(rank?: number) {
  return (await axios.get('user/recommand-ads'), { rank }).data
}


function main() {
  let info: User | null;
  try {
    info = await getUserInfo();
  } catch(err) {
    // 这里我们判断十分严谨，因为 **你永远不知道一个错误是什么东西**
    // 我们可以封装这些判断，用于区分具体的错误
    if (
      typeof err !== 'object' || !err ||
      typeof err.response !== 'object' || !err.response ||
      err.response.status !== 401
    ) {
      // 我们处理不了的东西，不要瞎处理
      throw err;  
    }
    info = null;
  }
  // 我们不一定有info
  const ads = await getUserAds(info?.vipRank);
  console.log(ads);

}

```

## 错误无处不在

其实这篇文章的标题不太准确，正确来说，错误的处理，跟promise或者await没有任何关系。
但是实在是觉得网上太多论调将这两个东西捆绑在一起。所以将这两个问题，统一一起讨论
推个极限，如果我们整个程序都是同步的，这些手段也是一致的。
所以我曾经说过 
> 我不喜欢别人将await/proimise跟错误处理捆在一起讲，这显得很菜。

## 参考
- [如何在 Swift 中进行错误处理](https://swift.gg/2016/06/12/let-it-throw/)
- [Throw What Don't Throw](https://robnapier.net/throw-what-dont-throw)

上述就是我认为应该如何正确优雅处理错误的例子，如果觉得有疑问的同学，欢迎评论区留言