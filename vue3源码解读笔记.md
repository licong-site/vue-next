# vue3 源码解读笔记







## 不同的构建版本

#### vue(.runtime).global(.prod).js

通过 `<script src="">` 直接使用，暴露 Vue到全局变量。内联所有 Vue 核心内部包。全局打包时被打包成 IIFEs 立即执行函数，使用 `<script>` 标签引入之后就可以直接使用。

- vue.global.js 包含编译器和运行时的完整构建，支持动态编译模板
- vue.runtime.global.js 只包含运行时，并且需要在构建步骤期间预编译模板



#### vue(.runtime).esm-browser(.prod).js

用于通过原生 ES Module导入使用，在浏览器中通过 `<script type="module">` 来使用。

与全局构建共享相同的运行时编译、依赖内联和硬编码的 prod/dev 行为。



#### vue(.runtime).esm-bundler.js

- 用于 webpack、rollup、parcel 等构建工具

- 不提供压缩版本，打包后和其余代码一起压缩

- import 依赖也是 esm bundler 构建，并将依次导入其依赖项。如 @vue/runtime-core 会imports @vue/reactivity。

- 浏览器内模板编译

  - vue.runtime.esm-bundler.js （默认）只有运行时，要求所有模板必须要预先编译。这是构建工具的默认入口，因为在使用构建工具时，模板通常是预先编译的。
  - vue.esm-bundler.js 包含运行时编译器，如果你使用了一个构建工具，但仍然想要运行时的模板编译 (例如，DOM 内 模板或通过内联 JavaScript 字符串的模板)，请使用这个文件。你需要配置你的构建工具，将 vue 设置为这个文件。

  

#### vue.cjs(.prod).js 服务端渲染

通过 require() 在 node.js 服务器端使用。如果你将应用程序与带 `target: 'node'` 的 webpack 打包在一起，并正确的将 vue 外部化，则加载此文件。



```
vue-next/packages
├─ compiler-core
├─ compiler-dom
├─ compiler-sfc
├─ compiler-ssr
├─ global.d.ts
├─ reactivity
├─ runtime-core
├─ runtime-dom
├─ runtime-test
├─ server-renderer
├─ sfc-playground
├─ shared  提供全局用到的工具函数
├─ size-check
├─ template-explorer
├─ vue
└─ vue-compat
```

## @vue/compiler-core

模板编译的核心代码，构建 node 到 effect 的连接。基本流程

1. **parser**: 将 template 模板解析生成 node 对象， 如 **TextNode**，**InterpolationNode**
2. **transform**: 优化，(节点标记，静态节点提升 ???)
3. **generate**: 将 ast 生成代码，（render 渲染函数 ？？）

依赖库：

- @vue/shared
- @babel/parser  解析模板中的复杂表达式
- @babel/types
- estree-walker  用于遍历符合ESTree规范的AST语法树的工具库，类似 acron
- source-map

#### ast.ts

定义虚拟DOM对象，常用的有 **ElementNode**、**InterpolationNode**、**IfNode**、**ForNode** 

#### parse.ts   将模板解析代码生成 虚拟节点

将 template 字符串通过正则匹配转换成 ast.ts 文件中定义的各种类型的 Node 对象

```typescript
test('it can have tag-like notation', () => {
      const ast = baseParse('{{ a<b }}')
      const interpolation = ast.children[0] as InterpolationNode

      expect(interpolation).toStrictEqual({
        type: NodeTypes.INTERPOLATION,
        content: {
          type: NodeTypes.SIMPLE_EXPRESSION,
          content: `a<b`,
          isStatic: false,
          constType: ConstantTypes.NOT_CONSTANT,
          loc: {
            start: { offset: 3, line: 1, column: 4 },
            end: { offset: 6, line: 1, column: 7 },
            source: 'a<b'
          }
        },
        loc: {
          start: { offset: 0, line: 1, column: 1 },
          end: { offset: 9, line: 1, column: 10 },
          source: '{{ a<b }}'
        }
      })
    })

```

#### tarnsform.ts  对虚拟节点进行优化

处理节点里面的指令，表达式，绑定的变量 ？？，并做一些优化

##### 1. **`traverseNode`** 

 遍历每一个 AST 节点，调用 transform 插件。

1. 节点转换 nodeTransform:

- `transformOnce`
- `transformIf`
- `transformFor`
- `transformExpression` 
- `transformFilter`
- `transformVForSlotScopes` 
- `transformElement`
- `transformText` 
- ....

2. 指令转换 

- `transformOn` 
- `transformBind` 
- `transformModel` 

##### 2. **`processExpression`** 

当解析节点指令或变量绑定时，如果有表达式，就会调用 `processExpression()` 方法

会调用 @babel/parser 的 `parse` 方法解析表达式。



##### 3. **`hoistStatic`** 静态节点提升

使用 @babel/parser 解节点中绑定和插入的表达式



##### 4.  `generate`  生成模板对应的渲染函数

`function render(...){}`

生成代码的时候会把前面标志的静态节点，从 render 函数中提取出来。



## @vue/compiler-dom 浏览器端编译

## @vue/compiler-sfc 将 .vue 文件编译成JavaScript

如果想要自己开发打包工具的plugin或loader的时候会需要用到。现在有在 vue-loader、rollup-plugin-vue 和 vite 中使用。

基本流程：

#### 1. parse 将单文件解析成  script\template\style

解析 .vue 文件的内容，返回 **`SFCDescriptor`** 和解析错误或语法错误。

```typescript
  const descriptor: SFCDescriptor = {
    filename,
    source,
    template: null, // <template>
    script: null, // <script>
    scriptSetup: null, // <script setup>
    styles: [], // <style>
    customBlocks: [], // <i18n>{"greeting": "hello"}</i18n>
    cssVars: [], // 样式中绑定的变量，<script> div{ color: v-bind(color); } </div>
    slotted: false // SFC 中有没有使用 ::v-slotted 或 :slotted
  }
```

解析过的文件会用 map 缓存，一个文件只解析一次。解析时使用 @vue/compiler-dom 的  `parse` 解析出 templateBlock、scriptBlock、styleBlock。

#### 2. compileScript

#### 3. compileTemplate 

#### 4. compileStyle





## @vue/compiler-ssr 服务端渲染时的编译



>  @vue/compiler-ssr、 @vue/compiler-dom、@vue/compiler-sfc 实质上都是使用 @vue/compiler-core完成模板（代码）解析的, 只是 **CompilerOptions** 配置不同



## @vue/reactivity 包装响应式对象，收集依赖、触发事件

一般在 @vue/runtime-dom 包中内置，也可以当成一个单独的标准库来使用。



```
📦src
 ┣ 📜baseHandlers.ts  数组、对象代理时的拦截方法
 ┣ 📜collectionHandlers.ts  map\set\weakmap\weakset 代理时的拦截方法
 ┣ 📜computed.ts
 ┣ 📜effect.ts   targetMap、effectStack、activeEffect、trigger方法、track方法
 ┣ 📜index.ts
 ┣ 📜operations.ts  枚举TrackOpTypes、TriggerOpTypes
 ┣ 📜reactive.ts   创建响应式对象
 ┗ 📜ref.ts  ref相关接口
```

### `effect.ts` 

#### track

#### trigger

### reactive.ts



## @vue/runtime-core 运行时

自定义渲染需要继续这个库进行开发。

### scheduler.ts    事件队列





### 生命周期



































