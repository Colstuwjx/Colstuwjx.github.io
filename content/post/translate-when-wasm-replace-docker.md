---
title: "[ 翻译 ] WASM 何时会取代 Docker "
date: 2022-06-10T08:00:00+08:00
tags: ["docker", "wasm", "translate"]
categories: ["translate"]
comments: false
showMeta: false
showAction: false
aliases:
    - /translate-when-webassembly-replaces-docker/
---

【编者的话】WASM 终有一天会替代 Docker ？事情可能没那么简单！

<!--more-->

在 KubeCon + CloudNativeCon 大会期间，其中一个挺有意思的话题吸引了众人的关注，那便是，由于其独特的设计，WebAssembly（也称为 Wasm ）在许多场景下是否可以取代 Docker。但是正如我们将在下文所看到的那样，过度关注 WebAssembly 的这个方面其实并没有真正抓住重点，因为更重要的在于 WebAssembly 可以支持哪些具体的业务，且听下文详细道来。

然而，就像任何一门有趣的新兴编程语言或者更宽泛来讲一项新的技术，真正能够考验 Wasm 价值所在的正是它的商业用途。而且看起来 Wasm 在简易性、可移植性和安全性方面的优势，使其至少可以作为一个弥补 Docker 短板的优秀替补，特别是针对边缘计算和分布式应用程序。

WebAssembly 可用于融合 [JavaScript（JS）](https://developer.mozilla.org/en-US/docs/Web/JavaScript)、[C++](https://en.wikipedia.org/wiki/C%2B%2B) 和 [Rust](https://www.rust-lang.org/)，加上 HTML 和 CSS 代码，以一个二进制的格式集成到单一的运行时平台里，该二进制格式可以直接在 CPU 的机器层面运行。它可用于支持 Web 应用程序并且可以扩展到在 CPU 上运行的任何边缘环境及云原生平台，包括服务网格和边缘 Kubernetes 的支持。在 2019 年[万维网联盟（W3C）](https://thenewstack.io/this-week-in-programming-the-time-has-come-to-pay-attention-to-webassembly/)将其定义为一项 Web 标准之前，Wasm 实际上已经存在有一段时间了，自那以后，Wasm 成为了继 HTML、CSS 和 JavaScript 之后的第四项 Web 标准。

[Adobe](https://www.adobe.com/) 高级软件工程师 [Colin Murphy](https://www.linkedin.com/in/colin-murphy-08b3601b/) 在他的演讲中详细介绍了 Adobe 的 CDN 边缘计算以及 Wasm/[WASI](https://wasi.dev/) 平台，还有一些眼下及未来的 Adobe 应用。为了提高业务效能，Murphy 提到 Wasm 如何能成为 Docker 的潜在继任者。Murphy 说，他只是“环顾四周，然后看看谁会是 Docker 和 Kubernetes 的继任者”，因此当我讲到 WebAssembly 时，我开始说，“好吧，我能应用到一个实际的线上微服务吗，然后我可以使用 WebAssembly 将它部署到边缘服务器的客户端上吗？”，继而他发现他的预感是对的。

## 使用 WASM 规避漏洞

举个例子，Docker 相关的最主要问题之一便是出现 CVE 级别漏洞的潜在可能性。“有时候，在一个 Docker 容器里，同一个缺陷可能会爆出来好几个 CVE 漏洞。使用 WebAssembly 的话，你没有任何第三方的东西。只需要把它看作是一个二进制文件”，Morphy 在一个播客采访中这样说道。“不过，当然了，这样总归还是会有安全问题。只不过，你不必带上一个操作系统的所有其余部分，也不必装作把它看成是一个完整的操作系统，因为它确实就是自己的单元”，这一说法有助于让 Wasm 变得更令人信服。

然而，Morphy 说，也不必指望 Wasm 会完全取代 Docker 。

“这个世界仍然会有大型机，仍然会有宿主机操作系统，仍然会有跑着非常特殊的业务案例的虚拟机，并且它们还将被继续沿用下去。但是 Wasm 也有一些非常好的使用场景，特别是在 5G 汽车的边缘环境，以及所有物联网与世界及边缘环境交界的这类应用程序，在这些地方你将无法再带上 Docker 。”

[Docker](https://www.docker.com/) 公司的产品负责人 [Jake Levirne](https://www.linkedin.com/in/jakelevirne) 在一封电子邮件的回复中说道，Wasm 是否有可能在某天最终完全取代 Docker 的这一问题首先是不成立的。Levirne 说，这个问题并没有正确解构开发者市场的运作方式，因为 Wasm 本质上是一项技术，它并不是要去替代 Docker 这个产品。

“ Wasm 可以作为 Docker 的补充 ———— 无论开发者如何选型其应用程序的架构和实现部分，Docker 都将在这里支持他们的开发体验”，Levirne 说。

Levirne 说，借助 Docker 的开发、测试及部署工具链，我们可以更轻松地去维护一套应用程序交付的可重现式流水线，无论应用程序采用了何种架构。此外，数百万个预构建好的 Docker 镜像，包括数以千计的官方的或是经过验证的镜像，提供了“一套核心服务的骨干（例如数据存储、缓存、搜索、框架等）”，它们可以和 Wasm 模块配套使用。

“随着时间的推移，容器运行时和注册表将会扩展到囊括原生的 Wasm 模块支持。事实上，这在今天已经发生了，”Levirne 说。

**原文链接：[when-webassembly-replaces-docker](https://thenewstack.io/when-webassembly-replaces-docker/)**
