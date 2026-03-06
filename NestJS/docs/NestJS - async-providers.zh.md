# Async providers（NestJS 异步提供者）

来源：[NestJS Async providers 文档](https://docs.nestjs.com/fundamentals/async-providers)

## Asynchronous providers  异步提供者

有时，应用启动应该延迟到完成一个或多个异步任务。例如，您可能不希望在建立数据库连接之前开始接受请求。您可以使用异步提供者来实现这一点。

为此的语法是使用 `async/await` 与 `useFactory` 语法。工厂返回一个 `Promise`，工厂函数可以 `await` 异步任务。Nest 将等待 promise 解析后再实例化任何依赖（注入）此类提供者的类。

```typescript

{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}

```

> **提示** 了解更多关于自定义提供者语法的信息[请点击这里](https://docs.nestjs.com/fundamentals/custom-providers)。

### Injection  注入

异步提供者像其他提供者一样通过它们的令牌注入到其他组件中。在上面的示例中，您将使用构造 `@Inject('ASYNC_CONNECTION')`。

### Example  示例

[TypeORM 配方](https://docs.nestjs.com/recipes/sql-typeorm)有一个更实质性的异步提供者示例。