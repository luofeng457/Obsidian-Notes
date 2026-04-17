# Webpack Plugin 核心原理与极简 Demo

Webpack Plugin 是扩展 Webpack 功能的核心方式，它基于 **Tapable 钩子系统**，让你能在 Webpack 编译的**任意阶段**插入自定义逻辑，实现 Loader 做不到的「全流程控制」。

---

## 一、核心原理（3 句话讲透）

### 1. 底层基石：Tapable 钩子系统

Webpack 内部用 `Tapable` 库管理所有生命周期，把编译流程拆成了几十个「钩子（Hook）」（比如 `compile`、`emit`、`done`）。 插件的本质就是：**在这些钩子上注册回调函数，当 Webpack 运行到对应阶段时，自动触发你的逻辑**。



### 2. 两个核心对象

写插件必须和这两个对象打交道：

|                 |                     |                                                  |                                     |
| --------------- | ------------------- | ------------------------------------------------ | ----------------------------------- |
| 对象              | 含义                  | 作用                                               | 常见钩子                                |
| **Compiler**    | 全局唯一的 Webpack 编译器实例 | 代表完整的 Webpack 环境，从启动到关闭只创建一次                     | `run`、`compile`、`done`、`beforeEmit` |
| **Compilation** | 单次编译的实例             | 每次文件变化触发重新编译时，都会创建新的 Compilation，包含当前的模块、依赖、输出资源 | `buildModule`、`optimize`、`emit`     |

### 3. 插件的标准结构

Webpack 插件是一个**包含** **`apply`** **方法的类**（或函数），`apply` 方法会在 Webpack 安装插件时被调用，接收 `compiler` 对象作为参数。

---

## 二、极简 Demo（从入门到实用）

### 前置准备

先创建一个简单的 Webpack 项目：

```bash
mkdir webpack-plugin-demo && cd webpack-plugin-demo
npm init -y
npm install webpack webpack-cli --save-dev
```

---

### Demo 1：Hello World 插件（理解钩子注册）

**功能**：在 Webpack 编译开始和结束时，打印自定义消息。

#### 1. 插件代码：`my-first-plugin.js`

```javascript
class MyFirstPlugin {
  // apply 方法是插件的入口，接收 compiler 对象
  apply(compiler) {
    // 1. 注册「编译开始」钩子（同步钩子，用 tap）
    compiler.hooks.compile.tap('MyFirstPlugin', () => {
      console.log('🚀 Webpack 开始编译啦！');
    });

    // 2. 注册「编译完成」钩子（异步钩子，用 tapAsync）
    compiler.hooks.done.tapAsync('MyFirstPlugin', (stats, callback) => {
      console.log('✅ Webpack 编译完成！');
      console.log(`📦 编译耗时：${stats.endTime - stats.startTime}ms`);
      // 异步钩子必须调用 callback 通知 Webpack 继续
      callback();
    });
  }
}

// 导出插件类
module.exports = MyFirstPlugin;
```

#### 2. Webpack 配置：`webpack.config.js`

```javascript
const path = require('path');
const MyFirstPlugin = require('./my-first-plugin');

module.exports = {
  entry: './src/index.js', // 随便创建一个 src/index.js 作为入口
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'
  },
  // 注册插件
  plugins: [new MyFirstPlugin()]
};
```

#### 3. 运行测试

```bash
npx webpack
```

你会看到控制台打印出我们的自定义消息，恭喜你写出了第一个 Webpack 插件！

---

### Demo 2：生成自定义文件（操作输出资源）

**功能**：在 `dist` 目录下自动生成一个 `build-info.md` 文件，记录编译时间、输出文件列表等信息。

#### 1. 插件代码：`generate-build-info-plugin.js`

```javascript
class GenerateBuildInfoPlugin {
  apply(compiler) {
    // 注册「emit」钩子：在输出资源到 dist 目录之前触发
    // 这是操作输出文件最常用的钩子！
    compiler.hooks.emit.tapAsync('GenerateBuildInfoPlugin', (compilation, callback) => {
      // 1. 收集编译信息
      const buildInfo = `
# 构建信息
- 构建时间：${new Date().toLocaleString()}
- 输出文件列表：
${Object.keys(compilation.assets).map(file => `  - ${file}`).join('\n')}
      `.trim();

      // 2. 把信息添加到 compilation.assets 中
      // compilation.assets 是一个对象，key 是文件名，value 是文件内容对象
      compilation.assets['build-info.md'] = {
        // 返回文件内容的方法
        source: () => buildInfo,
        // 返回文件大小的方法（必须）
        size: () => buildInfo.length
      };

      // 3. 通知 Webpack 继续
      callback();
    });
  }
}

module.exports = GenerateBuildInfoPlugin;
```

#### 2. 配置使用

在 `webpack.config.js` 中注册：

```javascript
const GenerateBuildInfoPlugin = require('./generate-build-info-plugin');

module.exports = {
  // ... 其他配置不变
  plugins: [new GenerateBuildInfoPlugin()]
};
```

#### 3. 运行效果

执行 `npx webpack` 后，`dist` 目录下会自动生成 `build-info.md`，里面有我们的构建信息！

---

### Demo 3：移除代码中的 console（结合 AST，实用级）

**功能**：在编译阶段自动移除所有 `console.log`，和你之前看的 Babel 插件联动，展示 Webpack 插件如何处理模块内容。

#### 1. 前置依赖

安装处理 AST 的库：

```bash
npm install @babel/parser @babel/traverse @babel/types @babel/generator --save-dev
```

#### 2. 插件代码：`remove-console-plugin.js`

```javascript
const parser = require('@babel/parser');
const traverse = require('@babel/traverse').default;
const t = require('@babel/types');
const generator = require('@babel/generator').default;

class RemoveConsolePlugin {
  apply(compiler) {
    // 注册「compilation」钩子：每次创建新编译实例时触发
    compiler.hooks.compilation.tap('RemoveConsolePlugin', (compilation) => {
      // 注册「优化模块」钩子：在模块构建完成后、优化前触发
      compilation.hooks.optimizeModules.tap('RemoveConsolePlugin', (modules) => {
        // 遍历所有模块
        modules.forEach((module) => {
          // 只处理 .js 文件，跳过 node_modules
          if (module.resource && !module.resource.includes('node_modules') && module.resource.endsWith('.js')) {
            // 获取模块的原始源码
            const originalSource = module._source.source();
            
            // 1. 把源码解析成 AST
            const ast = parser.parse(originalSource, {
              sourceType: 'module'
            });

            // 2. 遍历 AST，移除 console
            traverse(ast, {
              CallExpression(path) {
                if (
                  t.isMemberExpression(path.node.callee) &&
                  t.isIdentifier(path.node.callee.object, { name: 'console' })
                ) {
                  path.remove();
                }
              }
            });

            // 3. 把修改后的 AST 转回代码
            const { code } = generator(ast);

            // 4. 替换模块的源码
            module._source = new compiler.webpack.sources.RawSource(code);
          }
        });
      });
    });
  }
}

module.exports = RemoveConsolePlugin;
```

#### 3. 测试一下

在 `src/index.js` 中写点 console：

```javascript
console.log('这是一条调试日志');
console.warn('这是一条警告');
const a = 1;
console.log(a);
```

配置插件后运行 `npx webpack`，打开 `dist/bundle.js`，你会发现所有 console 都被移除了！

---

## 三、关键补充

### 1. 钩子的三种注册方式

|   |   |   |
|---|---|---|
|方式|适用场景|示例|
|`tap`|同步钩子，逻辑简单|`compiler.hooks.compile.tap('Plugin', () => {})`|
|`tapAsync`|异步钩子，有 callback|`compiler.hooks.done.tapAsync('Plugin', (stats, cb) => { cb(); })`|
|`tapPromise`|异步钩子，返回 Promise|`compiler.hooks.emit.tapPromise('Plugin', async (compilation) => {})`|

### 2. 常用钩子速查

|   |   |   |
|---|---|---|
|钩子|触发时机|典型用途|
|`beforeRun`|编译器启动前|检查环境、修改配置|
|`compile`|开始编译|初始化自定义数据|
|`emit`|输出文件前|修改、添加、删除输出资源（最常用！）|
|`afterEmit`|输出文件后|清理临时文件、上传到 CDN|
|`done`|编译完成|打印统计信息、通知构建结果|

### 3. 调试插件的小技巧

在插件代码中加 `console.log(compiler.hooks)` 可以看到所有可用的钩子； 用 `webpack --debug` 运行可以看到更详细的编译流程。