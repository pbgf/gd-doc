# @sailorgundam/gundam-js-to-groovy

> JavaScript to Groovy 代码转换器 — 基于 Babel AST 的 DSL 转译引擎

**版本**: 0.67.0  
**定位**: 将 JavaScript DSL 配置代码转换为 Groovy 脚本，用于 Gundam 活动页的服务端接口聚合  
**核心引擎**: Babel (@babel/parser + @babel/traverse + @babel/types)  
**CLI**: `js2groovy <input> [-o output] [-v]`

---

## 目录

- [架构概览](#架构概览)
- [两阶段转换流程](#两阶段转换流程)
- [核心模块](#核心模块)
  - [JSToGroovyTranspiler](#jstogroovytranspiler)
  - [GroovyPostProcessor](#groovypostprocessor)
  - [DSL 模板](#dsl-模板)
  - [CLI 工具](#cli-工具)
- [语法转换映射](#语法转换映射)
- [支持的 AST 节点](#支持的-ast-节点)
- [使用示例](#使用示例)
- [依赖关系](#依赖关系)

---

## 架构概览

```
JavaScript DSL 代码
        │
        ▼
┌──────────────────────┐
│  JSToGroovyTranspiler │  ← 第一阶段：JS AST → Groovy 代码
│  (Babel Parser)       │
│  - parse() → AST      │
│  - validateAST()      │
│  - convertProgram()   │
│  - convert*() 系列方法 │
└──────────┬───────────┘
           │
           ▼
   原始 Groovy 代码
           │
           ▼
┌──────────────────────┐
│  GroovyPostProcessor  │  ← 第二阶段：Groovy 代码后处理
│  - extends Script     │
│  - run() 参数绑定     │
│  - appkey 替换        │
└──────────┬───────────┘
           │
           ▼
   最终 Groovy 脚本
```

---

## 两阶段转换流程

### 第一阶段：JSToGroovyTranspiler

1. **解析**：使用 `@babel/parser` 将 JavaScript 代码解析为 AST
2. **验证**：遍历 AST 检查是否存在不支持的语法节点类型
3. **转换**：递归遍历 AST，将每个节点转换为对应的 Groovy 语法

### 第二阶段：GroovyPostProcessor

1. **添加继承**：为类添加 `extends Script`
2. **run 方法处理**：添加 `@Override` 注解，修改返回类型为 `Object`，移除参数声明
3. **参数绑定**：将 `run` 方法的参数转换为 `binding.getVariable("paramName")` 调用
4. **appkey 替换**：EU 机房环境下将 `com.sankuai.` 替换为 `com.keetapp.`

---

## 核心模块

### JSToGroovyTranspiler

**文件**: `src/index.js`  
**导出**: `JSToGroovyTranspiler` 类 + `getDSLTemplate` + `GroovyPostProcessor`

#### 构造函数

初始化支持的 AST 节点类型集合和输出缓冲区：

```javascript
constructor() {
    this.supportedNodeTypes = new Set([
      'Program', 'ClassDeclaration', 'ClassBody', 'MethodDefinition',
      'ClassMethod', 'FunctionDeclaration', 'ReturnStatement',
      'ObjectExpression', 'Property', 'ObjectProperty', 'ArrayExpression',
      'ConditionalExpression', 'LogicalExpression', 'BinaryExpression',
      'MemberExpression', 'Identifier', 'Literal', 'StringLiteral',
      'NumericLiteral', 'BooleanLiteral', 'NullLiteral', 'BlockStatement',
      'ExpressionStatement', 'VariableDeclaration', 'VariableDeclarator',
      'AssignmentExpression', 'CallExpression', 'OptionalMemberExpression',
      'OptionalCallExpression', 'NewExpression', 'ExportNamedDeclaration',
      'ExportDefaultDeclaration', 'TemplateLiteral', 'TemplateElement',
      'TryStatement', 'CatchClause', 'IfStatement', 'ObjectPattern',
      'ArrayPattern', 'ArrowFunctionExpression', 'UnaryExpression',
    ]);
    this.output = '';
    this.indentLevel = 0;
}
```

#### 核心方法

| 方法 | 说明 |
|------|------|
| `transpile(jsCode)` | **主入口**。解析 JS → 验证 AST → 转换为 Groovy 代码字符串 |
| `validateAST(ast)` | 遍历 AST，发现不支持的节点类型时抛出异常 |
| `convertProgram(ast)` | 转换 Program 节点，遍历所有顶层语句 |
| `convertStatement(node)` | 语句分发器，根据 `node.type` 调用对应的 `convert*` 方法 |

#### 语句转换方法

| 方法 | JS 输入 | Groovy 输出 |
|------|---------|-------------|
| `convertClass(node)` | `class Foo { ... }` | `class Foo { ... }` |
| `convertMethod(node)` | `run(props) { ... }` | `Map run(Map props) { ... }` |
| `convertFunction(node)` | `function foo(a, b) { ... }` | `def foo(a, b) { ... }` |
| `convertVariableDeclaration(node)` | `const x = 1` | `def x = 1` |
| `convertTryStatement(node)` | `try { } catch(e) { } finally { }` | `try { } catch(e) { } finally { }` |
| `convertIfStatement(node)` | `if (x) { } else if (y) { } else { }` | `if (x) { } else if (y) { } else { }` |

**变量声明解构支持**：

| JS 解构 | Groovy 输出 |
|---------|------------|
| `const { a, b } = obj` | `def a = obj.a` + `def b = obj.b` |
| `const [x, y] = arr` | `def x = arr[0]` + `def y = arr[1]` |

#### 表达式转换方法

| 方法 | JS 输入 | Groovy 输出 |
|------|---------|-------------|
| `convertObjectExpression(node)` | `{ key: value }` | `[key: value]` |
| `convertArrayExpression(node)` | `[1, 2, 3]` | `[1, 2, 3]` |
| `convertConditionalExpression(node)` | `a ? b : c` | `a ? b : c` |
| `convertLogicalExpression(node)` | `a \|\| b` | `a ?: b` (Elvis) |
| `convertLogicalExpression(node)` | `a && b` | `a && b` |
| `convertBinaryExpression(node)` | `a === b` | `a == b` |
| `convertBinaryExpression(node)` | `a !== b` | `a != b` |
| `convertMemberExpression(node)` | `obj.prop` | `obj.prop` |
| `convertOptionalMemberExpression(node)` | `obj?.prop` | `obj?.prop` |
| `convertArrowFunctionExpression(node)` | `(x) => x + 1` | `{ x -> x + 1 }` |
| `convertTemplateLiteral(node)` | `` `${name}` `` | `"${name}".toString()` |
| `convertUnaryExpression(node)` | `!x`, `-x` | `!x`, `-x` |
| `convertAssignmentExpression(node)` | `x = y`, `x += y` | `x = y`, `x += y` |

#### 特殊函数转换

`convertGlobalFunctionCall(callee, args)` 处理 JavaScript 全局函数到 Groovy 的映射：

| JS 调用 | Groovy 输出 |
|---------|------------|
| `Date.now()` | `System.currentTimeMillis()` |
| `JSON.parse(str)` | `new groovy.json.JsonSlurper().parseText(str)` |
| `decodeURIComponent(str)` | `java.net.URLDecoder.decode(str, 'UTF-8')` |
| `Number(x)` | `(x ?: 0) as BigDecimal` |

`convertMethodCall(node, args, chainOperator, isOptionalCallExpression)` 处理数组方法映射：

| JS 方法 | Groovy 输出 |
|---------|------------|
| `arr.includes(x)` | `arr.contains(x)` |
| `arr?.includes?.(x)` | `arr?.contains?.call(x)` |
| `arr.map(fn)` | `arr.collect(fn)` |
| `arr?.map?.(fn)` | `arr?.collect?.call(fn)` |

#### 可选调用表达式

`convertOptionalCallExpression(node)` 处理 JavaScript 可选调用语法：

| JS 语法 | Groovy 输出 |
|---------|------------|
| `foo?.()` | `foo?.call()` |
| `foo?.bar?.()` | `foo?.bar?.call()` |

---

### GroovyPostProcessor

**文件**: `src/groovy-post-processor.js`  
**导出**: `GroovyPostProcessor` 类

#### 核心方法

| 方法 | 说明 |
|------|------|
| `process(groovyCode, serverLoc)` | 处理 Groovy 代码，`serverLoc` 支持 `'HK'`（默认）或 `'EU'` |
| `parseMethodParams(paramString)` | 解析方法参数字符串，提取参数名（去掉 `Map` 类型声明） |

#### 转换规则

**1. 添加 extends Script**

```groovy
// 输入
class CustomApiConfigBuilder {
// 输出
class CustomApiConfigBuilder extends Script {
```

**2. run 方法参数绑定**

```groovy
// 输入
Map run(Map props, Map requestContext) {
// 输出
@Override
Object run() {
    def props = binding.getVariable("props")
    def requestContext = binding.getVariable("requestContext")
```

**3. appkey 机房替换**（仅 EU 机房）

```groovy
// 输入
appkey: 'com.sankuai.sailormarketing.supply.server'
// 输出 (EU)
appkey: 'com.keetapp.sailormarketing.supply.server'
```

替换规则：`com.sankuai.` → `com.keetapp.`（仅替换 `com.sankuai.` 模式，不影响其他域名）

---

### DSL 模板

**文件**: `src/template.js`  
**导出**: `getDSLTemplate()` 函数 + `DSL_TEMPLATE` 常量

提供标准的 DSL 模板，包含一个 `CustomApiConfigBuilder` 类示例，展示了所有支持的语法特性：

- 类定义和方法
- 静态方法
- 对象字面量 → Groovy Map
- 可选链操作符 (`?.`)
- 逻辑或 (`||`) → Elvis (`?:`)
- 逻辑与 (`&&`)
- 三元运算符
- ES6 模块导出

模板中的 `run` 方法接收 `props`、`requestContext`、`responseData`、`tracerContext` 四个参数，返回接口聚合配置对象。

---

### CLI 工具

**文件**: `bin/cli.js`  
**命令**: `js2groovy`

```
Usage: js2groovy [options] <input>

JavaScript到Groovy代码转换工具

Arguments:
  input                 JavaScript输入文件路径

Options:
  -o, --output <output>  输出文件路径（不指定则打印到终端）
  -v, --verbose          显示详细信息
  -V, --version          显示版本号
  -h, --help             显示帮助信息
```

使用示例：

```bash
# 转换并打印到终端
js2groovy ./dsl.js

# 转换并输出到文件
js2groovy ./dsl.js -o ./output.groovy

# 显示详细转换信息
js2groovy ./dsl.js -v -o ./output.groovy
```

---

## 语法转换映射

### 操作符转换

| JavaScript | Groovy | 说明 |
|-----------|--------|------|
| `===` | `==` | 严格相等 → 相等 |
| `!==` | `!=` | 严格不等 → 不等 |
| `\|\|` | `?:` | 逻辑或 → Elvis 操作符 |
| `&&` | `&&` | 逻辑与（保持不变） |

### 类型/函数转换

| JavaScript | Groovy | 说明 |
|-----------|--------|------|
| `const/let/var x = val` | `def x = val` | 变量声明统一为 `def` |
| `{ key: value }` | `[key: value]` | 对象字面量 → Map |
| `[:]` | `[:]` | 空对象 → 空 Map |
| `(x) => expr` | `{ x -> expr }` | 箭头函数 → 闭包 |
| `` `${x}` `` | `"${x}".toString()` | 模板字符串 → GString |
| `null` | `null` | 空值（保持不变） |

### 全局函数转换

| JavaScript | Groovy |
|-----------|--------|
| `Date.now()` | `System.currentTimeMillis()` |
| `JSON.parse(str)` | `new groovy.json.JsonSlurper().parseText(str)` |
| `decodeURIComponent(str)` | `java.net.URLDecoder.decode(str, 'UTF-8')` |
| `Number(x)` | `(x ?: 0) as BigDecimal` |

### 数组方法转换

| JavaScript | Groovy |
|-----------|--------|
| `.includes(x)` | `.contains(x)` |
| `.map(fn)` | `.collect(fn)` |

### 可选链转换

| JavaScript | Groovy |
|-----------|--------|
| `obj?.prop` | `obj?.prop` |
| `fn?.()` | `fn?.call()` |
| `arr?.includes?(x)` | `arr?.contains?.call(x)` |

---

## 支持的 AST 节点

| 节点类型 | 说明 |
|---------|------|
| `Program` | 程序根节点 |
| `ClassDeclaration` | 类声明 |
| `ClassBody` | 类体 |
| `MethodDefinition` | 方法定义 |
| `ClassMethod` | 类方法 |
| `FunctionDeclaration` | 函数声明 |
| `ReturnStatement` | 返回语句 |
| `ObjectExpression` | 对象表达式 |
| `Property` / `ObjectProperty` | 对象属性 |
| `ArrayExpression` | 数组表达式 |
| `ConditionalExpression` | 三元运算符 |
| `LogicalExpression` | 逻辑表达式 |
| `BinaryExpression` | 二元表达式 |
| `MemberExpression` | 成员表达式 |
| `OptionalMemberExpression` | 可选成员表达式 |
| `CallExpression` | 函数调用 |
| `OptionalCallExpression` | 可选函数调用 |
| `NewExpression` | new 表达式 |
| `AssignmentExpression` | 赋值表达式 |
| `VariableDeclaration` | 变量声明 |
| `VariableDeclarator` | 变量声明器 |
| `Identifier` | 标识符 |
| `Literal` / `StringLiteral` / `NumericLiteral` / `BooleanLiteral` / `NullLiteral` | 字面量 |
| `BlockStatement` | 块语句 |
| `ExpressionStatement` | 表达式语句 |
| `TemplateLiteral` | 模板字符串 |
| `TemplateElement` | 模板元素 |
| `TryStatement` | try 语句 |
| `CatchClause` | catch 子句 |
| `IfStatement` | if 语句 |
| `ObjectPattern` | 对象解构模式 |
| `ArrayPattern` | 数组解构模式 |
| `ArrowFunctionExpression` | 箭头函数 |
| `UnaryExpression` | 一元表达式 |
| `ExportNamedDeclaration` | 命名导出（忽略 export 关键字） |
| `ExportDefaultDeclaration` | 默认导出（忽略 export 关键字） |

---

## 使用示例

### 编程方式使用

```javascript
const JSToGroovyTranspiler = require('@sailorgundam/gundam-js-to-groovy');
const GroovyPostProcessor = require('@sailorgundam/gundam-js-to-groovy').GroovyPostProcessor;

// 第一阶段：JS → Groovy
const transpiler = new JSToGroovyTranspiler();
const groovyCode = transpiler.transpile(`
  export class CustomApiConfigBuilder {
    run(props, requestContext) {
      const { latitude, longitude } = requestContext.location;
      return {
        interface: "foodRecall",
        appkey: "com.sankuai.sailormarketing.supply.server",
        params: [
          {
            supplyType: props?.dataInput?.inputType || '',
            segmentIds: props?.dataInput?.foodSegmentIds || 1,
            location: {
              latitude: latitude,
              longitude: longitude,
            }
          }
        ]
      };
    }
  }
`);

// 第二阶段：Groovy 后处理
const processor = new GroovyPostProcessor();
const finalCode = processor.process(groovyCode, 'HK');

console.log(finalCode);
```

### 输出结果

```groovy
class CustomApiConfigBuilder extends Script {

    @Override
    Object run() {
        def props = binding.getVariable("props")
        def requestContext = binding.getVariable("requestContext")
        def latitude = requestContext.location.latitude
        def longitude = requestContext.location.longitude
        return [
            interface: 'foodRecall',
            appkey: 'com.sankuai.sailormarketing.supply.server',
            params: [
                [
                    supplyType: props?.dataInput?.inputType ?: '',
                    segmentIds: props?.dataInput?.foodSegmentIds ?: 1,
                    location: [
                        latitude: latitude,
                        longitude: longitude,
                    ]
                ]
            ]
        ]
    }

}
```

### 箭头函数转换

```javascript
// JavaScript
const result = arr.map(x => x.id);
const found = list.includes(item);
```

```groovy
// Groovy
def result = arr.collect({ x -> x.id })
def found = list.contains(item)
```

---

## 依赖关系

| 包名 | 版本 | 用途 |
|------|------|------|
| `@babel/parser` | ^7.23.0 | JavaScript 代码解析为 AST |
| `@babel/traverse` | ^7.23.0 | AST 遍历和验证 |
| `@babel/types` | ^7.23.0 | AST 节点类型判断 |
| `commander` | ^11.0.0 | CLI 命令行参数解析 |

### 开发依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `jest` | ^29.7.0 | 单元测试 |
