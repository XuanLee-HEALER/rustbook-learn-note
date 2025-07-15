# axum

## Extractors（提取器）

实现了`FromRequest`或`FromRequestParts`特质的类型，就是一个Extractor

* `FromRequestParts<S>` 这种提取器只能访问请求的元数据部分，比如请求头、URL、请求方法，以及通过中间件添加的扩展数据等。它不能访问请求体。因为不涉及请求体，所以它可以被多次使用在一个handler中
* `FromRequest` 这种提取器可以访问请求的全部内容。因为它可能会消耗请求体，通常在一个handler中，只能有一个会消耗请求体的提取器

extractor是将原始的、完整的HTTP请求，拆解成handler需要的、结构化的、强类型的数据。调用handler时，会检查函数参数，调用不同参数的extractor的内部逻辑来处理请求。如果所有extractor都成功了，它们的结果就会被作为参数传入函数，然后执行业务逻辑，如果任何一个失败了，请求会提前中断，并返回一个响应的客户端错误

### Sharing state with handlers（在handler中共享状态）

#### 使用`State`提取器

工作原理：在应用启动时创建一个全局状态（结构体），通过`.with_state()`将其附加到`Router`上，Axum会将这个状态的共享引用（`Arc`）保存起来。在handler中，通过`State<T>`提取器声明这个状态

优点

* 绝对类型安全：编译器编译时检查一切，如果handler中类型声明错误就无法通过编译
* 明确的依赖关系：handler的函数签名直接声明了它依赖的共享状态
* 高性能：克隆`Arc`的性能代价很低

缺点

* 状态的类型和内容在编译时已经确定。无法在请求处理的流程中动态为不同的路由添加不同类型的共享状态。整个应用共享的是同一个“状态集合”

适用场景：几乎所有应用级、生命周期与整个应用相同的共享资源。例如数据库连接池、HTTP客户端、应用配置、模版引擎

It is common to share some state between handlers. For example, a pool of database connections or clients to other services may need to be shared.

The four most common ways of doing that are:

Using the State extractor
Using request extensions
Using closure captures
Using task-local variables

You should prefer using State if possible since it’s more type safe. The downside is that it’s less dynamic than task-local variables and request extensions.

The downside to this approach is that you’ll get runtime errors (specifically a 500 Internal Server Error response) if you try and extract an extension that doesn’t exist, perhaps because you forgot to add the middleware or because you’re extracting the wrong type.

The downside to this approach is that it’s a the most verbose approach.

The main downside to this approach is that it only works when the async executor being used has the concept of task-local variables. The example above uses tokio’s task_local macro. smol does not yet offer equivalent functionality at the time of writing (see this GitHub issue).
