Webpack Loader 深度解析（原理 + Normal/Pitch 模式 + Webpack5 优化 + 实战示例）

这是全网最透彻、最适合实战的 Webpack Loader 讲解，**从底层原理 → Normal/Pitch 双模式执行流程 → 常用Loader → 手写Loader → Webpack5 最新优化方案**全覆盖，你能彻底搞懂 Loader 到底是什么、怎么工作、怎么用。

---

## 一、先搞懂：Loader 到底是什么？

### 1. 核心定义

Webpack **默认只能理解 JavaScript / JSON 文件**，遇到 CSS、图片、TypeScript、Vue 等文件时**完全不认识**。 **Loader 就是 Webpack 的“翻译官”**： 把**非 JS 资源**转换成 Webpack 能处理的 **JS 模块**。

### 2. 一句话总结

> **Loader = 资源转换器，运行在 Node.js 环境，接收文件源码，输出转换后的代码。**

---

## 二、Loader 底层核心原理（必须掌握）

### 1. 执行机制

1. Webpack 递归解析 `import`/`require` 依赖
    
2. 遇到**非 JS 文件**，根据 `module.rules` 匹配对应 Loader
    
3. **Loader 链式执行**：**从右到左、从下到上**
    
4. 每个 Loader 是一个**纯函数**：`(source) => 转换后的内容`
    
5. 最终输出 JS 代码给 Webpack 打包
    

### 2. 链式执行规则（高频考点）

```JavaScript
use: ['style-loader', 'css-loader', 'less-loader']
```

执行顺序： **less-loader** → **css-loader** → **style-loader**

> 口诀：**右边/下边先执行，左边/上边后执行**

---

## 三、核心进阶：Loader 的 Normal 与 Pitch 双模式（深度理解）

这是 Loader 最核心、最容易被忽略的高级特性，也是面试高频考点。

### 1. 双模式定义

每个 Loader 都有两个执行阶段：

- **Normal 阶段**：我们平时写的 Loader 主函数，负责实际转换文件内容
    
- **Pitch 阶段**：Loader 上的 `pitch` 方法，在 Normal 阶段**之前**执行，是一个**拦截器/预处理器**
    

### 2. 完整执行流程（图解）

对于配置：

```JavaScript
use: ['loader-a', 'loader-b', 'loader-c']
```

执行顺序如下：

```Plaintext
┌─────────────────────────────────────────────────────────┐
│                    Pitch 阶段（从左到右）                  │
│  loader-a.pitch() → loader-b.pitch() → loader-c.pitch() │
└─────────────────────────────────────────────────────────┘
                            ↓
                    读取原始文件内容
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   Normal 阶段（从右到左）                  │
│  loader-c() ← loader-b() ← loader-a()                    │
└─────────────────────────────────────────────────────────┘
```

### 3. Pitch 阶段的核心作用：拦截与短路

**如果某个 Loader 的** **`pitch`** **方法返回了非** **`undefined`** **的值，会直接跳过后续所有 Loader，进入 Normal 阶段的逆向执行。**

这是 Pitch 最强大的特性，常用于：

- 条件性跳过某些 Loader
    
- 注入自定义代码
    
- 收集元数据
    
- 性能优化（提前返回，避免不必要的处理）
    

### 4. 实战示例：Pitch 模式拦截

我们创建三个简单的 Loader 来演示 Pitch 模式的拦截效果：

**loader-a.js**

```JavaScript
module.exports = function (source) {
  console.log('loader-a normal 执行');
  return source + '\n// loader-a 处理过了';
};

module.exports.pitch = function (remainingRequest, previousRequest, data) {
  console.log('loader-a pitch 执行');
  // 如果这里返回内容，会直接跳过 loader-b 和 loader-c
  // return '// 被 loader-a pitch 拦截了';
};
```

**loader-b.js**

```JavaScript
module.exports = function (source) {
  console.log('loader-b normal 执行');
  return source + '\n// loader-b 处理过了';
};

module.exports.pitch = function () {
  console.log('loader-b pitch 执行');
};
```

**loader-c.js**

```JavaScript
module.exports = function (source) {
  console.log('loader-c normal 执行');
  return source + '\n// loader-c 处理过了';
};

module.exports.pitch = function () {
  console.log('loader-c pitch 执行');
};
```

**正常执行输出（无拦截）：**

```Plaintext
loader-a pitch 执行
loader-b pitch 执行
loader-c pitch 执行
loader-c normal 执行
loader-b normal 执行
loader-a normal 执行
```

**如果在 loader-a.pitch 中返回内容（拦截）：**

```JavaScript
module.exports.pitch = function () {
  console.log('loader-a pitch 执行，拦截！');
  return '// 被 loader-a pitch 拦截了';
};
```

**输出变为：**

```Plaintext
loader-a pitch 执行，拦截！
loader-a normal 执行
```

**loader-b 和 loader-c 被完全跳过了！**

### 5. Pitch 方法的参数

```JavaScript
module.exports.pitch = function (remainingRequest, previousRequest, data) {
  // remainingRequest: 剩余未执行的 Loader 链（字符串）
  // previousRequest: 已执行的 Loader 链（字符串）
  // data: 一个对象，可在 pitch 和 normal 之间传递数据
  data.customValue = '我是从 pitch 传过来的';
};

module.exports = function (source) {
  // 这里可以访问到 pitch 中设置的 data
  console.log(this.data.customValue); // 输出：我是从 pitch 传过来的
  return source;
};
```

---

## 四、Loader 分类（4 种，面试必问）

|   |   |   |
|---|---|---|
|类型|作用|示例|
|**前置 Loader**|优先执行|enforce: 'pre'|
|**普通 Loader**|默认执行|无 enforce|
|**后置 Loader**|最后执行|enforce: 'post'|
|**行内 Loader**|代码内直接指定|import 'loader1!loader2!./file.js'|

执行总顺序： **pre → 普通 → post**

---

## 五、最常用 Loader 功能解析（带示例）

### 1. css-loader + style-loader

- `css-loader`：解析 CSS 中的 `@import`/`url()`
    
- `style-loader`：把 CSS 插入到 DOM 的 `<style>` 标签
    

配置：

```JavaScript
module: {
  rules: [
    {
      test: /\.css$/,
      use: ['style-loader', 'css-loader']
    }
  ]
}
```

### 2. less-loader / sass-loader

把 Less/Sass 编译成 CSS

```JavaScript
use: ['style-loader', 'css-loader', 'less-loader']
```

### 3. babel-loader

把 ES6+ → ES5，兼容低版本浏览器

```JavaScript
{
  test: /\.js$/,
  exclude: /node_modules/,
  use: 'babel-loader'
}
```

### 4. file-loader / url-loader（Webpack5 已内置）

处理图片、字体等静态资源

```JavaScript
{
  test: /\.(png|jpg|svg)$/,
  type: 'asset' // webpack5 已内置，替代 file-loader/url-loader
}
```

---

## 六、手写一个 Loader（深度理解原理）

我们手写一个**去除代码中 console.log 的 Loader**，彻底吃透 Loader 本质。

### 1. 创建 loaders/remove-console-loader.js

```JavaScript
/**
 * 自定义 Loader：移除代码中的 console.log
 * @param {string} source 源文件内容（字符串）
 * @return 转换后的内容
 */
module.exports = function (source) {
  // 正则替换：删除所有 console.log(...)
  const result = source.replace(/console\.log\(.*?\);?/g, '');
  
  // 返回转换后的代码
  return result;
};
```

### 2. webpack.config.js 配置使用

```JavaScript
const path = require('path');

module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        use: [
          // 使用自定义 Loader
          path.resolve(__dirname, 'loaders/remove-console-loader.js')
        ]
      }
    ]
  }
};
```

### 3. 测试效果

源文件：

```JavaScript
console.log('hello');
const a = 1;
```

打包后：

```JavaScript
const a = 1;
```

✅ **console.log 被成功移除**

---

## 七、进阶：Loader 高级 API（实战必备）

### 1. 获取 Loader 配置项

```JavaScript
module.exports = function (source) {
  // 获取配置 { remove: true }
  const options = this.getOptions();
  
  if (options.remove) {
    return source.replace(/console\.log/g, '');
  }
  return source;
};
```

使用：

```JavaScript
use: {
  loader: 'xxx-loader',
  options: { remove: true }
}
```

### 2. 异步 Loader

```JavaScript
module.exports = function (source) {
  const callback = this.async(); // 开启异步
  
  setTimeout(() => {
    const result = source.replace(/log/g, '');
    callback(null, result); // 错误优先回调
  }, 100);
};
```

### 3. 缓存控制（性能优化）

```JavaScript
module.exports = function (source) {
  this.cacheable(false); // 关闭缓存（默认开启）
  return source;
};
```

---

## 八、Webpack5 最新 Loader 优化方案（2026 年实战必备）

Webpack5 带来了革命性的性能提升，以下是最新、最有效的 Loader 优化方案。

### 1. 使用更快的 Loader 替代方案（速度提升 10-100 倍）

传统的 `babel-loader`、`css-loader` 等虽然功能强大，但速度较慢。Webpack5 时代推荐使用以下高性能替代方案：

#### （1）JS/TS 编译：esbuild-loader 或 swc-loader

**esbuild-loader**（Go 语言编写，速度极快）：

```Bash
npm install esbuild-loader --save-dev
```

配置：

```JavaScript
module: {
  rules: [
    {
      test: /\.(js|jsx|ts|tsx)$/,
      exclude: /node_modules/,
      use: [
        {
          loader: 'esbuild-loader',
          options: {
            loader: 'jsx', // 支持 React
            target: 'es2020',
          },
        },
      ],
    },
  ],
}
```

**swc-loader**（Rust 语言编写，速度同样极快，生态更完善）：

```Bash
npm install @swc/core @swc/loader --save-dev
```

配置：

```JavaScript
module: {
  rules: [
    {
      test: /\.(js|jsx|ts|tsx)$/,
      exclude: /node_modules/,
      use: [
        {
          loader: '@swc/loader',
          options: {
            jsc: {
              parser: {
                syntax: 'typescript',
                jsx: true,
              },
              transform: {
                react: {
                  runtime: 'automatic',
                },
              },
              target: 'es2020',
            },
          },
        },
      ],
    },
  ],
}
```

**效果**：构建速度提升 **10-100 倍**，大型项目效果尤为明显。

#### （2）CSS 处理：lightningcss-loader

**lightningcss**（Rust 语言编写）替代 `css-loader` + `postcss-loader`：

```Bash
npm install lightningcss-loader --save-dev
```

配置：

```JavaScript
module: {
  rules: [
    {
      test: /\.css$/,
      use: [
        'style-loader',
        {
          loader: 'lightningcss-loader',
          options: {
            minify: true, // 同时压缩 CSS
          },
        },
      ],
    },
  ],
}
```

### 2. 精准控制 Loader 作用范围（减少不必要处理）

#### （1）使用 `include` 和 `exclude`

```JavaScript
{
  test: /\.js$/,
  include: path.resolve(__dirname, 'src'), // 只处理 src 目录
  exclude: /node_modules/, // 排除 node_modules
  use: 'babel-loader'
}
```

#### （2）使用 `noParse` 跳过无需解析的库

对于一些没有依赖的、预编译好的库（如 jQuery、Lodash），可以告诉 Webpack 不用去解析：

```JavaScript
module: {
  noParse: /jquery|lodash/,
}
```

### 3. 开启 Loader 缓存（大幅提升二次构建速度）

#### （1）babel-loader 缓存

```JavaScript
{
  test: /\.js$/,
  use: [
    {
      loader: 'babel-loader',
      options: {
        cacheDirectory: true, // 开启缓存
        cacheCompression: false, // 大项目关闭压缩，提升速度
      },
    },
  ],
}
```

#### （2）Webpack5 内置持久化缓存

这是 Webpack5 最强大的优化特性，开箱即用：

```JavaScript
module.exports = {
  cache: {
    type: 'filesystem', // 持久化到文件系统
    buildDependencies: {
      config: [__filename], // 配置文件变更时失效缓存
    },
  },
};
```

**效果**：二次构建速度提升 **80%+**。

### 4. 使用 thread-loader 多线程处理（大型项目必备）

对于耗时的 Loader（如 babel-loader），可以使用 `thread-loader` 开启多线程处理：

```Bash
npm install thread-loader --save-dev
```

配置：

```JavaScript
{
  test: /\.js$/,
  use: [
    'thread-loader', // 放在最前面，开启多线程
    {
      loader: 'babel-loader',
      options: {
        cacheDirectory: true,
      },
    },
  ],
}
```

**注意**：`thread-loader` 本身有启动开销，只适合大型项目，小型项目反而会变慢。

### 5. Webpack5 内置 Asset Modules（替代 file-loader/url-loader）

Webpack5 内置了资源处理能力，无需再安装额外的 Loader：

```JavaScript
module: {
  rules: [
    {
      test: /\.(png|jpg|gif)$/,
      type: 'asset',
      parser: {
        dataUrlCondition: {
          maxSize: 8 * 1024, // 8k 以下转 base64，超过 8k 输出文件
        },
      },
    },
  ],
}
```

---

## 九、完整实战配置（可直接复制使用）

```JavaScript
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  // Webpack5 持久化缓存
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },
  module: {
    rules: [
      // 处理 JS/TS：使用 esbuild-loader 替代 babel-loader
      {
        test: /\.(js|jsx|ts|tsx)$/,
        include: path.resolve(__dirname, 'src'),
        exclude: /node_modules/,
        use: [
          // 大型项目可开启 thread-loader
          // 'thread-loader',
          {
            loader: 'esbuild-loader',
            options: {
              loader: 'jsx',
              target: 'es2020',
            },
          },
        ],
      },
      // 处理 CSS/Less
      {
        test: /\.less$/,
        use: ['style-loader', 'css-loader', 'less-loader']
      },
      // 处理图片：Webpack5 内置 Asset Modules
      {
        test: /\.(png|jpg|gif)$/,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024 // 8k 以下转 base64
          }
        }
      },
      // 使用自定义 Loader
      {
        test: /\.js$/,
        use: path.resolve(__dirname, 'loaders/remove-console-loader.js')
      }
    ]
  }
};
```

---

## 十、Loader 核心总结（面试/工作必背）

1. **Loader 是资源转换器**，让 Webpack 能处理非 JS 文件
    
2. **双模式执行**：Pitch 阶段（从左到右，可拦截）→ Normal 阶段（从右到左，实际转换）
    
3. **执行顺序**：从右到左、从下到上
    
4. **结构**：一个接收 `source`、返回字符串的 Node 函数
    
5. **分类**：pre、普通、post、行内 Loader
    
6. **Webpack5 优化**：使用 esbuild/swc/lightningcss 替代传统 Loader，开启持久化缓存，使用内置 Asset Modules
    
7. **自定义 Loader**：简单、强大、可做代码转换、校验、压缩等
    

---

### 最终一句话记住 Loader

**Loader = 翻译官，把非 JS 变成 JS，Pitch 可拦截，Normal 来转换，Webpack5 用 esbuild 更快。**

如果你需要，我还可以给你：

- Loader + Plugin 区别对比
    
- 手写更多实战 Loader（校验、压缩、国际化替换）
    
- Webpack5 完整性能优化方案