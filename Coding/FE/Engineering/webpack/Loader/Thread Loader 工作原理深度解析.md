

# thread-loader 工作原理（极简 + 深度）

我用**最通俗、最核心、最容易记住**的方式给你讲清楚，**看完你就彻底懂它为什么必须放前面、为什么能加速、原理是什么**。

---

# 一、一句话核心原理

**thread-loader 是一个 “任务搬运工”**： 它在 **Pitch 阶段** 把**后面所有 Loader 的执行任务**，从 **Webpack 主线程** 搬运到 **独立的 worker 子线程** 里**并行执行**，利用 CPU 多核加速构建。

---

# 二、前置关键知识（必须懂）

1. **Webpack 是单线程的** 默认所有 Loader、编译、解析都跑在**一个主线程**里，多核 CPU 根本用不上。
    
2. **Loader 执行顺序**
    
    1. **Pitch 阶段：从左到右**
        
    2. **Normal 阶段：从右到左**
        
3. **thread-loader 自己不编译代码** 它只做一件事：**接管后面 Loader 的执行权**。
    

---

# 三、thread-loader 完整工作流程（深度）

## 流程 1：Pitch 阶段（从左到右）

thread-loader 排在最左 → **最先执行 Pitch**。

它在 Pitch 阶段干 3 件事：

1. **拦截**：告诉 Webpack：“后面的 Loader 别在主线程执行了，交给我”
    
2. **创建 worker 池**：启动几个子线程（默认 CPU 核数 -1）
    
3. **保存任务队列**：把**后面所有 Loader** 记录下来，准备丢给子线程
    

## 流程 2：Normal 阶段被劫持

正常情况下 Normal 是**从右往左**执行。

但因为 thread-loader 在 Pitch 阶段**拦截了**，所以：

- 后面所有 Loader **不会在主线程执行**
    
- 全部被序列化 → 发送给 **worker 子线程**
    

## 流程 3：worker 子线程执行真正的 Loader 编译

子线程中：

1. 反序列化 Loader
    
2. **按正常顺序从右往左执行 Loader**（babel-loader、ts-loader 等）
    
3. 编译完成后把结果返回主线程
    

## 流程 4：主线程接收结果，继续后续流程

编译结果回到主线程，Webpack 继续打包。

---

# 四、原理图（一眼看懂）

```Plain
主线程：
thread-loader Pitch（拦截）
    ↓
创建 worker 线程
    ↓
将【babel-loader、ts-loader、vue-loader】发给 worker
    ↓
====================================
worker 线程（多核并行）：
执行 babel-loader → 编译 JS
执行 ts-loader    → 编译 TS
执行 vue-loader  → 编译 Vue
返回编译结果
====================================
    ↓
主线程接收结果 → 继续打包
```

---

# 五、为什么 thread-loader 必须放在最前面？

因为： **它只在 Pitch 阶段工作，Pitch 是从左到右执行的。**

如果不放最前面：

- 其他 Loader 已经在主线程执行完了
    
- thread-loader 来不及拦截
    
- **多线程完全失效**
    

---

# 六、thread-loader 加速的真正原因

**主线程不再阻塞，Loader 编译任务多核并行执行。**

大型项目中 babel / TypeScript / Vue 编译非常耗时，单线程跑很慢。 thread-loader 让这些任务**并行跑**，所以速度提升明显。

---

# 七、最精简总结（背会就能面试）

1. **thread-loader 依靠 Pitch 阶段拦截后续 Loader**
    
2. **把后续 Loader 丢进 worker 子线程并行执行**
    
3. **利用 CPU 多核加速构建**
    
4. **必须放最前面，否则无法拦截**
    
5. **自己不编译，只做任务搬运和线程调度**
    

---

如果你愿意，我还能给你： ✅ **thread-loader 源码核心流程** ✅ **worker 池如何工作** ✅ **与 eslint-webpack-plugin、babel-loader 配合最佳实践** ✅ **thread-loader 为什么不适合小型项目（原理级解释）**

你要继续深入吗？