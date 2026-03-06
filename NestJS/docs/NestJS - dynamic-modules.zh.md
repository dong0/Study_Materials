# Dynamic modules（NestJS 动态模块）

来源：[NestJS Dynamic modules 文档](https://docs.nestjs.com/fundamentals/dynamic-modules)

## Dynamic modules  动态模块

[Modules chapter](https://docs.nestjs.com/modules) 介绍了 Nest 模块的基础知识，并简要介绍了[动态模块](https://docs.nestjs.com/modules#dynamic-modules)。本章将扩展动态模块的主题。通过本章的学习，您将对什么是动态模块以及何时以及如何使用它们有很好的理解。

### Introduction  简介

大多数应用代码示例在概述部分的文档中使用的是常规的或静态模块。模块定义了组成整体应用模块化部分的组件组，如[提供者](https://docs.nestjs.com/providers)和[控制器](https://docs.nestjs.com/controllers)。它们为这些组件提供执行上下文或作用域。例如，在模块中定义的提供者对该模块的其他成员可见，而无需导出它们。当提供者需要在模块外部可见时，它首先从其宿主模块导出，然后导入到其消费模块。

让我们看一个熟悉的示例。

首先，我们将定义一个 `UsersModule` 来提供和导出 `UsersService`。`UsersModule` 是 `UsersService` 的宿主模块。

```typescript

import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

```

接下来，我们将定义一个 `AuthModule`，它导入 `UsersModule`，使 `UsersModule` 的导出提供者可在 `AuthModule` 中使用：

```typescript

import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}

```

这些构造允许我们在例如 `AuthModule` 中托管的 `AuthService` 中注入 `UsersService`：

```typescript

import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    使用 this.usersService 的实现
  */
}

```

我们将此称为静态模块绑定。Nest 需要的将模块连接在一起的所有信息都已经在宿主和消费模块中声明了。让我们解开这个过程中发生的事情。Nest 通过以下方式使 `UsersService` 在 `AuthModule` 中可用：

1. 在 `AuthService` 中注入 `UsersService` 的实例。
2. 实例化 `AuthModule`，并使 `UsersModule` 的导出提供者可用于 `AuthModule` 中的组件（就像它们在 `AuthModule` 中声明一样）。
3. 实例化 `UsersModule`，包括传递导入其他模块（`UsersModule` 本身消费的模块），并传递解析任何依赖项（参见[自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers)）。

### Dynamic module use case  动态模块用例

使用静态模块绑定时，消费模块没有机会影响从宿主模块导入的提供者的配置。这很重要吗？考虑这样的情况，我们有一个通用的模块，需要在不同的用例中表现不同。这类似于许多系统中的"插件"概念，其中通用设施需要一些配置才能被消费者使用。

在 Nest 中，一个很好的例子是配置模块。许多应用发现使用配置模块来外部化配置细节很有用。这使得在不同的部署中动态更改应用设置变得容易：例如，开发人员的开发数据库、用于暂存/测试环境的暂存数据库等。通过将配置参数的管理委托给配置模块，应用源代码独立于配置参数。

挑战在于配置模块本身，因为它是通用的（类似于"插件"），需要由其消费模块自定义。这就是动态模块发挥作用的地方。使用动态模块功能，我们可以使我们的配置模块动态化，以便消费模块可以使用 API 在导入时控制配置模块的自定义。

换句话说，动态模块提供了一个 API，用于将一个模块导入另一个模块，并自定义该模块的属性和行为，而不是使用我们迄今为止看到的静态绑定。

### Config module example  配置模块示例

我们将使用本节中配置章节的基本示例代码版本。完整的版本在本章末尾可作为工作[示例](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)获得。

我们的要求是使 `ConfigModule` 接受一个 `options` 对象来定制它。让我们假设我们希望能够管理 `.env` 文件在项目根目录下的文件夹中。例如，假设我们想将各种 `.env` 文件存储在项目根目录下名为 `config` 的文件夹中（与 `src` 同级文件夹）。我们希望能够在不同项目中使用 `ConfigModule` 时选择不同的文件夹。

动态模块使我们能够将参数传递到被导入的模块中，以便我们可以更改其行为。让我们看看这是如何工作的。从消费模块的角度来看，这种情况可能是什么样子，这很有帮助，然后反向工作。首先，让我们快速回顾静态导入 `ConfigModule` 的示例（即，没有导入模块的行为影响能力的途径）。请特别注意 `@Module()` 装饰器中的 `imports` 数组：

```typescript

import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

```

让我们考虑传递配置对象时的动态模块导入可能是什么样子。在这些示例中比较 `imports` 数组之间的差异：

```typescript

import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

```

让我们看看上面动态示例中发生了什么。移动部件有哪些？

1. 我们可以推断 `register()` 方法必须返回类似 `module` 的东西，因为它的返回值出现在我们熟悉的 `imports` 列表中，该列表包含我们迄今为止看到的模块列表。
2. `register()` 方法是由我们定义的，所以我们可以接受任何我们喜欢的输入参数。在这种情况下，我们将接受一个具有合适属性的简单 `options` 对象，这是典型的案例。
3. `ConfigModule` 是一个普通的类，所以我们可以推断它必须有一个名为 `register()` 的静态方法。我们知道它是静态的，因为我们是在 `ConfigModule` 类上调用它，而不是在类的实例上。注意：这个方法，我们很快就会创建，可以有任何任意名称，但按照约定我们应该称它为 `forRoot()` 或 `register()`。

事实上，我们的 `register()` 方法将返回一个 `DynamicModule`。动态模块只是在运行时创建的模块，具有与静态模块完全相同的属性，加上一个名为 `module` 的附加属性。让我们快速回顾一个静态模块声明示例，特别注意传递给装饰器的模块选项：

```typescript

@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})

```

动态模块必须返回具有完全相同接口的对象，加上一个名为 `module` 的附加属性。`module` 属性作为模块的名称，应该与模块的类名相同，如下例所示。

> **提示** 对于动态模块，模块选项对象的所有属性都是可选的，除了 `module`。

那么静态 `register()` 方法呢？我们现在可以看到它的任务是返回具有 `DynamicModule` 接口的对象。当我们调用它时，我们实际上是为 `imports` 列表提供了一个模块，类似于通过列出模块类名的方式。换句话说，动态模块 API 只返回一个模块，但与其在 `@Module` 装饰器中固定属性，我们以编程方式指定它们。

还有几个细节需要涵盖以使图片完整：

1. 动态模块本身可以导入其他模块。我们在这个示例中不会这样做，但如果动态模块依赖于其他模块的提供者，您将使用可选的 `imports` 属性导入它们。同样，这与使用 `@Module()` 装饰器为静态模块声明元数据的方式完全类似。
2. 我们现在可以说 `@Module()` 装饰器的 `imports` 属性不仅可以接受模块类名（例如，`imports: [UsersModule]`），还可以接受返回动态模块的函数（例如，`imports: [ConfigModule.register(...)]`）。

有了这个理解，我们现在可以看看我们的动态 `ConfigModule` 声明必须是什么样子。让我们试试。

```typescript

import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}

```

现在应该清楚这些部分是如何组合在一起的。调用 `ConfigModule.register(...)` 返回一个具有属性的 `DynamicModule` 对象，这些属性本质上与我们迄今为止通过 `@Module()` 装饰器作为元数据提供的属性相同。

> **提示** 从 `@nestjs/common` 导入 `DynamicModule`。

然而，我们的动态模块还不够有趣，因为我们还没有引入我们希望做的那种配置能力。让我们接下来解决这个问题。

### Module configuration  模块配置

显然，为 `ConfigModule` 定制行为的解决方案是在静态 `register()` 方法中传递一个 `options` 对象，就像我们上面猜测的那样。让我们再次看看我们消费模块的 `imports` 属性：

```typescript

import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

```

这很好地处理了将 `options` 对象传递到我们的动态模块。那么我们如何在 `ConfigModule` 中使用这个 `options` 对象？让我们考虑一下。我们知道我们的 `ConfigModule` 基本上是一个提供和导出可注入服务的主机 - `ConfigService` - 以供其他提供者使用。实际上是我们的 `ConfigService` 需要读取 `options` 对象来定制其行为。让我们假设暂时我们知道如何以某种方式将 `options` 从 `register()` 方法中获取到 `ConfigService`。有了这个假设，我们可以对服务进行一些更改，使其行为基于 `options` 对象中的属性进行定制。（注意：暂时以来，我们还没有实际确定如何传递它，我们将硬编码 `options`。我们一会儿会修复这个）。

```typescript

import { Injectable } from '@nestjs/common';
import * as fs from 'node:fs';
import * as path from 'node:path';
import * as dotenv from 'dotenv';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}

```

现在我们的 `ConfigService` 知道如何在 `options` 中指定的文件夹中查找 `.env` 文件。

我们剩下的任务是以某种方式将 `options` 对象从 `register()` 步骤注入到我们的 `ConfigService`。当然，我们将使用依赖注入来做到这一点。这是一个关键点，所以确保您理解它。我们的 `ConfigModule` 正在提供 `ConfigService`。`ConfigService` 反过来依赖于仅在运行时提供的 `options` 对象。所以，在运行时，我们需要首先将 `options` 对象绑定到 Nest IoC 容器，然后让 Nest 将其注入到我们的 `ConfigService`。请记住自定义提供者章节，提供者可以[包括任何值](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers)，不仅仅是服务，所以我们使用依赖注入来处理简单的 `options` 对象是没问题的。

让我们首先解决将 options 对象绑定到 IoC 容器的问题。我们在静态 `register()` 方法中这样做。请记住，我们正在动态构造一个模块，模块的一个属性是其提供者列表。所以我们需要做的是将我们的 options 对象定义为提供者。这将使其可注入到 `ConfigService`，我们将在下一步中利用这一点。在下面的代码中，请注意 `providers` 数组：

```typescript

import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}

```

现在我们可以通过将 `'CONFIG_OPTIONS'` 提供者注入到 `ConfigService` 来完成这个过程。回想一下，当我们使用非类令牌定义提供者时，我们需要使用 `@Inject()` 装饰器[如这里所述](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)。

```typescript

import * as fs from 'node:fs';
import * as path from 'node:path';
import * as dotenv from 'dotenv';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}

```

最后一点说明：为简单起见，我们上面使用了基于字符串的注入令牌（`'CONFIG_OPTIONS'`），但最佳实践是将它定义为单独文件中的常量（或 `Symbol`），并导入该文件。例如：

```typescript

export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';

```

### Example  示例

本章代码的完整示例可以[在这里](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)找到。

### Community guidelines  社区指南

您可能已经在一些 `@nestjs/` 包中看到了 `forRoot`、`register` 和 `forFeature` 等方法的使用，并可能想知道所有这些方法的区别是什么。对此没有硬性规定，但 `@nestjs/` 包试图遵循这些指南：

创建具有以下方法的模块时：

`register`，您期望使用特定配置配置动态模块，仅供调用模块使用。例如，使用 Nest 的 `@nestjs/axios`：`HttpModule.register({ baseUrl: 'someUrl' })`。如果在另一个模块中使用 `HttpModule.register({ baseUrl: 'somewhere else' })`，它将具有不同的配置。您可以为任意数量的模块执行此操作。

`forRoot`，您期望配置一次动态模块并在多个地方重用该配置（尽管可能不知不觉，因为它被抽象化了）。这就是为什么您有一个 `GraphQLModule.forRoot()`、一个 `TypeOrmModule.forRoot()` 等。

`forFeature`，您期望使用动态模块的 `forRoot` 的配置，但需要根据调用模块的需求修改一些特定配置（即，此模块应该访问哪个存储库，或记录器应该使用的上下文。）

所有这些通常也有它们的 `async` 对应项，如 `registerAsync`、`forRootAsync` 和 `forFeatureAsync`，意思相同，但也使用 Nest 的依赖注入进行配置。

### Configurable module builder  可配置模块构建器

由于手动创建高度可配置的动态模块，这些模块暴露 `async` 方法（`registerAsync`、`forRootAsync` 等）相当复杂，特别是对于新手，Nest 暴露了 `ConfigurableModuleBuilder` 类，该类促进了这个过程，并让您只需几行代码就可以构造一个模块"蓝图"。

例如，让我们采用上面使用的示例（`ConfigModule`）并将其转换为使用 `ConfigurableModuleBuilder`。在开始之前，让我们确保我们创建了一个专用接口，表示我们的 `ConfigModule` 接受的选项。

```typescript

export interface ConfigModuleOptions {
  folder: string;
}

```

有了这个，让我们在现有 `config.module.ts` 文件旁边创建一个新的专用文件，并将其命名为 `config.module-definition.ts`。在这个文件中，让我们利用 `ConfigurableModuleBuilder` 来构造 `ConfigModule` 定义。

config.module-definition.ts

```typescript

import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();

```

现在让我们打开 `config.module.ts` 文件并修改其实现以利用自动生成的 `ConfigurableModuleClass`：

```typescript

import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}

```

扩展 `ConfigurableModuleClass` 意味着 `ConfigModule` 现在不仅提供 `register` 方法（像以前使用自定义实现一样），还提供 `registerAsync` 方法，该方法允许消费者异步配置该模块，例如，通过提供异步工厂：

```typescript

@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // 或者也可以：
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...任何额外依赖...]
    // }),
  ],
})
export class AppModule {}

```

`registerAsync` 方法接受以下对象作为参数：

```typescript

{
  /**
   * 解析为将作为提供者实例化的类的注入令牌。
   * 该类必须实现相应的接口。
   */
  useClass?: Type<
    ConfigurableModuleOptionsFactory<ModuleOptions, FactoryClassMethodKey>
  >;
  /**
   * 返回配置模块的选项的函数（或解析为选项的 Promise）。
   */
  useFactory?: (...args: any[]) => Promise<ModuleOptions> | ModuleOptions;
  /**
   * 工厂可能注入的依赖项。
   */
  inject?: FactoryProvider['inject'];
  /**
   * 解析为现有提供者的注入令牌。该提供者必须实现
   * 相应的接口。
   */
  useExisting?: Type<
    ConfigurableModuleOptionsFactory<ModuleOptions, FactoryClassMethodKey>
  >;
}

```

让我们逐一介绍上述属性：

- `useExisting` - `useClass` 的变体，允许您使用现有提供者而不是指示 Nest 创建类的实例。当您想要使用已在模块中注册的提供者时，这很有用。请记住，该类必须实现与 `useClass` 中使用的相同接口（因此它必须提供 `create()` 方法，除非您覆盖默认方法名，请参见下面的[自定义方法键](https://docs.nestjs.com/fundamentals/dynamic-modules#custom-method-key)部分）。
- `useClass` - 将作为提供者实例化的类。该类必须实现相应的接口。通常，这是提供返回配置对象的 `create()` 方法的类。有关此内容的更多信息，请参见下面的[自定义方法键](https://docs.nestjs.com/fundamentals/dynamic-modules#custom-method-key)部分。
- `inject` - 将注入到工厂函数中的依赖项数组。依赖项的顺序必须与工厂函数中参数的顺序匹配。
- `useFactory` - 返回配置对象的函数。它可以是同步的或异步的。要将依赖项注入到工厂函数中，请使用 `inject` 属性。我们在上面的示例中使用了这个变体。

始终选择上述选项之一（`useFactory`、`useClass` 或 `useExisting`），因为它们是互斥的。

最后，让我们更新 `ConfigService` 类以注入生成的模块选项提供者，而不是我们迄今为止使用的 `'CONFIG_OPTIONS'`。

```typescript

@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}

```

### Custom method key  自定义方法键

`ConfigurableModuleClass` 默认提供 `register` 及其对应项 `registerAsync` 方法。要使用不同的方法名，请使用 `ConfigurableModuleBuilder#setClassMethodName` 方法，如下所示：

config.module-definition.ts

```typescript

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();

```

这个构造将指示 `ConfigurableModuleBuilder` 生成一个类，该类暴露 `forRoot` 和 `forRootAsync` 而不是 `register` 和 `registerAsync`。示例：

```typescript

@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- 注意使用 "forRoot" 而不是 "register"
    // 或者也可以：
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...任何额外依赖...]
    // }),
  ],
})
export class AppModule {}

```

### Custom options factory class  自定义选项工厂类

由于 `registerAsync` 方法（或 `forRootAsync` 或任何其他名称，取决于配置）让消费者传递解析为模块配置的提供者定义，库消费者可能潜在地提供一个用于构造配置对象的类。

```typescript

@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}

```

这个类，默认情况下，必须提供返回模块配置对象的 `create()` 方法。但是，如果您的库遵循不同的命名约定，您可以更改该行为并指示 `ConfigurableModuleBuilder` 期望不同的方法，例如 `createConfigOptions`，使用 `ConfigurableModuleBuilder#setFactoryMethodName` 方法：

config.module-definition.ts

```typescript

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();

```

现在，`ConfigModuleOptionsFactory` 类必须暴露 `createConfigOptions` 方法（而不是 `create`）：

```typescript

@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- 这个类必须提供 "createConfigOptions" 方法
    }),
  ],
})
export class AppModule {}

```

### Extra options  额外选项

有一些边缘情况，您的模块可能需要接受额外选项，这些选项确定它应该如何表现（这样的选项的一个很好的例子是 `isGlobal` 标志 - 或只是 `global`），同时不应该包含在 `MODULE_OPTIONS_TOKEN` 提供者中（因为它们与模块内注册的服务/提供者无关，例如，`ConfigService` 不需要知道其宿主模块是否注册为全局模块）。

在这种情况下，可以使用 `ConfigurableModuleBuilder#setExtras` 方法。请参见以下示例：

```typescript

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>()
    .setExtras(
      {
        isGlobal: true,
      },
      (definition, extras) => ({
        ...definition,
        global: extras.isGlobal,
      }),
    )
    .build();

```

在上面的示例中，传递给 `setExtras` 方法的第一个参数是一个包含"额外"属性的默认值的对象。第二个参数是一个函数，它接受自动生成的模块定义（带有 `provider`、`exports` 等）和 `extras` 对象，后者表示额外属性（由消费者指定或默认值）。这个函数的返回值是一个修改后的模块定义。在这个特定示例中，我们将 `extras.isGlobal` 属性分配给模块定义的 `global` 属性（这反过来确定模块是否是全局的，更多信息请[点击这里](https://docs.nestjs.com/modules#dynamic-modules)）。

现在在消费这个模块时，可以传递额外的 `isGlobal` 标志，如下所示：

```typescript

@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}

```

但是，由于 `isGlobal` 被声明为"额外"属性，它不会在 `MODULE_OPTIONS_TOKEN` 提供者中可用：

```typescript

@Injectable()
export class ConfigService {
  constructor(
    @Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions,
  ) {
    // "options" 对象将不会有 "isGlobal" 属性
    // ...
  }
}

```

### Extending auto-generated methods  扩展自动生成的方法

如果需要，可以扩展自动生成的静态方法（`register`、`registerAsync` 等），如下所示：

```typescript

import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import {
  ConfigurableModuleClass,
  ASYNC_OPTIONS_TYPE,
  OPTIONS_TYPE,
} from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // 这里是您的自定义逻辑
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // 这里是您的自定义逻辑
      ...super.registerAsync(options),
    };
  }
}

```

注意使用 `OPTIONS_TYPE` 和 `ASYNC_OPTIONS_TYPE` 类型，它们必须从模块定义文件中导出：

```typescript

export const {
  ConfigurableModuleClass,
  MODULE_OPTIONS_TOKEN,
  OPTIONS_TYPE,
  ASYNC_OPTIONS_TYPE,
} = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();

```