Laravel 深入浅出指南
---
Laravel 5.7 源代码解析，新手进阶指南。 [在线版](https://git.io/Laravel)

## 写在前面

#### Q. 本书所讲述用到的 Laravel 框架是哪个版本的？
> [v5.7.0](https://github.com/laravel/laravel/releases/tag/v5.7.0)

#### Q. 为什么要用 Github Issue 来写解析文章?
> 这问题要分两步来回答：1 是为什么用 Github，2 是为什么用 Github Issue。

> 首先 “为什么用 `Github`”，这是因为 `Github` 是当今最活跃的开源社区，我们写的文章，可以很方便的跟大家在上面讨论交流观点，还有就是错误的修正校对比较容易。

> 然后 “为什么用 `Github Issue`”，这是因为 `Github Issue` 非常适合做[代码的引用](https://blog.github.com/2017-08-15-introducing-embedded-code-snippets/)；毕竟是技术文章嘛，引入代码来源是必不可少的。为了解决在 issue 引用不了跨项目的代码，我甚至把 [`vendor/`](https://github.com/xiaohuilam/laravel/tree/master/vendor) 都提交上来了。

#### Q. 内容覆盖面规划安排？
> 初步规划是从 HTTP 服务入口入手，开始 bootstrap 到注册启动服务提供者，到管道，到中间件，到路由分发为主干；然后后面在讲解容器的特性、实现及方法，和几个重要的服务提供者的代码分析，以及 Laravel 比较重要的设计的分析（如 Facade 类和 Macroable 实现等）。预计阅读后，能加深刚接触 Laravel 的开发者对其的理解，甚至能独立开发包含服务提供者扩展的能力提升。

#### Q. 文章的作者、版权及转载授权？
> 本电子书籍作者有 [蓝剑波](https://github.com/xiaohuilam/) 和 [陈远](https://github.com/cran208)，这是我们第一次尝试将知识输出成为电子书籍，我们虽都是使用 Laravel 超过3年的开发者，但因为精力和个人认知等原因难免纰漏，希望大家在查阅的同时留意我们的笔误、逻辑错误等疏忽并指正。
 
> 在校对完成前，本电子书暂不出版纸质书。 
 
> 本项目使用「署名 4.0 国际」创作共享协议，声明来源和作者信息的原则下即可转载。来源是一定要带的，毕竟文档可能有错误校对，如果拷了内容而不负责标注来源，传播了错误的知识那就是误人子弟了。 
