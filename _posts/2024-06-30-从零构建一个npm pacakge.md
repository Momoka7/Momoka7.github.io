---
layout: post
title: "从零构建一个npm package"
subtitle: "构建自己的npm package"
date: 2024-06-30 11:31:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - TypeScript
  - npm
  - vite
---

# 使用 TypeScript 构建

`TypeScript`相比较`vanilla javascript`，虽然两者编译后都是`.js`代码，但是`ts`构建的包可以有`.d.ts`类型声明。结合现代开发环境，这种类型提示可以给开发者一个较为良好的开发体验。

## 1. 设置项目

1. 首先使用`npm`构建一个空项目
2. 接下来，安装 TypeScript 和相关工具：
   ```
   npm install typescript --save-dev
   npm install @types/node --save-dev
   ```
3. 创建 tsconfig.json
   ```
   npx tsc --init
   ```
4. 根据需要修改 `tsconfig.json` 文件,常见的配置如下：
   ```json
   {
     "compilerOptions": {
       "target": "ES5",
       "module": "CommonJS",
       "declaration": true,
       "outDir": "./dist",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true
     },
     "include": ["src"],
     "exclude": ["node_modules", "**/*.test.ts"]
   }
   ```

## 2. 编写代码

在项目根目录下创建 `src `文件夹，并在其中编写 TypeScript 代码。例如，创建一个简单的工具函数：

```typescript
// src/index.ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

## 3. 测试代码（可选）

1. 可以使用 Jest 进行单元测试：
   ```
   npm install jest ts-jest @types/jest --save-dev
   ```
2. 创建 Jest 配置文件
   ```
   npx ts-jest config:init
   ```
3. 编写测试，在 `src `目录下创建 `__tests__` 文件夹，并在其中编写测试文件：

   ```typescript
   // src/__tests__/index.test.ts
   import { greet } from "../index";

   test("greet function", () => {
     expect(greet("World")).toBe("Hello, World!");
   });
   ```

4. 运行测试，在 `package.json` 文件中添加以下脚本：
   ```json
   "scripts": {
     "test": "jest"
   }
   ```
5. 然后运行测试：`npm test`

## 4. 打包

1. 编译 TypeScript 代码：在 package.json 文件中添加以下脚本：
   ```json
   "scripts": {
     "build": "tsc",
     "test": "jest"
   }
   ```
2. 然后编译 TypeScript 代码：`npm run build`
3. 创建入口文件：确保在 `package.json` 文件中指定了入口文件：
   ```json
   "main": "dist/index.js",
   "types": "dist/index.d.ts"
   ```

## 5. 发布

1. 首先，确保已经登录到 npm：`npm login`
2. 然后发布 package：`npm publish`

## 6. 其他

### 依赖了其他库的情况

所开发的包是`a-package`的插件或扩展的情况下，使用`peerDependencies`。

在`package.json`中使用`peerDependencies`列出需要使用者安装的包，如：

```
"peerDependencies": {
  "react": "^17.0.0"
}
```

若所开发的包直接依赖于`a-package`，完全独立于应用程序，并且只在内部使用，那么应该加入`dependencies`中。

# 结合 vite 构建工具

## 1. 创建项目

`npm create vite`，若要使用`ts`开发，可以在后续选择`ts`模板

## 2. 编写代码

```typescript
// src/index.ts：视为某些模块
export const echo = (str: string) => {
  console.log(str);
  return str;
};
// src/main.ts：视为vite.config.ts配置中的入口文件(见后续)
export * from "./index";
```

## 3. 配置 vite.config.ts

新建`vite.config.ts`文件，一个常用的配置如下：

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  build: {
    outDir: "dist",
    target: "es2020",
    lib: {
      entry: "./src/main.ts", //指定程序入口
      formats: ["es", "cjs"], //所支持的模块格式
      //输出文件名，若不设置默认为项目名称，不带后缀（可以使用函数进行更详细的配置）
      fileName: "index",
    },
  },
});
```

## 4. 输出类型声明文件（可选）

有两种方式可以输出`ts`的`.d.ts`类型声明文件

### 1. 使用`tsc`输出`.d.ts`文件

和上述直接使用`tsc`构建并输出方法类似，在`tsconfig.json`中主要配置好如下选项即可：

```json
  //其他配置
  "declaration": true,
  "outDir": "./dist",// 声明文件输出的目录
  "emitDeclarationOnly": true,// 只输出声明文件，不转译ts文件
  //其他配置
```

在`package.json` 中添加脚本`"build": "vite build && tsc"`并`npm run build`即可。

> 注意默认的`"build": "tsc && vite build"`脚本中`tsc`的指令不会生效，需要将`tsc`放到后面

### 2. 使用`vite-plugin-dts`插件

首先，安装`vite-plugin-dts`依赖：

```
npm i vite-plugin-dts -D
```

然后，在 vite 中引入该插件，并注册，修改`vite.config.ts`如下：

```typescript
import { defineConfig } from "vite";
import dts from "vite-plugin-dts";

export default defineConfig({
  plugins: [dts()],
  build: {
    outDir: "dist",
    target: "es2020",
    lib: {
      entry: "./src/main.ts",
      formats: ["es", "cjs"],
      fileName: "main",
    },
  },
});
```

`dist` 目录输出如下：

![截图](/img/in-post/npm-pacakge/d63d75fcab0797c09163f99824e177c0.png)

> **注意**：构建文件时需要注意文件名、类型声明文件名、入口文件名一致，否则可能出现声明文件不生效的情况。

## 5. 配置`package.json`

若有声明文件，需要注意添加`types`字段

```json
  "type": "module",
  "main": "dist/main.cjs",
  "module": "dist/main.js",
  "types": "dist/main.d.ts",
```
