# Controllers（NestJS 控制器）

来源：[NestJS Controllers 文档](https://docs.nestjs.com/controllers)

## Controllers  控制器

控制器负责处理传入的请求并将响应返回给客户端。

控制器的目的是处理应用特定的请求。路由机制确定哪个控制器处理每个请求。通常，一个控制器有多个路由，每个路由可以执行不同的操作。

要创建一个基本的控制器，我们使用类和装饰器。装饰器将类与必要的元数据链接起来，允许 Nest 创建连接请求到相应控制器的路由映射。

> **提示** 要快速创建一个带有内置验证的 CRUD 控制器，你可以使用 CLI 的 CRUD 生成器：`nest g resource [name]`。

### Routing  路由

在下面的示例中，我们将使用 `@Controller()` 装饰器，这是定义基本控制器所必需的。我们将为它指定一个可选的路由路径前缀 `cats`。在 `@Controller()` 装饰器中使用路径前缀有助于将相关路由分组在一起，并减少重复代码。例如，如果我们想将管理 cat 实体的路由分组在 `/cats` 路径下，我们可以在 `@Controller()` 装饰器中指定 `cats` 路径前缀。这样，我们就不需要在文件中每个路由中重复该路径部分。

`cats.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

> **提示** 要使用 CLI 创建控制器，只需执行 `nest g controller [name]` 命令。

`@Get()` HTTP 请求方法装饰器放在 `findAll()` 方法之前，告诉 Nest 为特定的 HTTP 请求端点创建处理器。该端点由 HTTP 请求方法（本例中为 GET）和路由路径确定。那么路由路径是什么？处理器的路由路径是由（可选的）控制器路径前缀和方法装饰器中指定的任何路径字符串组合而成的。由于我们为每个路由设置了前缀（`cats`），且在方法装饰器中没有添加任何特定路径，Nest 将映射 `GET /cats` 请求到此处理器。

如前所述，路由路径包括可选的控制器路径前缀和方法装饰器中指定的任何路径字符串。例如，如果控制器前缀是 `cats`，方法装饰器是 `@Get('breed')`，则生成的路由将是 `GET /cats/breed`。

在我们的示例中，当对该端点发出 GET 请求时，Nest 将请求路由到用户定义的 `findAll()` 方法。注意，我们选择的方法名称在这里是完全任意的。虽然我们必须声明一个方法来绑定路由，但 Nest 不会对方法名称附加任何特定意义。

此方法将返回 200 状态码以及相应的响应（本例中只是一个字符串）。为什么会这样？为了解释这一点，我们首先需要介绍 Nest 用于操纵响应的两种不同选项的概念：

| 标准（推荐） | 使用此内置方法，当请求处理器返回 JavaScript 对象或数组时，它将自动序列化为 JSON。当它返回 JavaScript 原始类型（例如 `string`、`number`、`boolean`）时，Nest 将只发送值而不会尝试序列化。此外，响应的状态码默认总是 200，除了 POST 请求使用 201。我们可以轻松更改此行为，在处理器级别添加 `@HttpCode(...)` 装饰器（见状态码）。 |
| --- | --- |
| 库特定 | 我们可以使用库特定的（例如 Express）响应对象，通过 `@Res()` 装饰器注入（例如 `findAll(@Res() response)`）。使用此方法，你可以访问该对象暴露的本机响应处理方法。例如，使用 Express，你可以使用代码如 `response.status(200).send()` 构造响应。 |

> **警告** Nest 检测到处理器是否使用了 `@Res()` 或 `@Next()`，表示你选择了库特定选项。如果同时使用这两种方法，标准方法会自动禁用此单个路由，且不会按预期工作。要同时使用这两种方法（例如，通过注入响应对象仅设置 cookie/headers 但让框架处理其余部分），你必须在 `@Res({ passthrough: true })` 装饰器中设置 `passthrough` 选项为 `true`。

### Request object  请求对象

处理器通常需要访问客户端请求的详细信息。Nest 提供从底层平台（默认 Express）访问请求对象的方法。你可以通过指示 Nest 在处理器签名中使用 `@Req()` 装饰器注入它来访问请求对象。

`cats.controller.ts`

```typescript
import { Controller, Get, Req } from '@nestjs/common';
import type { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
```

```typescript
import { Controller, Bind, Get, Req } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  @Bind(Req())
  findAll(request) {
    return 'This action returns all cats';
  }
}
```

> **提示** 要利用 Express 类型化（像上面的 `request: Request` 参数示例），请确保安装 `@types/express` 包。

请求对象代表 HTTP 请求，包含查询字符串、参数、HTTP 头和体的属性（更多请阅读[这里](https://expressjs.com/en/api.html#req)）。在大多数情况下，你不需要手动访问这些属性。相反，你可以使用像 `@Body()` 或 `@Query()` 这样的专用装饰器，这些是开箱即用的。下面是提供的装饰器及其对应的平台特定对象列表。

| `@Request(), @Req()` | `req` |
| --- | --- |
| `@Response(), @Res()`* | `res` |
| `@Next()` | `next` |
| `@Session()` | `req.session` |
| `@Param(key?: string)` | `req.params`/`req.params[key]` |
| `@Body(key?: string)` | `req.body`/`req.body[key]` |
| `@Query(key?: string)` | `req.query`/`req.query[key]` |
| `@Headers(name?: string)` | `req.headers`/`req.headers[name]` |
| `@Ip()` | `req.ip` |
| `@HostParam()` | `req.hosts` |

* 为跨底层 HTTP 平台（例如 Express 和 Fastify）的类型兼容性，Nest 提供了 `@Res()` 和 `@Response()` 装饰器。`@Res()` 只是 `@Response()` 的别名。两者都直接暴露底层本机平台响应对象接口。当使用它们时，你应该导入底层库的类型（例如 `@types/express`）以充分利用。此外，当你在方法处理器中注入 `@Res()` 或 `@Response()` 时，你将 Nest 置于库特定模式，该处理器负责管理响应。当这样做时，你必须通过调用响应对象发出某种响应（例如 `res.json(...)` 或 `res.send(...)`），否则 HTTP 服务器将挂起。

> **提示** 要了解如何创建自己的自定义装饰器，请访问[此章节](https://docs.nestjs.com/custom-decorators)。

### Resources  资源

之前，我们定义了一个端点来获取 cats 资源（GET 路由）。我们通常还需要提供一个创建新记录的端点。为此，让我们创建 POST 处理器：

`cats.controller.ts`

```typescript
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

```typescript
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create() {
    return 'This action adds a new cat';
  }

  @Get()
  findAll() {
    return 'This action returns all cats';
  }
}
```

就是这么简单。Nest 为所有标准 HTTP 方法提供装饰器：`@Get()`、`@Post()`、`@Put()`、`@Delete()`、`@Patch()`、`@Options()` 和 `@Head()`。此外，`@All()` 定义一个处理所有方法的端点。

### Route wildcards  路由通配符

基于模式的路由在 NestJS 中也是受支持的。例如，星号（`*`）可以用作通配符，在路径末尾匹配任意字符组合。在下面的示例中，无论后续有多少字符，`findAll()` 方法将为任何以 `abcd/` 开头的路由执行。

```typescript
@Get('abcd/*')
findAll() {
  return 'This route uses a wildcard';
}
```

`'abcd/*'` 路由路径将匹配 `abcd/`、`abcd/123`、`abcd/abc` 等。连字符（`-`）和点号（`.`）由字符串路径按字面解释。

此方法在 Express 和 Fastify 上都有效。然而，在最新的 Express 版本（v5）中，路由系统变得更严格。在纯 Express 中，你必须使用命名通配符才能让路由工作——例如 `abcd/*splat`，其中 `splat` 只是通配符参数的名称，没有特殊意义。你可以随意命名它。那说回来，由于 Nest 为 Express 提供了兼容性层，你仍然可以使用星号（`*`）作为通配符。

当在路由中间使用星号时，Express 需要命名通配符（例如 `ab{*splat}cd`），而 Fastify 根本不支持它们。

### Status code  状态码

如前所述，响应的默认状态码总是 200，除了 POST 请求使用 201。你可以使用处理器级别的 `@HttpCode(...)` 装饰器轻松更改此行为。

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

> **提示** 从 `@nestjs/common` 包导入 `HttpCode`。

通常，你的状态码不是静态的，而是取决于各种因素。在这种情况下，你可以使用库特定的响应（使用 `@Res()` 注入）对象（或在错误情况下抛出异常）。

### Response headers  响应头

要指定自定义响应头，你可以使用 `@Header()` 装饰器或库特定的响应对象（并直接调用 `res.header()`）。

```typescript
@Post()
@Header('Cache-Control', 'no-store')
create() {
  return 'This action adds a new cat';
}
```

> **提示** 从 `@nestjs/common` 包导入 `Header`。

### Redirection  重定向

要将响应重定向到特定 URL，你可以使用 `@Redirect()` 装饰器或库特定的响应对象（并直接调用 `res.redirect()`）。

`@Redirect()` 接受两个参数，`url` 和 `statusCode`，两者都是可选的。`statusCode` 的默认值是 `302`（`Found`），如果省略。

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

> **提示** 有时你可能想动态确定 HTTP 状态码或重定向 URL。通过返回遵循 `HttpRedirectResponse` 接口（来自 `@nestjs/common`）的对象来实现。

返回的值将覆盖传递给 `@Redirect()` 装饰器的任何参数。例如：

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

### Route parameters  路由参数

具有静态路径的路由在你需要接受动态数据作为请求一部分时不起作用（例如 `GET /cats/1` 来获取 id 为 `1` 的 cat）。要定义带有参数的路由，你可以在路由路径中添加路由参数 token 来捕获 URL 中的动态值。下面的 `@Get()` 装饰器示例中的路由参数 token 说明了这种方法。然后，可以使用 `@Param()` 装饰器访问这些路由参数，该装饰器应添加到方法签名中。

> **提示** 带有参数的路由应该在任何静态路径之后声明。这防止参数化路径拦截目的地为静态路径的流量。

```typescript
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

```typescript
@Get(':id')
@Bind(Param())
findOne(params) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()` 装饰器用于装饰方法参数（上面的示例中是 `params`），使路由参数作为该装饰的方法参数的属性在方法内可访问。如代码所示，你可以通过引用 `params.id` 来访问 `id` 参数。或者，你可以传递特定的参数 token 给装饰器，并在方法体内直接按名称引用路由参数。

> **提示** 从 `@nestjs/common` 包导入 `Param`。

```typescript
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

```typescript
@Get(':id')
@Bind(Param('id'))
findOne(id) {
  return `This action returns a #${id} cat`;
}
```

### Sub-domain routing  子域路由

`@Controller` 装饰器可以接受 `host` 选项，要求传入请求的 HTTP 主机匹配某个特定值。

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

> **警告** 由于 Fastify 不支持嵌套路由器，如果你使用子域路由，建议使用默认的 Express 适配器。

类似于路由 `path`，`host` 选项可以使用 token 来捕获主机名中该位置的动态值。下面的 `@Controller()` 装饰器示例中的主机参数 token 演示了此用法。这样声明的主机参数可以使用 `@HostParam()` 装饰器访问，该装饰器应添加到方法签名中。

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

### State sharing  状态共享

对于来自其他编程语言的开发者来说，在 Nest 中几乎所有东西都跨传入请求共享可能会令人惊讶。这包括数据库连接池、全局状态的单例服务等资源。重要的是要理解 Node.js 不使用请求/响应多线程无状态模型，其中每个请求由单独的线程处理。因此，在 Nest 中使用单例实例对我们的应用是完全安全的。

话虽如此，在某些特定边缘情况下，为控制器提供基于请求的生命周期可能是必要的。例如 GraphQL 应用中的按请求缓存、请求跟踪或实现多租户。你可以[在这里](https://docs.nestjs.com/fundamentals/injection-scopes)了解更多关于控制注入范围的信息。

### Asynchronicity  异步性

我们热爱现代 JavaScript，特别是它对异步数据处理的强调。这就是为什么 Nest 完全支持 `async` 函数。每个 `async` 函数必须返回 `Promise`，这允许你返回 Nest 可以自动解析的延迟值。这里是一个示例：

`cats.controller.ts`

```typescript
@Get()
async findAll(): Promise<any[]> {
  return [];
}
```

```typescript
@Get()
async findAll() {
  return [];
}
```

这段代码是完全有效的。但 Nest 更进一步，允许路由处理器返回 RxJS 可观察流作为响应。Nest 将在内部处理订阅，并在流完成时解析最终发出的值。

`cats.controller.ts`

```typescript
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
```

```typescript
@Get()
findAll() {
  return of([]);
}
```

两种方法都是有效的，你可以选择最适合你需求的一种。

### Request payloads  请求负载

在之前的示例中，POST 路由处理器没有接受任何客户端参数。让我们通过添加 `@Body()` 装饰器来修复这个问题。

在此之前（如果你使用 TypeScript），我们需要定义 DTO（数据传输对象）schema。DTO 是一个对象，它指定数据应如何通过网络发送。我们可以使用 TypeScript 接口或简单类定义 DTO schema。然而，我们建议在这里使用类。为什么？类是 JavaScript ES6 标准的一部分，所以它们在编译的 JavaScript 中作为真实实体保持完整。相比之下，TypeScript 接口在转译期间被移除，这意味着 Nest 无法在运行时引用它们。这很重要，因为像 Pipes 这样的特性依赖于在运行时访问变量的元类型，这只有类才能做到。

让我们创建 `CreateCatDto` 类：

`create-cat.dto.ts`

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

它只有三个基本属性。此后我们可以在 `CatsController` 中使用新创建的 DTO：

`cats.controller.ts`

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

```typescript
@Post()
@Bind(Body())
async create(createCatDto) {
  return 'This action adds a new cat';
}
```

> **提示** 我们的 `ValidationPipe` 可以过滤掉不应被方法处理器接收的属性。在这种情况下，我们可以白名单可接受的属性，任何不在白名单中的属性都会自动从结果对象中剥离。在 `CreateCatDto` 示例中，我们的白名单是 `name`、`age` 和 `breed` 属性。了解更多[这里](https://docs.nestjs.com/techniques/validation#stripping-properties)。

### Query parameters  查询参数

在路由中处理查询参数时，你可以使用 `@Query()` 装饰器从传入请求中提取它们。让我们看看这在实践中是如何工作的。

考虑一个我们想要根据 `age` 和 `breed` 等查询参数过滤 cat 列表的路由。首先在 `CatsController` 中定义查询参数：

`cats.controller.ts`

```typescript
@Get()
async findAll(@Query('age') age: number, @Query('breed') breed: string) {
  return `This action returns all cats filtered by age: ${age} and breed: ${breed}`;
}
```

在这个示例中，`@Query()` 装饰器用于从查询字符串中提取 `age` 和 `breed` 的值。例如，对以下请求：

```
GET /cats?age=2&breed=Persian
```

将导致 `age` 为 `2`，`breed` 为 `Persian`。

如果你的应用需要处理更复杂的查询参数，例如嵌套对象或数组：

```
?filter[where][name]=John&filter[where][age]=30
?item[]=1&item[]=2
```

你需要配置你的 HTTP 适配器（Express 或 Fastify）以使用适当的查询解析器。在 Express 中，你可以使用 `extended` 解析器，它允许丰富的查询对象：

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
app.set('query parser', 'extended');
```

在 Fastify 中，你可以使用 `querystringParser` 选项：

```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({
    querystringParser: (str) => qs.parse(str),
  }),
);
```

> **提示** `qs` 是一个支持嵌套和数组的查询字符串解析器。你可以使用 `npm install qs` 安装它。

### Handling errors  处理错误

有一个单独的章节关于处理错误（即使用异常）[这里](https://docs.nestjs.com/exception-filters)。

### Full resource sample  完整资源示例

下面是一个示例，演示了使用多个可用装饰器创建基本控制器。该控制器提供了几个方法来访问和操纵内部数据。

`cats.controller.ts`

```typescript
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```

```typescript
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat`;
  }
}
```

> **提示** Nest CLI 提供了一个生成器（schematic），它自动创建所有样板代码，为你节省手动操作并改善整体开发者体验。了解更多关于此功能的[这里](https://docs.nestjs.com/recipes/crud-generator)。

### Getting up and running  开始运行

即使 `CatsController` 完全定义，Nest 仍然不知道它，不会自动创建类的实例。

控制器必须始终属于模块，这就是为什么我们在 `@Module()` 装饰器中包含 `controllers` 数组。由于我们除了根 `AppModule` 之外还没有定义任何其他模块，我们将使用它来注册 `CatsController`：

`app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

我们使用 `@Module()` 装饰器将元数据附加到模块类，现在 Nest 可以轻松确定需要挂载哪些控制器。

### Library-specific approach  库特定方法

到目前为止，我们已经介绍了 Nest 操纵响应的标准方式。另一种方法是使用库特定的响应对象。要注入特定的响应对象，我们可以使用 `@Res()` 装饰器。为了突出差异，让我们重写 `CatsController`：

```typescript
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

```typescript
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

虽然这种方法有效并提供更多灵活性，通过给你对响应对象的完全控制（例如头操纵和访问库特定功能），但应谨慎使用。一般来说，这种方法不如标准方法清晰，并有一些缺点。主要缺点是你的代码变得平台依赖，因为不同的底层库可能有不同的响应对象 API。此外，它可以使测试更具挑战性，因为你需要模拟响应对象等。

此外，通过使用这种方法，你会失去与依赖标准响应处理的 Nest 功能的兼容性，例如拦截器和 `@HttpCode()`/`@Header()` 装饰器。要解决这个问题，你可以启用 `passthrough` 选项如下：

```typescript
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
```

```typescript
@Get()
@Bind(Res({ passthrough: true }))
findAll(res) {
  res.status(HttpStatus.OK);
  return [];
}
```

通过这种方法，你可以与本机响应对象交互（例如，基于特定条件设置 cookie 或头），同时仍然让框架处理其余部分。