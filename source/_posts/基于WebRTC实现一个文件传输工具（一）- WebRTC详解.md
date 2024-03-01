---
title: "基于 WebRTC 实现一个文件传输工具（一）- WebRTC 详解"
description: "WebRTC介绍"
date: 2024-02-05
tags:
  - WebRTC
categories:
  - hyper-transfer 文章合集
cover: https://img.cczywyc.com/post-cover/hyper_transfer_1_webrtc.jpeg
---

如题，最近在折腾一个文件传输工具，用于在多个不同设备间方便快速地传输文件。我的要求也很直接，这个文件传输工具，最好不需要繁琐的安装和配置过程，能够做到开箱即用，同时它还需要兼容我的 PC 设备和多个移动设备，经过一番调研之后，我决定把技术方案定为了 WebRTC。

本篇文章是此系列文章的第一篇，详细介绍 WebRTC 技术。

# WebRTC 概述

WebRTC（Web Real-Time Communication）是一个开源项目，旨在通过简单的应用程序接口（API）实现浏览器和移动应用之间的实时通信（RTC）。它允许网页浏览器进行语音通话、视频聊天和点对点文件分享，而无需用户安装特殊的插件或第三方软件。

简单来说，WebRTC 项目主要思想是定义 WebRTC API，该 API 允许浏览器安全地访问设备上的输入外设，如麦克风和网络摄像头，以点对点的方式与远程设备共享或交换媒体数据以及实时数据。借助 WebRTC，多个不同设备可以在一个平台上流畅地、安全地共享语音、视频和实时数据。

WebRTC 是 Google 在 2011 开源的项目，并将其纳入 Chrome 浏览器的开发计划，经过多年持续的发展和壮大，它已经成为浏览器间实时通信的主流技术之一，现在，几乎所有主流的浏览器都支持 WebRTC 标准，并且在各个领域得到了广泛的应用和推广。有关 WebRTC 的历史可以参考[这篇文章](https://princiya777.wordpress.com/2017/08/06/webrtc-a-detailed-history/)。

WebRTC 是实时通信的未来，它受欢迎的原因有很多，我总结了 WebRTC 以下几个特点：

* WebRTC 是一种无需插件的现代实时通信技术。它不需要任何额外的插件或应用程序来进行音频、视频流和数据共享，它使用 JavaScript、应用程序编程接口（API）和 HTML5 将通信技术嵌入到浏览器中。像 Google Hangouts、Whatsapp、Facebook Messenger、ZOOM 团队通信、Zendesk 客户支持、Skype for Web 等产品都集成了 WebRTC。
* 不同设备的浏览器能够以对等的方式直接与其他浏览器交换实时媒体。
* WebRTC 提供了比其他流媒体更高的安全性，并且无需借助第三方软件。
* 它是免费开源的，并且在全球范围内应用，这正是推动此项技术发展的动力。
* WebRTC 支持多种编程语言的客户端 API。WebRTC 标准受到多种编程语言和框架的支持，例如：JavaScript、Java(Android)、NodeJs Rest、NodeJs WebSocket、Go、PHP、Python、React Native、REST、Ruby、Swift(IOS)。

# 核心组件和架构设计

## 三个核心组件

WebRTC 是通过浏览器进行的实时通信，通过标准化的 WebRTC API 与 Web 浏览器进行工作和通信，因此 WebRTC API 必须提供一系列的实用工具，其中包括一些类似于连接管理（以点对点方式）、编解码功能协商、选择和控制、媒体控制、安全和防火墙等。WebRTC 的功能大体可以分为以下三个组件。

### MediaStream

媒体捕获和流处理组件，MediaStream 允许浏览器访问设备的摄像头和麦克风，并捕获音视频流。

这是实现实时通信的第一步：获取用户想要分享的数据。在这个阶段下，可能会发生以下情况：

1. 用户期望的流数据（音频/视频/其他数据）和即将要建立的通信模式被捕获；
2. 即将分享的本地媒体流允许浏览器访问流设备，如摄像头、麦克风等；
3. 允许浏览器捕获媒体。

以上操作均是通过 getUserMedia() 方法获取访问权限，具体 API 操作参考[这篇文章](https://webrtc.github.io/samples/)。

### RTCPeerConnection

网络通信组件，它是 WebRTC 中最核心的组件，负责在浏览器之间建立、维护和管理直接的音视频或数据通信连接。

实时通信的第二步：确定了通信流之后，下一步就是将不同的设备连接起来，这涉及到网络通信组件，通信双方通过 STUN 和 TURN 服务允许发送者和接受者之间建立点对点连接。具体连接过程将在后面详细说明。

此阶段具体 API 操作参考[这篇文章](https://webrtc.github.io/samples/)。

### RTCDataChannel

数据通信组件，允许网页应用直接在用户浏览器之间建立安全的双向数据通道，用于传输任何类型的数据。

实时通信的最后一步：数据通信和交换，发挥作用的是数据通信组件，具体来说就是不同设备通过浏览器直接点对点交换数据。具体发生在当第一次在一个实例化的 PeerConnection 对象上调用 CreateDataChannel() 方法时。

此阶段具体 API 操作参考[这篇文章](https://webrtc.github.io/samples/)。

## WebRTC 架构设计

以下是源自[官网](https://webrtc.github.io/webrtc-org/architecture/)的 WebRTC 结构设计图![](https://img.cczywyc.com/webrtc_architecture.png)

可以看出，WebRTC 的设计相当复杂，如果你是浏览器开发者，你应该关注 WebRTC C++ API 和可以使用的捕获/渲染钩子；如果你是 Web 程序开发者，你应该更多关注 Web API 部分。

# 关键技术

## 信令

前面说到 WebRTC 是一种点对点的实时通信技术，不同设备的客户端浏览器可以直接通过 WebRTC 提供的 Web API 建立对等点对点连接，信息的交换与分享不需要借助中间服务器。但是在建立连接之前，客户端之间是如何知晓彼此的连接信息呢？多个客户端之间的实时通信会话又是如何维护的呢？这就需要借助信令服务器了。

简单说来，信令服务器就是在连接建立之前负责在通信双方传递信令数据来建立和维护实时通信会话，信令数据包括但不限于传递会话初始化信息、客户端信息、媒体元信息以及网络配置信息等。客户端通过信令服务器交换和协商上述信息，从而建立点对点的连接，需要说明的是，连接建立以后，客户端之间的实时通信和数据交换是不通过信令服务器的，它仅仅是在会话建立前发挥作用，以及维护多个会话状态等。

WebRTC 客户端实现实时通信的第一步就是连接信令服务器，需要特别说明的是，WebRTC 标准自身不规定任何特定的信令协议，开发者可以选择 WebSocket、SIP 或者 HTTPS 等其他协议实现会话控制、媒体元数据交换、网络配置信息交换等功能，所以信令服务器在 WebRTC 技术框架中是极其重要且实现方式多种多样的，保证了技术的灵活性和开放性。

## NAT 穿透

上面说到，WebRTC 客户端之间建立连接是通过信令服务器来交换网络连接信息的，这一般涉及客户端的 ip 地址和端口信息，这里涉及到两种情况：

### WebRTC 客户端处在同一个局域网内

这种情况比较好处理，只需要通过信令服务器交换客户端彼此的局域网地址，通过 WebRTC 提供的 Web API 即可建立连接，实现点对点通信。

### WebRTC 客户端不在同一个局域网内

这种情况更符合大多数的情况，因为现如今互联网上大多数设备都在分配私有 IP 地址的 NAT 后面，并且一般限制了来自外部的访问，这种情况下因为不在同一个局域网内，直接交换客户端的私有 IP 地址就行不通了，这就需要 NAT 穿透技术。

因此，NAT 穿透技术在 WebRTC 技术体系内是必要且至关重要的。

### WebRTC 使用多种技术来实现 NAT 穿透

#### STUN（Session Traversal Utilities for NAT）

* STUN 是一种协议，是基于 UDP 的完整的穿透 NAT 的解决方案，属于打洞技术，它允许 NAT 后面的客户端发现其公共 IP 地址以及它们所在的 NAT 类型
* 客户端向 STUN 服务器发送请求，STUN 服务器以客户端的公网 IP 地址和端口号响应
* 如果两个对等连接的客户端都位于允许直接连接的 NAT 类型后面，则该信息将用于建立直接连接

#### TURN（Traversal Using Relays around NAT）

* TURN 是一种数据传输协议，通过 Relay 方式穿越 NAT，TURN 主要用在 STUN 无法穿透的场景下，例如一方或者双方位于对称 NAT 后的情况
* TURN 服务器在对等点之间中继流量，有效绕过 NAT
* 此方法会引入额外的延迟和带宽成本，因为所有流量必须经过 TURN 服务器

#### ICE（Interactive Connectivity Establishment）

* ICE 是一个结合 STUN 和 TURN 建立连接的框架
* ICE 尝试多种方法来找到连接的最佳路径，它使用 STUN 进行打洞，若失败，则使用 TURN 进行中转



# 通信过程

## 局域网通信

WebRTC 客户端都位于同一个局域网内，这种情况属于比较简单的一种通信模式，借助信令服务器，客户端连接信令服务器时，会携带客户端在局域网内的 IP 地址和端口信息，此信息可用于直接建立点对点连接。

通信模型如下：![](https://img.cczywyc.com/simple_webrtc_connect.png)

## 广域网通信

这种模式涉及的场景就复杂了，在这种通信模式下，不同的客户端处于不同的网络环境，客户端前面的 NAT 也有不同的部署方式，所以存在不同类型的 NAT，而穿透不同类型的 NAT 所使用的方式和技术成本是不一样的，在本系列的第三篇文章里，我会专门详细介绍 NAT 穿透技术，这里不作展开。

通信模型如下：![](https://img.cczywyc.com/how_webrtc_works.png)



# 安全性

WebRTC 的安全性这里拿出来单独说。

一直以来，人们都希望自己的通信和交流处在安全且私密的环境中，安全性方面，WebRTC 也做了大量的考虑和设计。

* 加密：所有 WebRTC 组件（包括信令）都必须加密
* 安全协议：WebRTC 强制使用安全的实时传输协议（SRTP）来传输音视频数据，在数据通道上使用数据加密传输协议（DTLS）确保通信安全
* 浏览器保证：当使用 API 获取设备的摄像头、麦克风权限时，浏览器会有明确的提示，且必须经过用户同意后才能开启

附图：WebRTC 的协议栈和加密通信协议示意图

![](https://img.cczywyc.com/webrtc_protocol_stack.png)

![](https://img.cczywyc.com/webrtc_security_protocol.png)

# 应用场景

现如今，WebRTC 的应用场景极为丰富，包括但不限于：

* 视频会议和语音通话：提供低延迟、高质量的实时视频和音频通信体验
* 实时数据共享：支持文本、文件、屏幕共享等多种形式的实时数据共享
* 直播和点播：支持实时直播与点播服务，为用户提供丰富的媒体体验
* 远程协作和教育：在线教育、远程会议和协作工具等应用中提供实时互动能力



## Reference

https://www.geeksforgeeks.org/introduction-to-webrtc/

https://web.dev/articles/webrtc-basics?hl=zh-cn

https://webrtc.github.io/samples/

https://bloggeek.me/how-webrtc-works/

https://medium.com/callstatsio/what-are-stun-and-turn-in-computer-networking-95d5130597c7

https://medium.com/dvt-engineering/introduction-to-webrtc-cad0c6900b8e

https://blog.jianchihu.net/webrtc-av-transport-basis-nat-traversal.html

https://webrtc-security.github.io/



（全文完）
