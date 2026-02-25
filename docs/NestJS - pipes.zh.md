# Pipes（NestJS 管道）

来源：[NestJS Pipes 文档](https://docs.nestjs.com/pipes)

## Pipes  管道

管道是一个使用 `@Injectable()` 装饰器标注并实现 `PipeTransform` 接口的类。

管道有两个典型用途：

- **转换**：把输入数据转换成期望的格式（例如字符串转整数）
- **验证**：评估输入数据，若有效则原样通过，否则抛出异常

在这两种场景中，管道都作用于[控制器路由处理器](controllers#route-parameters)正在处理的 `arguments`。Nest 会在方法调用前插入管道，管道接收目标方法的参数并进行处理。转换或验证在此时完成，之后路由处理器用（可能已变换的）参数被调用。

Nest 内置了若干可直接使用的管道，你也可以构建自定义管道。本章会介绍内置管道并展示如何绑定到路由处理器，然后通过多个自定义管道示例说明如何从零实现管道。

> **提示**
> 管道运行在异常处理区间内。这意味着管道抛出的异常会被异常层（全局异常过滤器和当前上下文应用的[异常过滤器](https://docs.nestjs.com/exception-filters)）处理。由此可知，当管道抛出异常时，控制器方法不会被执行。这为在系统边界对外部输入进行验证提供了最佳实践方式。

### Built-in pipes  内置管道

Nest 内置了多种管道：

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`
- `ParseDatePipe`

它们均从 `@nestjs/common` 包导出。

先看 `ParseIntPipe` 的使用，这是 **转换** 的例子：它确保方法处理器参数被转换为 JavaScript 整数（若转换失败则抛出异常）。本章后续会给出一个简单自定义版 `ParseIntPipe`。下述示例也同样适用于其他内置转换管道（`ParseBoolPipe`、`ParseFloatPipe`、`ParseEnumPipe`、`ParseArrayPipe`、`ParseDatePipe`、`ParseUUIDPipe`，本章统称为 `Parse*` 管道）。

### Binding pipes  绑定管道

使用管道时，需要将管道实例绑定到合适的上下文。在 `ParseIntPipe` 示例中，我们希望将管道与某个路由处理器方法关联，并确保在方法调用前执行。我们通过以下构造来实现，这种方式称为在 **方法参数级** 绑定管道：

```
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

这保证 `findOne()` 方法收到的参数要么是数字（符合 `this.catsService.findOne()` 的预期），要么在路由处理器被调用之前抛出异常。

例如，若路由被这样调用：

```
GET localhost:3000/abc
```

Nest 会抛出如下异常：

```
{"statusCode":400,"message":"Validation failed (numeric string is expected)","error":"Bad Request"}
```

异常会阻止 `findOne()` 方法体执行。

上例中我们传入的是类（`ParseIntPipe`），而不是实例，实例化由框架负责，从而支持依赖注入。与管道、守卫类似，你也可以传入就地实例。当你需要通过选项自定义内置管道行为时，这很有用：

```
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE })) id: number,
) {
  return this.catsService.findOne(id);
}
```

绑定其他转换管道（所有 **Parse\*** 管道）也类似。这些管道可用于验证路由参数、查询参数与请求体值。

例如，查询参数：

```
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

下面示例使用 `ParseUUIDPipe` 解析字符串参数并验证其是否为 UUID：

```
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

```js
@Get(':uuid')
@Bind(Param('uuid', new ParseUUIDPipe()))
async findOne(uuid) {
  return this.catsService.findOne(uuid);
}
```

> **提示**
> 使用 `ParseUUIDPipe()` 时，你会解析 UUID 版本 3/4/5。若只需要特定版本，可在管道选项中传入版本。

上面我们看到的是如何绑定内置 `Parse*` 管道。验证管道的绑定略有不同，我们将在下一节讨论。

> **提示**
> 另请参阅[验证技术](https://docs.nestjs.com/techniques/validation)获取更多验证管道示例。

### Custom pipes  自定义管道

你可以自定义管道。虽然 Nest 提供了健壮的 `ParseIntPipe` 与 `ValidationPipe`，我们还是从零实现一个简化版来展示自定义管道的构建方式。

先实现一个简单的 `ValidationPipe`。最初它只接收输入值并立即返回，行为类似恒等函数。

`validation.pipe.ts`

```ts
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

```js
import { Injectable } from '@nestjs/common';

@Injectable()
export class ValidationPipe {
  transform(value, metadata) {
    return value;
  }
}
```

> **提示**
> `PipeTransform<T, R>` 是任何管道必须实现的泛型接口。`T` 表示输入 `value` 的类型，`R` 表示 `transform()` 的返回类型。

每个管道必须实现 `transform()` 方法以满足 `PipeTransform` 接口约定。该方法有两个参数：

- `value`
- `metadata`

`value` 是当前被处理的方法参数（在进入路由处理器之前），`metadata` 是该参数的元数据对象。其属性如下：

```
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

这些属性描述当前被处理的参数：

- `type`：参数类型，指示是请求体 `@Body()`、查询 `@Query()`、路由参数 `@Param()`，或自定义参数（详见[这里](https://docs.nestjs.com/custom-decorators)）
- `metatype`：参数的元类型，例如 `String`。注意：如果你省略类型声明或使用原生 JavaScript，则该值为 `undefined`
- `data`：传给装饰器的字符串，例如 `@Body('string')`；若装饰器括号留空则为 `undefined`

> **警告**
> TypeScript 接口在编译后会消失。因此，如果方法参数类型是接口而不是类，则 `metatype` 值会变为 `Object`。

### Schema based validation  基于 Schema 的验证

让验证管道更有用一些。看看 `CatsController` 的 `create()` 方法，我们可能希望在运行服务方法前确保 post body 对象有效。

```
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

重点是 `createCatDto` 这个 body 参数，其类型是 `CreateCatDto`：

`create-cat.dto.ts`

```
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

我们希望任何对 create 方法的请求都包含有效的请求体，因此需要验证 `createCatDto` 的三个成员。可以在路由处理器里做，但这会违反 **单一职责原则**（SRP）。

另一个方案是创建 **验证类** 并委托给它，但这要求在每个方法开头记得调用验证器。

那用验证中间件呢？可以，但无法创建可跨全应用使用的 **通用中间件**，因为中间件不了解 **执行上下文**，包括将被调用的处理器及其参数。

这正是管道的用武之地。接下来完善验证管道。

## Learn the right way!  正确学习方式！

- 80+ 章节
- 5+ 小时视频
- 官方证书
- 深度研讨

[探索官方课程](https://courses.nestjs.com)

### Object schema validation  对象 Schema 验证

实现对象验证有多种方案，以 [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) 的方式保持简洁。其中一种常见方法是 **基于 Schema 的验证**。我们来尝试这种方式。

[Zod](https://zod.dev/) 库可以用易读的 API 构建 schema。下面用 Zod schema 构建验证管道。

先安装依赖：

```
$ npm install --save zod
```

下例中我们创建一个类，在构造函数中接收 schema，并调用 `schema.parse()` 来验证输入参数。正如前述，**验证管道** 要么返回原值，要么抛出异常。

下一节会展示如何使用 `@UsePipes()` 为指定控制器方法提供合适的 schema，使管道可在不同上下文复用。

```
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

```js
import { BadRequestException } from '@nestjs/common';

export class ZodValidationPipe {
  constructor(private schema) {}
  transform(value, metadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

### Binding validation pipes  绑定验证管道

前面我们看到了如何绑定转换管道（`ParseIntPipe` 与其他 `Parse*` 管道）。

绑定验证管道也很简单。此时我们需要在方法级别绑定管道。在当前示例中，我们需要：

1. 创建 `ZodValidationPipe` 实例
2. 在构造函数中传入特定上下文的 Zod schema
3. 将管道绑定到方法

Zod schema 示例：

```
import { z } from 'zod';

export const createCatSchema = z.object({
  name: z.string(),
  age: z.number(),
  breed: z.string(),
}).required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

使用 `@UsePipes()` 装饰器如下：

`cats.controller.ts`

```
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

```js
@Post()
@Bind(Body())
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> **提示**
> `@UsePipes()` 装饰器来自 `@nestjs/common` 包。

> **警告**
> `zod` 库要求在 `tsconfig.json` 中启用 `strictNullChecks`。

### Class validator  类验证器

> **警告**
> 本节技术需要 TypeScript；若应用使用原生 JavaScript，则不可用。

我们来看看另一种验证实现方式。Nest 与 [class-validator](https://github.com/typestack/class-validator) 库配合良好。它允许使用装饰器进行验证。在与 Nest 的 **Pipe** 能力结合后非常强大，因为我们可以访问处理属性的 `metatype`。开始前需要安装依赖：

```
$ npm i --save class-validator class-transformer
```

安装后，为 `CreateCatDto` 添加一些装饰器。该方法的优点是 `CreateCatDto` 类仍是 Post body 的单一真相来源，而无需另建验证类。

`create-cat.dto.ts`

```
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> **提示**
> 更多 class-validator 装饰器请看[这里](https://github.com/typestack/class-validator#usage)。

现在可以创建使用这些注解的 `ValidationPipe` 类：

`validation.pipe.ts`

```
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> **提示**
> 不必自己构建通用验证管道，因为 Nest 内置了 `ValidationPipe`。内置 `ValidationPipe` 提供了比本章示例更多的选项（示例为说明机制而保持简化）。详见[这里](https://docs.nestjs.com/techniques/validation)。

> **注意**
> 上面使用了 [class-transformer](https://github.com/typestack/class-transformer) 库，它与 **class-validator** 库同作者，因此配合非常好。

我们来逐步理解这段代码。首先，`transform()` 方法被标记为 `async`，这是因为 Nest 支持同步与 **异步** 管道。我们将其设为 `async`，因为 class-validator 的某些验证[可以是异步的](https://github.com/typestack/class-validator#custom-validation-classes)（返回 Promise）。

其次，我们使用解构从 `ArgumentMetadata` 中取出 `metatype` 字段。这只是简写写法，本质上仍是获取完整 `ArgumentMetadata` 再赋值给 `metatype` 变量。

接着是辅助函数 `toValidate()`，用于当当前参数是原生 JavaScript 类型时跳过验证（这些类型无法附加验证装饰器，因此无需验证）。

然后使用 class-transformer 的 `plainToInstance()` 将普通对象转换为具备类型信息的对象，以便进行验证。原因是请求体从网络反序列化后 **不包含类型信息**（这是底层平台如 Express 的工作方式）。class-validator 需要 DTO 上的验证装饰器，因此需要进行此转换。

最后，如前所述，作为 **验证管道**，它要么返回原值，要么抛出异常。

下一步是绑定 `ValidationPipe`。管道可作用于参数级、方法级、控制器级或全局级。之前 Zod 验证管道示例中，我们看到的是方法级绑定。

下面示例把管道实例绑定到 `@Body()` 装饰器上，以验证 post body：

`cats.controller.ts`

```
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

参数级管道适合只针对某一个参数进行验证的场景。

### Global scoped pipes  全局管道

由于 `ValidationPipe` 尽量保持通用性，把它设置为 **全局管道** 可以最大化其价值，从而应用到整个应用的所有路由处理器。

`main.ts`

```
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> **注意**
> 在[混合应用](faq/hybrid-application)中，`useGlobalPipes()` 不会为网关和微服务设置管道。对于“标准”（非混合）微服务应用，`useGlobalPipes()` 会全局挂载管道。

全局管道作用于整个应用，对所有控制器与路由处理器生效。

注意，从模块之外注册的全局管道（如上使用 `useGlobalPipes()`）无法注入依赖，因为绑定发生在模块上下文之外。为了解决这一问题，你可以 **直接从任何模块** 设置全局管道，使用如下构造：

`app.module.ts`

```
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> **提示**
> 使用此方式进行管道依赖注入时，请注意无论在哪个模块中使用该构造，管道实际上都是全局的。应放在管道（上例中的 `ValidationPipe`）定义所在的模块中。另外，`useClass` 并不是注册自定义提供者的唯一方式，更多请看[这里](https://docs.nestjs.com/fundamentals/custom-providers)。

### The built-in ValidationPipe  内置 ValidationPipe

提醒一下：你不必自己构建通用验证管道，因为 Nest 提供了内置 `ValidationPipe`。内置 `ValidationPipe` 提供了比本章示例更丰富的选项（本章示例为说明机制而简化）。完整细节与示例请看[这里](https://docs.nestjs.com/techniques/validation)。

### Transformation use case  转换用例

验证并不是自定义管道的唯一用途。章节开头提到，管道还可以 **转换** 输入数据为所需格式。原因在于 `transform` 的返回值会完全覆盖之前的参数值。

何时有用？例如客户端传来的数据在交给路由处理器前需要转换（如字符串转整数）；或者必需字段缺失时需要默认值。**转换管道** 可以在客户端请求与处理器之间插入处理逻辑来完成这些工作。

下面是一个简单的 `ParseIntPipe`，负责把字符串解析成整数。（如前所述，Nest 内置 `ParseIntPipe` 更完善；这里仅作自定义管道示例。）

`parse-int.pipe.ts`

```
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

```js
import { Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe {
  transform(value, metadata) {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

可以像下面这样把管道绑定到选定的参数：

```
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

```js
@Get(':id')
@Bind(Param('id', new ParseIntPipe()))
async findOne(id) {
  return this.catsService.findOne(id);
}
```

另一个有用的转换案例是根据请求中的 id 选出数据库中的 **已有用户** 实体：

```
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

```js
@Get(':id')
@Bind(Param('id', UserByIdPipe))
findOne(userEntity) {
  return userEntity;
}
```

我们将该管道的实现留给读者，但要注意：像其他转换管道一样，它接收输入值（`id`）并返回输出值（`UserEntity` 对象）。这能让你的代码更声明式、更 [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)，把样板逻辑从处理器抽到通用管道中。

### Providing defaults  提供默认值

`Parse*` 管道期望参数值已定义。遇到 `null` 或 `undefined` 时会抛出异常。若要允许端点处理缺失的查询参数值，需要在 `Parse*` 管道处理前注入默认值。`DefaultValuePipe` 正是为此服务。只需在 `@Query()` 装饰器中先实例化 `DefaultValuePipe`，再接上相关 `Parse*` 管道，如下所示：

```
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```
