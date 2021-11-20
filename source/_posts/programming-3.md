---
title: 重学编程(3)-注册登录
date: 2021-09-01 23:12:22
tags:
- 重学编程系列
categories:
- 编程
- 游戏
---

>你好！
我是千无，接下来的日子里，我会不定时更新[重学编程系列](https://miyoosan.github.io/article/tags/重学编程系列/)的文章。
这是该系列的第3篇：[重学编程(3)-注册登录](https://miyoosan.github.io/article/2021/09/01/programming-3/)。

接上篇，今天我们要完成一个简单的游戏客户端与服务端，以支持玩家进行注册登录。

在正式写代码之前，有两个工作需要我们处理好，它们分别是：需求设计、技术实现。

## 需求设计

如果你仔细回顾过第一篇贴出的[游戏设计稿(点击查看)](https://miyoosan.github.io/article/2021/09/01/programming-1/#策划游戏)，就会发现里面其实没有`注册登录`这个界面。

按照一般游戏的惯例，比如王者荣耀的，闪屏之后应该是注册登录，然后才是游戏内容。

这里，我简单设计一下登录界面：

![注册登录](../../../../images/imgs-3/register.jpg#w360)

然后更新角色选择：

![选择角色](../../../../images/imgs-3/role.jpg#w360)

注意到，登录界面里，如果是新玩家，我们会给他生成随机账号密码，这样一些懒玩家就可以一键登录了，有心的玩家可以自己输入账号密码。

玩家第一次登录之后，可以创建角色，如果已经创建过角色，那就直接进入角色主界面。

我们来画一张详细点的业务流程图：

![注册登录流程，鉴权采用JWT](../../../../images/imgs-3/login.png)

## 技术实现

厘清了需求，我们来考虑一下具体的技术方案。

`注册登录`功能其实比较简单，随便一个web应用都能满足它的需求，但我们要做的是一款游戏，那就不得不把后续的其他功能也考虑进来了，比如`闯荡江湖`、`闭关修炼`。

我们在闭关修炼时，可以随时停止修炼，也能查看背包，角色属性，甚至与其他人聊天互动等等，所有这些操作，都有即时反馈，界面会发生变化，特别是闯荡江湖与NPC角色或玩家角色切磋的时候，他一刀我一剑，有来有回，顺序不会乱，伤害正确计算。这些都要求所有角色的时间都是同步的，因为他们都在同一个时空里。

要怎么才能做到这样呢？

<font color="OliveDrab">玩家登录后，一旦创建了角色，其实就应该已经进入了游戏世界了，在这个虚拟的世界里，时间是一直在流逝的，哪怕玩家退出了登录，游戏世界也在时刻变化着。</font>

关键点在于：`玩家做出操作后，他在游戏世界里的角色，接收到了操作指令，去做出相应的动作(比如攻击)，周围的角色(比如某个NPC)能看到这个动作，并作出反应(比如逃跑)。`

这就要求游戏世界里的一切都按一定的时序变化，由于计算机的离散特性，时序并不是连续的，反映到屏幕画面上，是一帧一帧画面的刷新，当刷新速度足够快时，利用视觉残留原理就可以达到看起来是连续变化的效果。同时，游戏世界要能处理来自外界的信息，并及时更新。

可以用代码简单描述一下游戏世界的主逻辑：

```js
// 无限循环，代表时间永远在流逝：一次循环代表一个时刻
while(true) {
  processInput(); // 每一刻，游戏世界都查看并处理来自外界的信息：比如玩家点击施展剑法、拾取秘籍
  update(); // 及时更新游戏世界里的一切，比如内力减少，经验增加
  render(); // 在屏幕上绘制玩家在这一刻能看到的画面
}
```

很显然，登录逻辑不会在无限循环里，一个玩家的登录只不过是在游戏世界里又开辟了一条连接外界的通道，通俗点说，就是玩家无法直接降临游戏世界，只能让意念通过这条通道给游戏世界里的傀儡(角色)下达行动命令。

这样就很容易理解了，玩家退出登录，就只是关闭这条通道而已，不会影响游戏世界的持续衍变。

### 游戏服务端

我们现在可以着手服务端的概要设计了。

实际上，玩家登录后通常不会直接在游戏世界里开辟通道，因为他都不一定创建有游戏角色。因此，我们要求玩家先创建角色，才能申请去开辟通道。

而从游戏世界的角度来看，通道也不应该由玩家来开辟，我认为，只有游戏世界的管理者才有资格开辟，玩家角色仅仅有权利去申请使用它。

对于一些恶意账号，我们会想封禁他的角色甚至账号，这种情况下，只需要让它的角色失去这种权利即可，这种权利，可以叫做世界通行证，有这张通行证，你就能申请连接到游戏世界，意念降临！

综上，相应的业务逻辑已经很清晰了，我们用一段伪代码来描述这个过程：

```js
// 服务端处理登录逻辑
app.on('login', request => {
  // 1. 玩家请求登录时，检测他的账号，有效就返回他的真实账号；无效会抛错，让玩家输入正确账号密码
  let account = app.checkUserAccount(request.name, request.password, request.token);
  // 2. 然后找到他的游戏角色ID; 如果找不到，就抛错，让玩家先去创建角色
  let roleId = app.findRoleId(account);
  // 3. 接着给角色申请一张世界通行证，有了它，就有资格去申请世界通道了；(被封禁的账号角色进行申请时会失败)
  let pass = app.applyForPass(roleId);
  // 4. 申请一条连接游戏世界的通道; 如果申请失败(可能服务器容纳人数已满，需要排队)，就抛错，让玩家等待或者重试
  let channelId = world.applyForChannel(pass);
  // 5. 最后，激活(启用)这条通道, 属于玩家的游戏，正式开始···
  world.activateChannel(channelId);
})
```

```js
// 服务端处理退出登录逻辑
app.on('logout', request => {
  // 1. 玩家退出登录时，找到他的真实账号
  let account = app.findUserAccount(request.token);
  // 2. 然后找到他的游戏角色ID; 如果找不到，直接退出
  let roleId = app.findRoleId(account);
  // 3. 取出玩家的通行证；如果没有，直接退出
  let pass = app.findPass(roleId);
  // 4. 根据通行证找到玩家连接的通道；如果没找到，直接退出
  let channelId = world.findChannel(pass);
  // 5. 废弃(停用)这条游戏世界通道
  world.inactivateChannel(channelId);
})
```

需要注意的是，玩家退出登录时，我们在具体实现里，要把他的凭证token也废弃掉，这样就可以让玩家在重进游戏时必须重新登录了。

最后，我们组织一下整个游戏服务端的入口流程：

```js
// 创建服务端应用
const app = new App();
// 创建游戏
const game = new Game();
// 创建虚拟世界
const world = new World();
// 创建网络服务
const server = new Server();
// 创建消息管道
const messager = new Messager();
// 创建通信规范
const specification = new Specification();

// 让应用、游戏、虚拟世界、网络，通过消息管道互联互通，并遵循指定的通信规范
connect(app, game, world, server).by(messager).following(specification.msg);
// 启动虚拟世界，开启时间的无限循环
start(world);
```

以上是伪代码，具体的实现我们留待后面去做，可能会有些细微调整，但不影响大局。

### 游戏客户端

接下来是客户端。

客户端要做的事情很简单，展示一个登录表单，调用一下服务端提供的接口就可以了，稍微麻烦点的是，想把页面背景样式调整好要花费一番功夫，高兴的是，这个细节我们暂时不用理会。

`为什么呢？因为先学会了走路，再尝试去奔跑，这很重要。`

`重要到我特意用两行字来强调它：先保证功能正确，程序能跑通，再考虑完善细节和性能优化。`

话说回来，由于我们客户端程序是运行在浏览器里的，因此使用HTML和JS就能完成注册登录任务。

我们用代码描述一下这个业务流程：

```html
<body>
  <form id="loginForm">
    <input type="text" name="name" value="随机账号" />
    <input type="password" name="password" value="123456" />
    <input type="submit" value="登录武林" onclick="login(event)" />
  </form>
  <section>
    <p>健康游戏忠告</p>
    <p>抵制不良游戏，拒绝盗版游戏。</p>
    <p>注意自我保护，谨防受骗上当。</p>
    <p>适度游戏益脑，沉迷游戏伤身。</p>
    <p>合理安排时间，享受健康生活。</p>
  </section>
</body>
```
```js
// 登录流程
function login(event) {
  event && event.preventDefault(); // 表单提交后浏览器会刷新页面，体验糟糕，我们不让它刷
  const form = document.querySelector('#loginForm');
  const message = {
    type: 'login', // 这个操作指令告诉服务端，我们想登录
    data: {
      name: form.name.value, // 账号
      password: form.password.value, // 密码
      token: localStorage.token,  // 凭证
    }
  };
  // 发送登录请求
  websocket.send(JSON.stringify(message));
}

// 启动Websocket服务
let websocket = new Websocket('ws://localhost:8081');

// 监听服务端返回的消息
websocket.onmessage = function(event) {
  switch(event.data.type) { // 根据不同业务类型，做不同处理
    case 'login':  // 处理登录
      if (event.data.success) return alert('登录成功');
      localStorage.token = ''; // 登录失败时，这里清空token
      alert(event.data.error); // 提示失败信息
      break;
    // ··· 其他业务逻辑，比如登出、创建角色等，略
  }
}
if (localStorage.token) login(); // 如果本地存有凭证，我们直接去登录
```
除了注册登录，我们的游戏客户端其实还有30+的界面，也可能会更多一些，无论如何，我们在开发中再挨个补上就好了。

客户端程序编写完后，我们要为它提供静态文件服务，使得玩家们(`我们自己`)可以通过网络访问它，从而玩到这个游戏。

成熟的解决方案是通过nginx配合CDN来做这件事情，我们本地开发时，不必考虑如此周全，简单的用node起个文件服务就好，如果你跟我一样懒，可以下载这个[简单的web程序](https://github.com/rsdoiel/ws/releases/download/v0.0.8/ws-binary-release.zip)，解压后，linux环境通过命令`ws path/to/dirname`就能为你指定的目录提供文件服务。

## 编写程序

最后的最后，一切准备就绪！

我在github上建了一个[重学编程的代码仓库](https://github.com/VirtualMilky/swordsman)，用来保存和演示这一系列文章的示例，里面的具体实现可能会与文章里描述的有些许出入。

这是正常的，因为这就是程序开发的真实模样。

`跟画画一样，写代码也是一笔一笔的添上去，中途可能重绘甚至重新换N张画纸，直到真正完成作品。`

请务必保持耐心，细心，还有：好奇心。

本篇的登录注册客户端与服务端，我已完成代码实现，并已上传，接下来，我们将要挑战`游戏角色的创建`这个重要任务。

让我们下一篇见！
