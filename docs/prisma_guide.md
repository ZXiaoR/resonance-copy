# Prisma 核心概念入门

本文档根据我们的对话内容整理而成，旨在帮助你快速理解 Prisma 的几个核心概念。

## 1. `schema.prisma`：核心配置文件

简单来说，`schema.prisma` 文件是 **Prisma 框架的核心**。它是你项目中数据库结构的**唯一“事实来源” (single source of truth)**。

这个文件的主要作用有三个：

### a. 定义数据模型 (Data Models)
这是最核心的部分，它会直接映射成数据库里的表（Table）。

**示例：**
```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
}
```

### b. 指定数据库连接 (Datasource)
这部分告诉 Prisma 你的数据库类型和连接信息。

**示例：**
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL") // 从环境变量读取连接字符串
}
```

### c. 配置 Prisma 客户端生成器 (Generator)
这部分告诉 Prisma 需要生成哪种类型的客户端库，以便在你的应用代码中进行类型安全的数据查询。

**示例：**
```prisma
generator client {
  provider = "prisma-client-js"
}
```

---

## 2. 数据库迁移：`prisma migrate`

每一次你对 `schema.prisma` 文件中的表结构做出修改，都应该执行一次新的迁移。这就像是**为你的数据库结构进行版本控制**。

### 命令：`npx prisma migrate dev --name <MIGRATION_NAME>`

-   `npx prisma migrate dev`: 在开发环境下，对比 `schema.prisma` 与数据库的差异，生成并应用新的迁移。
-   `--name <MIGRATION_NAME>`: 为本次迁移起一个有描述性的名字，例如 `add_user_age`。

**流程：**
1.  修改 `schema.prisma` 文件。
2.  执行 `npx prisma migrate dev --name <...>`。
3.  Prisma 自动创建新的迁移文件并更新数据库。

---

## 3. 项目初始化：`prisma init`

`npx prisma init` 是你在一个项目中**第一次**使用 Prisma 时要运行的**初始化命令**。每个项目**只运行一次**。

### 命令：`npx prisma init`

执行后，它会：
1.  创建 `prisma/schema.prisma` 文件，并填入基础模板。
2.  创建 `.env` 文件，用于存放数据库连接字符串等环境变量，并自动将其加入 `.gitignore`。

---

## 4. `init` vs. `migrate`：核心区别总结

| 命令 | `npx prisma init` | `npx prisma migrate dev` |
| :--- | :--- | :--- |
| **目的** | **初始化**项目配置 | **迁移**数据库结构 |
| **频率** | 每个项目**仅一次** | **每次**修改表结构后 |
| **作用对象**| 本地文件系统 | 数据库 |
| **结果** | 创建 `schema.prisma` 和 `.env` 文件 | 创建新的迁移文件并更新数据库 |
| **比喻** | **准备蓝图** | **根据蓝图施工** |

---

## 5. `prisma generate`：生成类型安全的客户端

`src/generated/prisma` 目录（具体路径取决于配置）包含了 **Prisma Client**，它是由 Prisma CLI **自动生成**的。

### 它是如何以及为何生成的？

1.  **配置来源**：生成过程的依据是 `schema.prisma` 文件中的 `generator` 配置块。你可以使用 `output` 属性指定生成客户端的位置。

    ```prisma
    generator client {
      provider = "prisma-client-js"
      // output   = "../src/generated/prisma" // 可选：指定输出目录
    }
    ```

2.  **触发命令**：
    *   `npx prisma generate`：专门用于（重新）生成客户端。
    *   `npx prisma migrate dev`：在迁移成功后会自动调用 `generate`，确保客户端与数据库结构同步。

3.  **核心目的：类型安全 (Type Safety)**
    这是 Prisma 最强大的功能。生成的客户端是为你当前项目**量身定做**的，它能让你的 TypeScript 代码完全理解你的数据库模型。

    **示例：**
    如果你在 schema 中有 `model User`，你的代码就会享受到：
    ```typescript
    import { PrismaClient } from '../generated/prisma'; // 路径取决于你的配置
    const prisma = new PrismaClient();

    const user = await prisma.user.findUnique({ where: { email: 'test@example.com' } });
    // 编辑器会知道 user.name、user.email 等所有属性及其类型
    // 如果你尝试访问 user.age (一个不存在的字段)，会立刻得到一个类型错误
    ```
    这可以在编码阶段就发现并避免大量的潜在错误。

---

## 6. 实例化PrismaClient的最佳实践 (`src/lib/db.ts`)

在实际项目中，我们不应该在代码的各个地方都去 `new PrismaClient()`。`src/lib/db.ts` 这样的文件就是为了解决这个问题而存在的，它确保了整个应用共享一个 Prisma Client 实例。

### 为什么需要这个文件？

1.  **避免开发环境下连接耗尽**：在 Next.js 等框架的开发模式下，"热重载"会频繁执行代码。如果没有特殊处理，每次重载都会创建一个新的数据库连接池，很快就会耗尽数据库的连接资源。
2.  **优化 Serverless 环境性能**：在 Vercel 等 Serverless 平台，函数是短暂执行的。为每个函数都启动一个完整的 Prisma 查询引擎效率低下。

### 解决方案解析

`src/lib/db.ts` 文件通过两种方式解决了上述问题：

1.  **单例模式 (Singleton Pattern)**：
    它利用 Node.js 的 `global` 对象（在热重载之间持久存在）来存储和复用 `PrismaClient` 实例，确保在开发过程中只有一个实例在运行。
    ```typescript
    // 伪代码示意
    const prisma = global.prisma || new PrismaClient();
    if (process.env.NODE_ENV !== 'production') global.prisma = prisma;
    ```

2.  **使用适配器 (Adapter)**：
    通过 `@prisma/adapter-pg`，它让 Prisma Client 在 Serverless 环境下使用更轻量的连接方式，提高了执行效率。
    ```typescript
    // 伪代码示意
    import { PrismaPg } from "@prisma/adapter-pg";
    const adapter = new PrismaPg({ connectionString: "..." });
    const prisma = new PrismaClient({ adapter });
    ```

**结论：** 整个项目都应该从 `src/lib/db.ts` 导入 `prisma` 实例，这是保证应用性能和稳定性的关键一步。

---

## 7. 环境变量管理：`src/lib/env.ts`

该文件的主要作用是**校验、管理并类型化**你项目中的环境变量。这样做可以确保你的应用在启动前就获得了所有必要的配置，避免因为缺少环境变量而导致运行时错误，并提供更好的开发体验（比如代码自动补全和类型检查）。

它主要使用了两个核心的库：

### a. `zod`
*   **作用**: 这是一个 TypeScript-first 的 schema 声明和验证库。你可以用它来定义任何数据的“形状”，从简单的字符串到复杂的嵌套对象。
*   **在此文件中**: `zod` 被用来为每一个环境变量定义一个验证规则。例如:
    *   `z.string().min(1)` 表示这个环境变量必须是一个长度至少为1的字符串。
    *   `z.enum(["sandbox", "production"])` 表示这个环境变量的值必须是 "sandbox" 或 "production" 两者之一。
    *   `z.url()` 表示这个环境变量必须是一个合法的 URL。

### b. `@t3-oss/env-nextjs`
*   **作用**: 这个库专门为 Next.js 应用设计，它利用 `zod` schema 来验证 `process.env` 中的环境变量。如果环境变量缺失或不符合 `zod` 定义的规则，它会在应用构建时就抛出错误，让你能及早发现问题。
*   **在此文件中**: `createEnv` 函数就是这个库的核心。它接收一个配置对象：
    *   `server`: 这里定义了所有**仅在服务器端**可用的环境变量。这是一种安全措施，可以防止敏感信息（如 API 密钥、数据库连接字符串）泄露到客户端浏览器。
    *   `experimental__runtimeEnv`: 它会去 `process.env` 中读取环境变量并使用 `zod` schema进行验证。
    *   `skipValidation`: 允许你在特定情况下（例如，设置了 `SKIP_ENV_VALIDATION=true`）跳过环境变量的验证。

**总结：**

`src/lib/env.ts` 文件通过 `zod` 定义了你的应用所依赖的环境变量的类型和规则，然后使用 `@t3-oss/env-nextjs` 在程序启动时对这些环境变量进行严格的校验，最后导出一个完全类型化的 `env` 对象，让你可以安全、方便地在代码中访问这些配置。
