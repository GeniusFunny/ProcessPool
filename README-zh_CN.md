![](https://i.loli.net/2019/03/24/5c973b70e65b3.png)
![](https://img.shields.io/npm/v/node-process-pool.svg)![](https://img.shields.io/npm/dw/node-process-pool.svg)![](https://img.shields.io/bundlephobia/min/node-process-pool.svg)![](https://img.shields.io/npm/l/node-process-pool.svg)

[English](./README.md) | 简体中文

#### 🖥 背景

Node是单线程模型，当需要执行多个独立且耗时任务的时候，只能通过child_process来分发任务，提高处理速度；不像Java这种多线程语言，可以通过线程来解决并行问题，Node只能创建进程来进行处理；但是进程相对于线程来说，开销太大。一旦进程数较多时，CPU和内存消耗严重（影响我干其他的事情），所以做了一个简易版的进程池，用来解决并行任务的处理。

适用场景：相同且独立且耗时的任务，例如，拿到某网站1000个用户的账号密码，我现在想要他们的信息，爬他，node-process-pool非常适合。

#### 🤔思路

主控进程+工作进程群

ProcessPool是我们管理进程的地方，我们通过传递配置参数（任务脚本、脚本需要的参数、最大并行进程数）生成一个ProcessPool实例，然后通过这个实例来管控进程池。

ProcessItem是我们进程池里的进程对象，ProcessItem对象除了process的信息，我们还增加了唯一标识和状态（忙碌、任务失败、任务完成、进程不可用）。

一批任务开始时，我们会一次性fork到最大并行进程数，然后开始监控是否有工作进程完成任务，如果有工作进程完成了任务，那我们就可以复用这个工作进程，让其执行新任务；如果任务执行失败，我们会将任务归还给进程池，等待下一次分发。

**由于是相同且独立且耗时的任务，所以当某个工作进程完成任务时，我们很有必要去检测所有的工作进程是否已完成任务，而不只是复用这个工作进程，我们要一批一批的复用！！！**

**因为差不多的时间开始执行相同的任务，当一个工作进程完成时，完全可以相信其他工作进程也完成了任务，所以检测一轮所有的工作进程，若空闲，给他们分配新任务。**

**既然是批量分配任务，就不会存在只有某个工作进程在辛苦的运行，其他工作进程袖手旁，哈哈哈哈哈，总得雨露均沾嘛。**

~~由于主控进程即要负责IPC又要不断监听批任务完成的情况，目前我采用的方式是setInterval切割，让IPC和监控能交替进行（ps：应该有更好的方法~~

**我们真的需要setInterval来去轮询任务状态吗，什么时候才需要轮询任务状态然后调度？**
工作进程状态发生改变的时候，才是我们需要去检测任务状态和调度的时机；所以，我们也可以利用IPC来通知主控进程进行检测任务状态和调度。
**ps：当然，还有更好的方法，嘿嘿**

## ✨ Features

- 快

  某大学某系统，12000个用户，每个用户登陆需要两次API访问，两次API访问直接必须有0.5s间隔，然后将信息写入文本。

  单线程(未测试，理论上)：12000*0.5 —> **至少需要6000s**。

  进程池(已测试，容量为50的进程池)：**206s**，平均每个任务耗时17.1ms

  **效率提升：接近30倍**

## 📦 安装

```bash
npm install node-process-pool
```

## 🔨使用

```js
// 进程池使用示例
const ProcessPool = require('../src/ProcessPool')
const taskParams = []
for (let i = 0; i < 100; i++) {
  taskParams[i] = [i]
}
// 创建进程池实例
const processPool = new ProcessPool({
  maxParallelProcess: 50, // 支持最大进程并行数
  timeToClose: 60 * 1000, // 单个任务被执行最大时长
  dependency: `const path = require('path')`, // 任务脚本依赖
  workDir: __dirname, // 当前目录
  taskName: 'test', // 任务脚本名称
  script: async function task(workParam) {
    console.log(workParam)
  }, // 任务脚本内容
  taskParams // 需要执行的任务参数列表，二维数组
})
// 利用进程池进行处理大规模任务
processPool.run()
```

## 🤝贡献 [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)

I welcome all contributions. You can submit any ideas as [pull requests](https://github.com/geniusfunny/node-process-pool/pulls) or as [GitHub issues](https://github.com/geniusfunny/node-process/issues). If you'd like to improve code, please create a Pull Request.
