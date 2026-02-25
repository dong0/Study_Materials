# Custom decorators（NestJS 自定义装饰器）

来源：[NestJS Custom decorators 文档](https://docs.nestjs.com/custom-decorators)

## Custom route decorators  自定义路由装饰器

Nest 构建在一种称为 **装饰器** 的语言特性之上。装饰器在许多常用编程语言中都很常见，但在 JavaScript 世界里仍相对较新。为更好理解装饰器的工作方式，我们建议阅读[这篇文章](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)。一个简单定义如下：

> ES2016 装饰器是一个表达式，返回一个函数，该函数可以接收目标、名称和属性描述符作为参数。你通过在要装饰的对象前加 `@` 字符并放在最顶部来应用装饰器。装饰器可以用于类、方法或属性。

### Param decorators  参数装饰器

Nest 提供了一组有用的 **参数装饰器**，可与 HTTP 路由处理器一起使用。下面是内置装饰器及其对应的 Express（或 Fastify）对象：

- `@Request(), @Req()` → `req`
- `@Response(), @Res()` → `res`
- `@Next()` → `next`
- `@Session()` → `req.session`
- `@Param(param?: string)` → `req.params` / `req.params[param]`
- `@Body(param?: string)` → `req.body` / `req.body[param]`
- `@Query(param?: string)` → `req.query` / `req.query[param]`
- `@Headers(param?: string)` → `req.headers` / `req.headers[param]`
- `@Ip()` → `req.ip`
- `@HostParam()` → `req.hosts`

此外，你也可以创建自己的 **自定义装饰器**。为什么这有用？

在 Node.js 世界中，常见做法是把属性附加到 **request** 对象，然后在每个路由处理器中手动取出，例如：

```
const user = req.user;
```

为了让代码更可读、更直观，你可以创建 `@User()` 装饰器，在所有控制器中复用。

`user.decorator.ts`

```
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

随后，你就可以在需要的地方使用它。

```
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
```

```js
@Get()
@Bind(User())
async findOne(user) {
  console.log(user);
}
```

### Passing data  传递数据

当装饰器的行为依赖某些条件时，可以使用 `data` 参数向装饰器工厂函数传入数据。一个典型场景是创建一个自定义装饰器，通过键从请求对象中提取属性。假设我们的[认证层](techniques/authentication#implementing-passport-strategies)验证请求并把用户实体附加到请求对象上。一个已认证请求的用户实体可能如下：

```
{"id":101,"firstName":"Alan","lastName":"Turing","email":"alan@email.com","roles":["admin"]}
```

我们可以定义一个装饰器，接收属性名作为 key，并返回其对应值（如果不存在则返回 undefined，或 `user` 对象还未创建时也返回 undefined）。

`user.decorator.ts`

```
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);
```

```js
import { createParamDecorator } from '@nestjs/common';

export const User = createParamDecorator((data, ctx) => {
  const request = ctx.switchToHttp().getRequest();
  const user = request.user;
  return data ? user && user[data] : user;
});
```

接下来你就可以通过 `@User()` 访问特定属性：

```
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

```js
@Get()
@Bind(User('firstName'))
async findOne(firstName) {
  console.log(`Hello ${firstName}`);
}
```

你可以用同一个装饰器、传不同的 key 来访问不同属性。如果 `user` 对象很深或很复杂，这会使处理器实现更简洁、更易读。

> **提示**
> 对 TypeScript 用户：`createParamDecorator<T>()` 是泛型，可显式加强类型安全。例如 `createParamDecorator<string>((data, ctx) => ...)`。或者在工厂函数中指定参数类型，如 `createParamDecorator((data: string, ctx) => ...)`。若两者都省略，`data` 的类型将为 `any`。

### Working with pipes  与管道协作

Nest 对自定义参数装饰器的处理方式与内置装饰器（`@Body()`、`@Param()`、`@Query()`）相同。这意味着管道也会对自定义参数执行（在示例中为 `user` 参数）。此外，你可以直接把管道应用到自定义装饰器上：

```
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true })) user: UserEntity,
) {
  console.log(user);
}
```

```js
@Get()
@Bind(User(new ValidationPipe({ validateCustomDecorators: true })))
async findOne(user) {
  console.log(user);
}
```

> **提示**
> 必须将 `validateCustomDecorators` 选项设为 true。默认情况下，`ValidationPipe` 不会验证使用自定义装饰器标注的参数。

### Decorator composition  装饰器组合

Nest 提供了一个辅助方法来组合多个装饰器。例如，你希望把与认证相关的所有装饰器合并成一个装饰器，可以这样做：

`auth.decorator.ts`

```
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

```js
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

然后可这样使用自定义 `@Auth()`：

```
@Get('users')
@Auth('admin')
findAllUsers() {}
```

这会以一次声明同时应用四个装饰器。

> **警告**
> `@nestjs/swagger` 包的 `@ApiHideProperty()` 装饰器不可组合，无法与 `applyDecorators` 一起正常工作。

