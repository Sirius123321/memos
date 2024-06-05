# Memos
Memos that support collaborative editing by multiple people.
以支持多人协同编辑功能为主的备忘录

## 后端(Backend)
- Springboot3: 后端主框架，约定大于配置，大幅度减少手动编辑的XML配置文件
- Spring Webflux: 反应式(Reactive)Web框架，利用线程池和Reactor库实现反应式编程模型，以提高线程利用率的形式提高性能
- Neo4J: 图形数据库，以有向图形式维护实体(节点,Node)间的关系，同时支持事务
- Redis: 键值对数据库，在本项目主要用于实现简单的消息队列及维护文档正在编辑客户端列表

## 前端(Frontend)
- Vue3: 前端主框架
- Yjs: 多人协同，以CRDT形式实现，比起OT算法实现难度更低，但维持的元数据(metadata)更多
- Tiptap: 富文本编辑，ProseMirror的支持与Yjs数据绑定的Vue组件形式的可定制化富文本编辑器实现

## 数据流(关键逻辑)
1. 协同编辑
    ```
    每次编辑(缓存)：客户端(浏览器) -(yjs状态更新增量包)-> Websocket端点 -(读取Redis维护的客户端并广播)-> 另一个客户端
    定期操作数后(事务化的持久化内容到数据库)： 客户端-> -(yjs的uint类型包含元数据快照)-> 内容保存端点 -(neo4j保存)->Neo4j数据库
    ```
因为内容变更是一个非常频繁的事件，所以使用Redis的消息队列缓存每次内容变更时的状态更新增量包，并在客户端设计在一定次数的编辑后生成快照保存到数据库，在这之后服务器会清空缓存。这个设计也为以后扩展用户返回栈提供可行性。
2. 用户登录
因为项目是前后端分离的，后端的API设计基于`RESTful思想`，使用名词`URI`并以`GET/POST/PUT/DELETE`区分操作语义，所以使用`JWT token`作为身份验证方案，具体实现为在HTTP Header中以`Authorization: Bearer token`形式存在。
而登录基于![RFC7617](https://datatracker.ietf.org/doc/html/rfc7617)，使用`Authorization: Basic base64(username:password)`这一`HTTP Basic`标准登录

## 关键技术
1. CRDT (协同编辑的根本)
CRDT（Conflict-free Replicated Data Type，冲突自由复制数据类型）是一类数据结构，设计用于在分布式系统中处理数据复制和并发更新问题，而无需复杂的冲突解决机制(复杂的解决机制: OT算法)。CRDT 保证在最终一致性模型下，各副本的数据一致性。CRDT的基本原理是通过特定的算法和数据结构设计，使得所有副本都能独立处理更新操作，并且通过有限次数的状态同步达到一致性。CRDT 分为两大类：状态型（state-based）和操作型（operation-based）。本次项目主要使用的是基于状态的 Yjs 实现，它使用 Y.Doc 这一文档结构表示整个协作文档，Y.Doc 内部包含多种数据类型，如 Y.Array、Y.Map 和 Y.Text，这些数据类型都是 CRDT，可以独立维护其一致性。Yjs JavaScript 库实现的CRDT支持增量更新和同步，在 Web 应用方面很有优势，并且由于 CRDT 本身的设计使其能够在合并操作时自动处理冲突，保证所有副本的最终一致性。
2. Spring Webflux
Spring WebFlux 是 Spring 框架中的一个模块，用于构建异步、非阻塞、事件驱动的 Web 应用程序。它是对 Spring MVC 模块的一种补充和替代，特别适用于需要高吞吐量和低延迟的应用场景。Spring WebFlux 基于反应式编程模型，使用 Project Reactor 库。反应式编程允许开发者处理异步数据流和处理背压。WebFlux 遵循 Reactive Streams 规范，这是一组用于处理异步流的标准，更有利于协同编辑这种更重视延迟，需要频繁大量请求的 Web 应用。
3. Redis与Websocket
WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议，使客户端和服务器之间能够低延迟地交换数据。Redis 则是一个高性能的内存数据库，常用于缓存和消息队列。结合使用 WebSocket 和 Redis，可以实现高效的实时数据处理和存储。具体来说，WebSocket 接收到高频数据后，可以立即将这些数据推送到 Redis 队列中。Redis 的高吞吐量和低延迟特性，使得它能够迅速处理和存储这些实时数据。这样，在协同编辑应用中，WebSocket 能够保证数据的实时传输，而 Redis 则确保数据的快速存储和可用性。这种组合不仅提升了系统的响应速度，还简化了数据的管理和处理逻辑，适用于高并发的场景。
4. Neo4j图数据库
图数据库是一种 NoSQL 数据库，使用有向图结构存储和查询数据，它更重视节点 (Node) 和关系 (Relationship)，并且节点和关系都有携带的键值对属性 (Properties)，可以更好地处理复杂情况下的关系操作。本次项目将把备忘录、用户都构建为 Node 形式，以键值对存储 id、content 等内容，并使用 关系 连接及区分备忘录的 拥有、分享、协作 等类型的关系。Neo4j 虽然也支持 ACID 特性，但其优势在于复杂关系中的性能表现。在处理涉及多层次关系和实时查询的场景中，Neo4j 能够显著提升数据查询和分析的效率。

## 关键技术总结
总结

通过引入 CRDT、Spring WebFlux、Redis 与 WebSocket、Neo4j 等关键技术，本次项目旨在构建一个高效、低延迟且具备强大数据处理能力的协同编辑应用。这些技术的结合将确保系统在高并发和复杂关系处理的场景下依然能够保持良好的性能和稳定性。
