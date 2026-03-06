# Providers（NestJS 提供者）

来源：[NestJS Providers 文档](https://docs.nestjs.com/providers)

## Providers  提供者

提供者是 Nest 的核心概念。许多基础的 Nest 类，如服务、仓储、工厂和辅助类，都可以被视为提供者。提供者的关键思想是：它可以作为依赖被 **注入**，从而让对象之间建立各种关系。“连接”这些对象的职责主要由 Nest 运行时系统处理。

在上一章中，我们创建了一个简单的 `CatsController`。控制器应当处理 HTTP 请求，并将更复杂的任务委托给 **提供者**。提供者是普通的 JavaScript 类，在 NestJS 模块中以 `providers` 的形式声明。更多细节请参考 “Modules” 章节。

> **提示**
> 由于 Nest 使你能够以面向对象的方式设计和组织依赖，我们强烈建议遵循 [SOLID 原则](https://en.wikipedia.org/wiki/SOLID)。

### Services  服务

让我们先创建一个简单的 `CatsService`。这个服务将处理数据的存储与检索，并由 `CatsController` 使用。因为它负责管理应用逻辑，所以非常适合作为提供者来定义。

`cats.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

```js
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> **提示**
> 使用 CLI 创建服务，只需执行 `nest g service cats` 命令。

我们的 `CatsService` 是一个基础类，包含一个属性和两个方法。这里的关键点是 `@Injectable()` 装饰器。该装饰器为类附加元数据，表明 `CatsService` 可以由 Nest 的 [IoC](https://en.wikipedia.org/wiki/Inversion_of_control) 容器管理。

此外，这个示例还使用了一个 `Cat` 接口，它可能类似于：

`interfaces/cat.interface.ts`

```ts
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

现在我们有了用于检索猫的服务类，让我们在 `CatsController` 中使用它：

`cats.controller.ts`

```ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

```js
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

`CatsService` 通过类构造函数被 **注入**。注意 `private` 关键字的用法。这个简写允许我们在同一行完成 `catsService` 成员的声明与初始化，从而简化过程。

### Dependency injection  依赖注入

Nest 以一种强大的设计模式构建，即 **依赖注入**。我们非常建议阅读一篇官方 [Angular 文档](https://angular.dev/guide/di) 中关于该概念的文章。

在 Nest 中，得益于 TypeScript 的能力，依赖管理非常直接，因为它们是基于类型解析的。在下面的示例中，Nest 会通过创建并返回 `CatsService` 的实例来解析 `catsService`（若为单例，且已在其他地方被请求过，则返回已存在的实例）。该依赖随后被注入到控制器构造函数中（或赋值给指定属性）：

```
constructor(private catsService: CatsService) {}
```

### Scopes  作用域

提供者通常具有一个生命周期（“作用域”），与应用生命周期一致。当应用启动时，每个依赖都必须被解析，这意味着每个提供者都会被实例化。同样，当应用关闭时，所有提供者都会被销毁。不过，也可以让提供者变成 **请求作用域**，也就是其生命周期与单次请求绑定，而不是与应用生命周期绑定。更多内容可参考 [注入作用域](https://docs.nestjs.com/fundamentals/injection-scopes) 章节。

## Learn the right way!  正确学习方式！

- 80+ 章节
- 5+ 小时视频
- 官方证书
- 深度研讨

[探索官方课程](https://courses.nestjs.com)

### Custom providers  自定义提供者

Nest 自带一个反转控制（IoC）容器，用于管理提供者之间的关系。这是依赖注入的基础，但它实际上比我们目前覆盖的内容更强大。定义提供者有多种方式：可以使用普通值、类、以及异步或同步工厂。更多提供者定义示例请参阅 [依赖注入](https://docs.nestjs.com/fundamentals/dependency-injection) 章节。

### Optional providers  可选提供者

有时你会遇到并非总是需要解析的依赖。例如，你的类可能依赖一个 **配置对象**，但如果未提供，应该使用默认值。这种情况下依赖是可选的，缺少配置提供者不应导致错误。

要将提供者标记为可选，请在构造函数签名中使用 `@Optional()` 装饰器。

```
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

在上面的示例中，我们使用了自定义提供者，这就是为什么包含了 `HTTP_OPTIONS` 自定义 **token**。之前的示例展示了基于构造函数的注入，其中依赖通过构造函数中的类来指明。关于自定义提供者及其关联 token 的更多细节，请参阅 [自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers) 章节。

### Property-based injection  基于属性的注入

我们迄今使用的方式称为基于构造函数的注入，即提供者通过构造函数注入。在某些特定场景中，**基于属性的注入** 会更有用。例如，如果顶层类依赖一个或多个提供者，在子类中通过 `super()` 逐级传递可能变得繁琐。为避免这种情况，你可以直接在属性层使用 `@Inject()` 装饰器。

```
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> **警告**
> 如果你的类不继承其他类，一般更推荐使用 **基于构造函数** 的注入方式。构造函数能清晰地说明需要哪些依赖，相比用 `@Inject` 标注的类属性，代码可读性更好、可见性更强。

### Provider registration  提供者注册

现在我们已经定义了一个提供者（`CatsService`）和一个使用方（`CatsController`），我们需要将服务注册到 Nest 中，以便它能完成注入。这通过编辑模块文件（`app.module.ts`）并将服务添加到 `@Module()` 装饰器中的 `providers` 数组来完成。

`app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

Nest 现在可以解析 `CatsController` 类的依赖。

此时，我们的目录结构应类似于：

```
src
cats
dto
create-cat.dto.ts
interfaces
cat.interface.ts
cats.controller.ts
cats.service.ts
app.module.ts
main.ts
```

### Manual instantiation  手动实例化

到目前为止，我们已经介绍了 Nest 如何自动处理依赖解析的细节。然而在某些情况下，你可能需要绕开内置的依赖注入系统，手动获取或实例化提供者。下面简要介绍两种方式：

- 要获取现有实例或动态实例化提供者，可使用 [Module reference](https://docs.nestjs.com/fundamentals/module-ref)。
- 要在 `bootstrap()` 函数中获取提供者（例如用于独立应用，或在启动期间使用配置服务），可参考 [独立应用](https://docs.nestjs.com/standalone-applications)。
