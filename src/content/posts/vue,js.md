---
title: vue.js的设计与实现阅读笔记1
published: 2025-09-07
description: 'block diff 与双端 diff'
image: ''
tags: [Vue]
category: 'Vue'
draft: false 
---

### 一、什么是 block diff？
- 编译阶段：编译器将模板按动态内容分块（block），静态节点被提升（hoist）出来。
- 渲染/更新阶段：首次渲染创建 DOM；更新时只对含动态节点的 block 进行 diff，静态节点跳过比较。  
优点：减少不必要的比较与 DOM 操作，尤其对大量静态内容的页面效果明显。

### 二、block diff 的简要流程
- 编译器生成 openBlock() / createBlock() 把动态节点收进 block；运行时维护 dynamicChildren 列表；
- patch 时若 vnode 有 dynamicChildren，就只对这些动态子节点做更细粒度的更新（跳过静态子树）。

示例（编译输出示意）：
```js
openBlock();
createBlock(Fragment, null, [
  createVNode("div", null, "static"), // 静态提升
  createVNode("span", null, dynamicValue) // 动态节点，属于 block
]);
```

### 三、双端 diff（双端指针算法） — 举例与流程
双端 diff 用 4 个指针从两端同时比对，快速处理头尾相同、少量移动的场景；只在无法匹配时才建立 key->index 映射并使用 LIS 最小化移动。

示例（具体步骤）：
- 旧序列（old）：[A, B, C, D, E]（索引 0..4）
- 新序列（new）：[B, D, A, E, C]

1. 指针初始化：
   - oldStart=0(A), oldEnd=4(E)
   - newStart=0(B), newEnd=4(C)

2. 逐步匹配（伪操作）：
   - 比较 oldStart(A) vs newStart(B)：不同 → 继续其他匹配
   - 比较 oldEnd(E) vs newEnd(C)：不同
   - 比较 oldStart(A) vs newEnd(C)：不同
   - 比较 oldEnd(E) vs newStart(B)：不同
   - 这时快速双端匹配失败 → 进入映射阶段

3. 建立 oldKey->index:
   - { A:0, B:1, C:2, D:3, E:4 }

4. 把 new 映射为旧索引序列（存在则取旧索引，否则挂载新节点）：
   - newIndexes = [1, 3, 0, 4, 2]

5. 求 newIndexes 的 LIS（最长递增子序列），假设 LIS = [1,3,4]（对应 old 的 B,D,E），表示这些节点按相对顺序可保持不动；
   - 其余索引（0 -> A，2 -> C）需要移动到新位置。

6. 最终操作：
   - 根据从右向左遍历并参考 LIS，最少次数移动/插入/删除 DOM 节点，完成 reorder。

这个过程把不必要的 DOM move 降到最少。

### 四、Vue 3 中与 diff 相关的所有主要点
- vnode 类型判定（type + shapeFlag）：快速分支决定如何处理（component、element、text、fragment、portal/teleport 等）。
- block / dynamicChildren：编译时标记动态节点，运行时只对 dynamicChildren 做细粒度 patch（减少遍历）。
- patch 的 fast path：当 vnode type 相同且 patchFlag 支持快速更新时，直接只更新必要内容（props、文本、部分子节点）。
- keyed vs unkeyed children：
  - keyed：使用双端指针 + key 映射 + LIS（最少移动）。
  - unkeyed（或没有 key）：采用索引对齐，依次 patch，多退少补，适用于简单场景但对重排支持差。
- patchKeyedChildren：实现双端 diff 的具体函数（包含快速头尾匹配、映射构造、LIS 优化）。
- Fragment：允许多个子节点作为一体处理，diff 时对 Fragment 子列表做同样的 children diff。
- Component 更新：区分组件更新（props、slots）与卸载/挂载，尽量复用实例。
- 其他优化：文本合并、小写次数控制、避免不必要的 DOM api 调用、懒更新（scheduleFlush）等。

### 五、Vue3 相比 Vue2 的主要 diff 改进（模拟人的说明）
- 更细粒度的静态/动态区分（block diff）：Vue3 的最大进步之一，编译器把静态提升、动态收集到 block，运行时能跳过大量静态节点比较；Vue2 没有这层显式的 block 跳过机制。
- 更丰富的 patchFlag（编译器标记）：Vue3 在编译阶段输出 patchFlag，让运行时知道“只更新文本/props/事件/子节点”等，避免全量比对；Vue2 更多依赖运行时的常规 diff。
- 更高效的 children diff：虽然 Vue2 使用的是类似双端算法，但 Vue3 在实现上更模块化、配合 block 与 patchFlag 使用时能减少工作量，并且对 keyed 列表使用 LIS 做最少移动优化更规范、更高效。
- 更小的 vnode/更快的类型判断：Vue3 的 shapeFlag、块优化与更紧凑的 vnode 表示减少了判断成本。
- Fragment/Teleport/Suspense 支持：这些新 vnode 类型在 diff 时被更合理地处理，避免了额外的占位和不必要的 DOM 操作。
- 性能与内存：整体实现更注重零分配（或少分配）、减少垃圾回收、以及在常见路径上优化，带来更稳定的性能提升。

总结一句话：Vue3 把编译器的静态分析与运行时的 diff 逻辑结合得更紧密（block + patchFlag + 双端 diff + LIS），从“统一运行时比较”变为“只在必要时比较并尽量少修改 DOM”。
