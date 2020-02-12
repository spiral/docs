# 框架 - 设计理念

Spiral 框架的各个组件尽可能遵循实用（KISS）设计原则。基本上来说，这意味着每个组件都必须遵循以下列出的规则：

- 尽可能避免交叉依赖
- 优先采用组合而不是继承
- 偏好更小的而不是更复杂的接口
- 避免使用魔术方法（magic）

## 混合运行时

Spiral 框架依赖应用服务器来提供它的部分服务。PHP 代码主要负责高效开发业务逻辑和快速交付，而基于 Golang 的应用服务器则专注于高效地解决底层的基础服务。

> Spiral 应用服务器是一个定制化版本的 [RoadRunner](https://roadrunner.dev).

![顶层架构图](https://user-images.githubusercontent.com/796136/64451724-762d0800-d0ed-11e9-8c34-9c054a7bb0bd.png)

> 请参阅应用生命周期的[说明](/basic/workers.md).
