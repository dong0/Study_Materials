# Modules（NestJS 模块）

来源：[NestJS Modules 文档](https://docs.nestjs.com/modules)

## Modules  模块

模块是一个使用 `@Module()` 装饰器标注的类。该装饰器提供元数据，**Nest** 用这些元数据高效地组织和管理应用结构。

每个 Nest 应用至少有一个模块，即 **根模块**，它是 Nest 构建 **应用图** 的起点。该图是 Nest 用于解析模块与提供者之间关系与依赖的内部结构。小型应用可能只有一个根模块，但一般不会如此。模块 **强烈推荐** 用来组织组件。对大多数应用而言，你会有多个模块，每个模块封装一组紧密相关的 **能力**。

`@Module()` 装饰器接收一个包含以下属性的对象来描述模块：

- `providers`：由 Nest 注入器实例化的提供者，并可至少在此模块内共享
- `controllers`：此模块中定义并需要实例化的一组控制器
- `imports`：导入的模块列表，这些模块导出当前模块所需的提供者
- `exports`：此模块提供者的子集，用于向导入该模块的其他模块公开。你可以导出提供者本身或其 token（`provide` 值）

模块默认 **封装** 提供者，这意味着你只能注入当前模块的提供者，或从已导入模块中显式导出的提供者。模块导出的提供者本质上就是该模块的公共接口/API。

### Feature modules  功能模块

在我们的示例中，`CatsController` 和 `CatsService` 紧密相关并服务于同一业务领域。把它们分组到一个功能模块是合理的。功能模块组织与特定功能相关的代码，有助于保持清晰边界与更好的组织结构。这在应用或团队扩展时尤为重要，也与 [SOLID](https://en.wikipedia.org/wiki/SOLID) 原则一致。

接下来我们创建 `CatsModule` 来展示如何把控制器和服务进行分组。

`cats/cats.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> **提示**
> 使用 CLI 创建模块，只需执行 `nest g module cats` 命令。

上面我们在 `cats.module.ts` 中定义了 `CatsModule`，并把与该模块相关的内容移动到 `cats` 目录。最后一步是将该模块导入根模块（`app.module.ts` 中定义的 `AppModule`）。

`app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

此时目录结构如下：

```
src
cats
dto
create-cat.dto.ts
interfaces
cat.interface.ts
cats.controller.ts
cats.module.ts
cats.service.ts
app.module.ts
main.ts
```

### Shared modules  共享模块

在 Nest 中，模块默认是 **单例**，因此你可以轻松地在多个模块之间共享同一个提供者实例。

每个模块自动就是一个 **共享模块**。一旦创建，就可被任何模块复用。假设我们想在多个模块之间共享 `CatsService` 的实例，首先需要在模块的 `exports` 数组中 **导出** `CatsService` 提供者，如下所示：

`cats.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

现在，任何导入 `CatsModule` 的模块都可以访问 `CatsService`，并与所有导入它的模块共享同一实例。

如果我们在每个需要它的模块中直接注册 `CatsService`，当然也能工作，但会导致每个模块拥有不同的 `CatsService` 实例。这会增加内存使用，并可能引起意外行为（例如服务持有内部状态时导致状态不一致）。

通过将 `CatsService` 封装在 `CatsModule` 中并导出它，我们确保所有导入 `CatsModule` 的模块复用同一 `CatsService` 实例。这不仅降低内存消耗，也使行为更可预测，更易管理共享状态或资源。这是 NestJS 等框架中模块化与依赖注入的关键优势之一。

## Explore your graph with NestJS Devtools  使用 NestJS Devtools 探索你的图谱

- 图谱可视化
- 路由导航
- 交互式演练场
- CI/CD 集成

[注册](https://devtools.nestjs.com)

### Module re-exporting  模块再导出

如上所示，模块可以导出自身内部的提供者。此外，模块还可以再导出它所导入的模块。在下面例子中，`CommonModule` 既被导入到 `CoreModule`，又被 `CoreModule` 导出，从而供其他导入该模块的模块使用。

```
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

### Dependency injection  依赖注入

模块类本身也可以 **注入** 提供者（例如用于配置）：

`cats.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

```js
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

但是，模块类本身不能作为提供者被注入，这是因为会产生 [循环依赖](https://docs.nestjs.com/fundamentals/circular-dependency)。

### Global modules  全局模块

如果你必须在很多地方重复导入相同的一组模块，会变得繁琐。与 Nest 不同，[Angular](https://angular.dev) 的 `providers` 是在全局作用域注册的，一旦定义即可全局可用。Nest 则把提供者封装在模块作用域内，不导入模块就无法在其他模块中使用其提供者。

当你希望一组提供者在全局默认可用（例如辅助工具、数据库连接等）时，可以使用 `@Global()` 装饰器将模块声明为 **全局**：

```
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` 装饰器使模块具备全局作用域。全局模块应 **仅注册一次**，通常在根模块或核心模块中。在上述示例中，`CatsService` 将随处可用，任何需要注入该服务的模块都不必再在 imports 中引入 `CatsModule`。

> **提示**
> 将一切设为全局并非推荐的设计实践。全局模块虽能减少样板代码，但更好的方式通常是使用 `imports` 数组，以受控且清晰的方式将模块 API 暴露给其他模块。这能提供更好的结构与可维护性，并避免应用中无关部分的耦合。

### Dynamic modules  动态模块

动态模块让你创建可在运行时配置的模块。它在需要提供灵活、可定制模块（可根据选项或配置生成提供者）时特别有用。下面是 **动态模块** 的简要示例：

```ts
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

```js
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> **提示**
> `forRoot()` 方法可以同步或异步（例如通过 `Promise`）返回动态模块。

该模块默认在 `@Module()` 元数据中定义 `Connection` 提供者，但根据 `forRoot()` 方法传入的 `entities` 和 `options`，还会暴露一组提供者（例如仓储）。注意：动态模块返回的属性会 **扩展**（而不是覆盖）`@Module()` 装饰器中定义的基础元数据。这意味着静态声明的 `Connection` 提供者与动态生成的仓储提供者都会被导出。

如果你想在全局范围注册动态模块，可将 `global` 属性设置为 `true`：

```
{ global: true, module: DatabaseModule, providers: providers, exports: providers }
```

> **警告**
> 如前所述，把一切设为全局 **不是好的设计决策**。

`DatabaseModule` 可以按如下方式导入并配置：

```
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

如果你要再导出动态模块，可以在 exports 数组中省略 `forRoot()` 调用：

```
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[动态模块](https://docs.nestjs.com/fundamentals/dynamic-modules)章节更深入地讲解了该主题，并提供了一个[可运行示例](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)。

> **提示**
> 了解如何使用 `ConfigurableModuleBuilder` 构建高度可定制的动态模块，请看[此章节](https://docs.nestjs.com/fundamentals/dynamic-modules#configurable-module-builder)。
