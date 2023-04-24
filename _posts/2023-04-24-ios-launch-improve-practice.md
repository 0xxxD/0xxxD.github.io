---
layout: post
title: "iOS启动优化实践"
date: 2023-04-24
author: "0xxxD"
header-style: text
catalog:      true
tags: [
iOS,
启动优化
]
---

>  [抖音 iOS 启动优化实战](https://www.infoq.cn/article/tx0tcv9h6lkvknokqn7i)   对启动优化做了理论上的详细描述，BUT没有任何代码Demo开源 : (
>
>  这里我通过文中细节以及自己的测试实践来展示下demo级别的实现和原理，仅供参考。
>
>  有任何想法或者不足可以留言交流



### 监控实现

<img src="https://raw.githubusercontent.com/0xxxD/0xxxd.github.io/main/img/in-post/launch-phase.webp" style="zoom:75%;" />



这里将上图箭头中六个时机用 `forkstart`  `loadstart` `mainStart` `appDelegateFinishLaunchStart` `appDelegateFinishLaunchEnd` `launchEnd` 表示

实现细节见[Demo](https://github.com/0xxxD/LaunchTimeImproveDemo)

- 其中1到2阶段的时间节点是通过手动构造一个A的**动态库**，保证第一个+load执行的时机（Cocoapods会根据pod名升序来注入依赖顺序），注意A的MachO Type为动态链接才行，最开始其实是实验出来的（用静态链接方式会发现其他某些库的load加载会在A之前），可以通过dyld源码找到答案，这里就不展开了。
- 时间节点获取通过`CACurrentMediaTime()`，注意下进程获取的时间处理。
- iOS10以上可以使用更为高效的`clock_gettime(CLOCK_MONOTONIC, ...)`来替代


-------

### 优化方案：

##### **Process Start阶段**: 

- 尽量减少动态库数量，demo中将本地库的podspec中加入`s.static_framework = true`，当然更多的时候通过在PodFile中加入`use_frameworks! :linkage => :static`来保证源码依赖会被静态引用，这里没有使用是因为`:linkage => :static`会覆盖掉podspec的配置，我又懒得直接打包A的动态库产物来依赖了
- 减少`load`方法数量以减少pageIn和本身带来的耗时,这里可以通过在Adhoc和Debug环境下hook所有的load方法将其收集和打印来做防劣化
- 二进制重排，详细见这个[repo](https://github.com/yulingtianxia/AppOrderFiles)
- 动态库懒加载（暂没实现）
- 通过重命名段来减少TEXT段的解密时间（仅for iOS13以下）

##### pre main阶段:

- load本身执行也会带来耗时 可以通过hook load来进行耗时检测，比如 [TTAnalyzeLoadTime](https://github.com/huakucha/TTAnalyzeLoadTime)提供了具体实现
- demo中也对某些影响启动时间的三方库，（比如QCloudCOSXML中这个[issue](https://github.com/tencentyun/qcloud-sdk-ios/issues/36)）中category和class中的+load通过hook的方式，延迟到启动结束且runloop空闲时执行，这里其实可以加入更多非启动必须库的+load的白名单范围做进一步优化

##### Main阶段:

- 在main方法中一般不应该执行SDK初始化或者一些任务，特殊的防逆向可以放这



##### LifeCycle阶段:

- 在较多的依赖场景对于二方和三方的组件初始化需要针对业务类型做区分，尽可能利用异步和闲时加载来设计的一个启动管理库

##### FirstFrame:

- 保证合理的视图层级加载（尽量少的VC层级，历史需求迭代积累往往会积累较深的层级），冷启动的childVC预加载可以放到闲时进行等
- 资源的处理 icon图片放到asset
- lottie加载等耗时功能的避免等
- 应该还有不少，待补充

### TODO：

- demo中加入动态库懒加载的实现
- C++构造函数的全局耗时检测/和优化（延迟执行？）
- demo中加入任务管理项实现
- 触发了后台启动的情况下能做的启动优化有哪些？

