# Custom providers（自定义提供者）

来源：[NestJS Custom providers 文档](https://docs.nestjs.com/fundamentals/custom-providers)

## Custom providers  自定义提供者

在前面的章节中，我们探讨了依赖注入（DI）的各个方面以及它在 Nest 中的使用方式。其中一个例子是[基于构造函数的依赖注入](https://docs.nestjs.com/providers#dependency-injection)，用于将实例（通常是服务提供者）注入到类中。你不会惊讶地发现，依赖注入是以一种基本的方式构建到 Nest 核心中的。到目前为止，我们只探索了一种主要的模式。随着你的应用程序变得更加复杂，你可能需要利用 DI 系统的完整功能，所以让我们更详细地探索它们。

### DI fundamentals  DI 基础

依赖注入是一种[控制反转（IoC）](https://en.wikipedia.org/wiki/Inversion_of_control)技术，你将依赖项的实例化委托给 IoC 容器（在我们的例子中，是 NestJS 运行时系统），而不是在自己的代码中强制执行。让我们检查一下[Providers 章节](https://docs.nestjs.com/providers)中这个例子中发生了什么。

首先，我们定义了一个提供者。`@Injectable()` 装饰器将 `CatsService` 类标记为提供者。

`cats.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  findAll() {
    return this.cats;
  }
}
```

然后我们请求 Nest 将提供者注入到我们的控制器类中：

`cats.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

```typescript
import { Controller, Get, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

最后，我们在 Nest IoC 容器中注册提供者：

`app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

幕后到底发生了什么使这一切工作？在这个过程中有三个关键步骤：

1. 在 `cats.controller.ts` 中，`CatsController` 通过构造函数注入声明了对 `CatsService` token 的依赖：
2. 在 `cats.service.ts` 中，`@Injectable()` 装饰器声明 `CatsService` 类是一个可以被 Nest IoC 容器管理的类。

```typescript
  constructor(private catsService: CatsService)
```

1. 在 `app.module.ts` 中，我们将 token `CatsService` 与来自 `cats.service.ts` 文件的 `CatsService` 类关联起来。我们将在[下面](https://docs.nestjs.com/fundamentals/custom-providers#standard-providers)确切地看到这种关联（也称为注册）是如何发生的。

当 Nest IoC 容器实例化 `CatsController` 时，它首先查找任何依赖*。当它找到 `CatsService` 依赖时，它对 `CatsService` token 执行查找，根据注册步骤（上面的#3）返回 `CatsService` 类。假设是 `SINGLETON` 作用域（默认行为），Nest 将创建 `CatsService` 的实例，缓存它并返回它，或者如果已经缓存了一个，则返回现有实例。

*这个解释有点简化以说明要点。我们忽略的一个重要领域是，分析代码以查找依赖项的过程是非常复杂的，并且发生在应用程序启动期间。一个关键特性是依赖项分析（或"创建依赖图"）是可传递的。在上面的例子中，如果 `CatsService` 本身有依赖项，那些依赖项也会被解析。依赖图确保依赖项以正确的顺序被解析——基本上是"自下而上"。这种机制使开发者免于管理这种复杂的依赖图。

### Standard providers  标准提供者

让我们更仔细地看看 `@Module()` 装饰器。在 `app.module` 中，我们声明：

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers` 属性接受一个 `providers` 数组。到目前为止，我们通过类名列表提供了这些提供者。实际上，语法 `providers: [CatsService]` 是更完整的语法 `providers: [{ provide: CatsService, useClass: CatsService }]` 的简写。

现在我们看到了这种显式构造，我们可以理解注册过程。这里，我们清楚地将 token `CatsService` 与类 `CatsService` 关联起来。简写表示法仅仅是简化最常见用例的便利方式，其中 token 用于按相同名称请求类的实例。

### Custom providers  自定义提供者

当你的需求超出标准提供者提供的范围时会发生什么？这里有一些例子：

- 你想用模拟版本覆盖一个类进行测试
- 你想在第二个依赖中重用现有类
- 你想创建自定义实例而不是让 Nest 实例化（或返回缓存实例）一个类

Nest 允许你定义自定义提供者来处理这些情况。它提供了几种定义自定义提供者的方法。让我们逐步了解它们。

> **提示** 如果你在依赖解析方面遇到问题，你可以设置 `NEST_DEBUG` 环境变量，并在启动期间获取额外的依赖解析日志。

### Value providers: useValue  值提供者

`useValue` 语法对于注入常量值、将外部库放入 Nest 容器，或用模拟对象替换真实实现很有用。假设你想强制 Nest 为测试目的使用模拟的 `CatsService`。

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

在这个例子中，`CatsService` token 将解析为 `mockCatsService` 模拟对象。`useValue` 需要一个值——在这种情况下是一个具有与它正在替换的 `CatsService` 类相同接口的字面量对象。由于 TypeScript 的[结构化类型](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)，你可以使用任何具有兼容接口的对象，包括字面量对象或用 `new` 实例化的类实例。

### Non-class-based provider tokens  非基于类的提供者 tokens

到目前为止，我们使用类名作为提供者 tokens（在 `providers` 数组中列出的提供者中 `provide` 属性的值）。这与[基于构造函数的注入](https://docs.nestjs.com/providers#dependency-injection)使用的标准模式匹配，其中 token 也是类名。（如果这个概念不是完全清楚，请参考[DI 基础](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals)以复习 tokens）。有时，我们可能希望使用字符串或符号作为 DI token。例如：

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

在这个例子中，我们将字符串值 token（`'CONNECTION'`）与我们从外部文件导入的预先存在的 `connection` 对象关联起来。

> **注意** 除了使用字符串作为 token 值，你还可以使用 JavaScript [symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) 或 TypeScript [enums](https://www.typescriptlang.org/docs/handbook/enums.html)。

我们之前看到了如何使用标准[基于构造函数的注入](https://docs.nestjs.com/providers#dependency-injection)模式注入提供者。这种模式要求依赖项用类名声明。`'CONNECTION'` 自定义提供者使用字符串值 token。让我们看看如何注入这样的提供者。为此，我们使用 `@Inject()` 装饰器。这个装饰器接受单个参数——token。

```typescript
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

```typescript
@Injectable()
@Dependencies('CONNECTION')
export class CatsRepository {
  constructor(connection) {}
}
```

> **提示** `@Inject()` 装饰器是从 `@nestjs/common` 包导入的。

虽然我们在上面的例子中直接使用字符串 `'CONNECTION'` 作为说明目的，但为了干净的代码组织，最佳实践是在单独的文件中定义 tokens，例如 `constants.ts`。像对待在单独文件中定义并在需要的地方导入的 symbols 或 enums 一样对待它们。

### Class providers: useClass  类提供者

`useClass` 语法允许你动态确定 token 应该解析到的类。例如，假设我们有一个抽象（或默认）`ConfigService` 类。根据当前环境，我们希望 Nest 提供配置服务的不同实现。下面的代码实现了这样的策略。

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

让我们在这个代码示例中查看几个细节。你会注意到我们首先用字面量对象定义 `configServiceProvider`，然后将它传递给模块装饰器的 `providers` 属性。这只是一些代码组织，但功能上等同于我们到目前为止在本章中使用的例子。

此外，我们使用了 `ConfigService` 类名作为我们的 token。对于任何依赖于 `ConfigService` 的类，Nest 将注入所提供类（`DevelopmentConfigService` 或 `ProductionConfigService`）的实例，覆盖可能在其他地方声明的任何默认实现（例如，用 `@Injectable()` 装饰器声明的 `ConfigService`）。

### Factory providers: useFactory  工厂提供者

`useFactory` 语法允许动态创建提供者。实际的提供者将由工厂函数返回的值提供。工厂函数可以根据需要简单或复杂。一个简单的工厂可能不依赖于任何其他提供者。一个更复杂的工厂可以注入它需要计算结果的其他提供者。对于后一种情况，工厂提供者语法有一对相关的机制：

1. （可选的）`inject` 属性接受一个提供者数组，Nest 将在实例化过程中解析这些提供者并将它们作为参数传递给工厂函数。此外，这些提供者可以标记为可选的。这两个列表应该是相关的：Nest 将按照相同顺序将 `inject` 列表中的实例作为参数传递给工厂函数。下面示例演示了这一点。
2. 工厂函数可以接受（可选的）参数。

```typescript
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: MyOptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/             \__________________/
  //        This provider                The provider with this token
  //        is mandatory.                can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // class-based provider
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

```typescript
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider, optionalProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/            \__________________/
  //        This provider               The provider with this token
  //        can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // class-base provider
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

### Alias providers: useExisting  别名提供者

`useExisting` 语法允许你为现有提供者创建别名。这为访问同一提供者创建了两种方式。在下面的例子中，（字符串基础的）token `'AliasedLoggerService'` 是（类基础的）token `LoggerService` 的别名。假设我们有两个不同的依赖项，一个用于 `'AliasedLoggerService'`，一个用于 `LoggerService`。如果两个依赖项都以 `SINGLETON` 作用域指定，它们都将解析为同一实例。

```typescript
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

### Non-service based providers  非基于服务的提供者

虽然提供者经常提供服务，但它们的使用并不限于此。一个提供者可以提供任何值。例如，一个提供者可以基于当前环境提供配置对象数组，如下所示：

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

### Export custom provider  导出自定义提供者

像任何提供者一样，自定义提供者被限定在其声明模块的范围内。要使它对其他模块可见，它必须被导出。要导出自定义提供者，我们可以使用它的 token 或完整的提供者对象。

下面的示例显示了使用 token 导出：

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

或者，使用完整的提供者对象导出：

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```