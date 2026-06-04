# 《你不知道的JavaScript（中卷）》类型和语法 — 详细讲义

## 前言

本文档是 **"You Don't Know JS"**（作者 Kyle Simpson）中卷技术分享的配套详细讲义。

中卷分为两大部分：

- **第一部分：类型和语法**（Types & Grammar）
- 第二部分：异步和性能（Async & Performance）

本讲义覆盖 **第一部分的第 1-3 章**：

| 章节 | 主题 | 核心问题 |
|------|------|----------|
| 第 1 章 | 类型 | JavaScript 到底有没有"类型"？`typeof` 为什么如此混乱？ |
| 第 2 章 | 值 | 浮点数陷阱、`NaN` 的本质、值和引用的真相 |
| 第 3 章 | 原生函数 | `new String("abc")` 和 `"abc"` 有什么区别？何时该用构造函数？ |

**目标读者**：有一定经验的前端开发者，希望深入理解 JavaScript 语言内部机制，而不仅仅停留在"能用就行"的层面。

> 📖 原书 GitHub 开源地址：<https://github.com/getify/You-Dont-Know-JS>

---

## 第一章：类型

JavaScript 经常被说成是"无类型"（untyped）语言——这是不准确的。准确地说，JavaScript 是 **动态类型**（dynamically typed）语言：变量没有类型，值有类型。

### 1.1 七种内置类型

ES6 规范定义了 **七种** 内置类型：

| # | 类型 | 类别 | 示例 |
|---|------|------|------|
| 1 | `null` | 基本类型 | `null` |
| 2 | `undefined` | 基本类型 | `undefined` |
| 3 | `boolean` | 基本类型 | `true`, `false` |
| 4 | `number` | 基本类型 | `42`, `3.14`, `NaN`, `Infinity` |
| 5 | `string` | 基本类型 | `"hello"`, `'world'` |
| 6 | `object` | 引用类型 | `{}`, `[]`, `function(){}`, `new Date()` |
| 7 | `symbol` | 基本类型（ES6） | `Symbol("desc")` |

**基本类型（Primitives） vs 引用类型（Object）**

这是 JavaScript 类型系统的根本分界线：

- **基本类型**（6 种）：`null`、`undefined`、`boolean`、`number`、`string`、`symbol`。值不可变（immutable），按值传递。
- **引用类型**（1 种）：`object`。所有非基本类型的值都是对象——数组、函数、日期、正则、错误等统统是 `object` 的子类型。

> **为什么 `function` 不是独立类型？**
>
> 函数是 `object` 的子类型，它只是一个"可调用的对象"（callable object），内部拥有 `[[Call]]` 属性。虽然 `typeof function(){}` 返回 `"function"`，但这是 `typeof` 操作符的特殊行为，并不意味着 `function` 是一种独立类型。

**扩展：BigInt（ES2020）— 第 8 种类型**

ES2020 引入了 `BigInt`，用于表示任意精度的整数：

```js
typeof 42n; // "bigint"

// BigInt 是一种新的基本类型
const big = 9007199254740993n; // 超过 Number.MAX_SAFE_INTEGER
big + 1n; // 9007199254740994n

// 不能与 Number 混合运算
42n + 1; // TypeError: Cannot mix BigInt and other types
```

至此，现代 JavaScript 共有 **8 种** 内置类型：7 种基本类型 + 1 种引用类型。

### 1.2 typeof 操作符完整行为表

`typeof` 是检查值类型的主要手段，但它的结果表充满了"惊喜"：

| 值 | `typeof` 结果 | 说明 |
|------|---------------|------|
| `undefined` | `"undefined"` | 正常 |
| `true` | `"boolean"` | 正常 |
| `42` | `"number"` | 正常 |
| `"hello"` | `"string"` | 正常 |
| `{ a: 1 }` | `"object"` | 正常 |
| `Symbol()` | `"symbol"` | ES6 新增，正常 |
| `null` | `"object"` | **BUG！** |
| `function(){}` | `"function"` | **特殊！** 函数不是独立类型，但 typeof 给了特殊待遇 |
| `[1, 2, 3]` | `"object"` | 数组是对象的子类型 |
| `42n` | `"bigint"` | ES2020 新增，正常 |

注意两个异常点：

**`typeof null === "object"` — JavaScript 最著名的 Bug**

这个 Bug 的根源要追溯到 1995 年 Brendan Eich 最初实现 JavaScript 的时候。

当时 JS 引擎内部使用 **类型标签**（type tag）系统来标识值的类型，存储在值的低位 bit 中：

- `000` → `object`
- `1` → `int`（整数）
- `010` → `double`（浮点数）
- `100` → `string`
- `110` → `boolean`

而 `null` 的内部表示是 **空指针**（`0x00`），即全零——它的低位恰好是 `000`，和 `object` 的标签完全相同。

所以 `typeof null` 检查类型标签时，看到 `000`，就返回了 `"object"`。

> TC39（JavaScript 标准委员会）曾尝试修复这个 Bug（提案 [typeof null === "null"](https://web.archive.org/web/20160331031726/http://wiki.ecmascript.org/doku.php?id=harmony:typeof_null)），但由于大量现存代码依赖 `typeof null === "object"` 这一行为，提案被否决，**这个 Bug 将永远保留**。

**安全检测 `null` 的方式**：

```js
// 方式一：严格等于
const isNull = (val) => val === null;

// 方式二：利用 typeof + falsy
const isNull2 = (val) => !val && typeof val === "object";
```

**`typeof function(){} === "function"` — 有用的"偏心"**

严格来说，`function` 不是一种类型，它是 `object` 的子类型。但 `typeof` 对函数做了特殊处理，返回 `"function"` 而非 `"object"`——这反而是一个非常实用的行为，因为我们经常需要检测一个值是否是函数。

### 1.3 undefined vs undeclared 深入

这是 JavaScript 中一个极容易混淆的概念对：

```js
var a;

a;  // undefined — 已声明，未赋值
b;  // ReferenceError: b is not defined — 从未声明过
```

| | `undefined` | `undeclared` |
|------|-------------|-------------|
| 含义 | 已声明，但当前没有值 | 从未在任何可访问的作用域中声明 |
| 访问 | 返回 `undefined` | 抛出 `ReferenceError` |
| `typeof` | `"undefined"` | `"undefined"`（！） |
| 本质 | 值的缺失 | 变量的缺失 |

**关键混淆点**：`typeof` 对两者都返回 `"undefined"`！

```js
var a;

typeof a; // "undefined" — 这合理
typeof b; // "undefined" — b 从未声明！为什么不报错？
```

**为什么？** 这是 `typeof` 内置的一种 **安全防护机制**（safety guard）。当你对一个未声明的变量使用 `typeof` 时，它不会抛出 `ReferenceError`，而是安全地返回 `"undefined"`。这个行为在实际开发中非常有用（见 1.4 节）。

> **吐槽**：报错信息 `"b is not defined"` 极其容易误导人。它的真正含义是 `"b is not declared"`（未声明），而不是 `"b is undefined"`（未定义）。如果错误信息改成 `"b is not declared"` 或 `"b is not found"`，会清晰得多。

### 1.4 typeof 安全防护机制

`typeof` 对未声明变量返回 `"undefined"` 而非抛错，这个特性在以下场景中极为有用：

**场景一：特性检测（Feature Detection）**

```js
// 检测是否在 Debug 模式下（DEBUG 可能根本没定义）
if (typeof DEBUG !== "undefined") {
  console.log("调试模式已开启");
}

// 如果直接用 if (DEBUG)，当 DEBUG 未声明时会抛 ReferenceError
// if (DEBUG) { ... } // ReferenceError!
```

**场景二：Polyfill 安全检测**

```js
// 检测 atob 是否存在，不存在则 polyfill
if (typeof atob === "undefined") {
  atob = function() { /* polyfill 实现 */ };
}
```

**替代方案：检查全局对象属性**

```js
// 在浏览器中，全局变量也是 window 的属性
if (!window.atob) {
  // polyfill
}
```

但 `window.XXX` 的问题：
- **不通用**：Node.js 中没有 `window`，有的是 `global`
- **不安全**：某些环境可能重新定义了 `window`
- 无法检测非全局作用域中的变量

**场景三：依赖注入模式**

```js
// 工具库中的安全检测
function doSomethingCool() {
  var helper =
    (typeof FeatureXYZ !== "undefined") ?
    FeatureXYZ :
    function() { /* 默认实现 */ };

  var val = helper();
  // ...
}
```

**扩展：globalThis（ES2020）**

ES2020 引入了 `globalThis`，提供了一个跨环境的全局对象引用：

```js
// 以前：
// 浏览器 → window / self
// Node.js → global
// Web Worker → self

// 现在：
globalThis.atob; // 在所有环境中都能工作
```

**扩展：可选链（Optional Chaining，ES2020）**

```js
// 现代代码中，很多 typeof 检测可以用可选链替代
// 旧写法
if (typeof obj !== "undefined" && obj !== null && obj.foo) { ... }

// 新写法
if (obj?.foo) { ... }

// 但注意：可选链解决的是属性访问，不解决变量声明问题
// someUndeclaredVar?.foo  // 仍然 ReferenceError
```

### 1.5 第一章 Quiz 答案详解

**Q1：`typeof void 0` 的结果是什么？**

```js
typeof void 0; // "undefined"
```

> **为什么？** `void` 操作符对任何表达式求值后，**总是返回 `undefined`**。`void 0`、`void 1`、`void "hello"` 结果都一样。在早期 JS 中，`undefined` 不是保留字（可以被重新赋值），所以 `void 0` 被用作获取真正 `undefined` 值的可靠方式。现代严格模式下 `undefined` 不能被重新赋值了，但 `void 0` 写法在压缩工具（如 Terser）的输出中仍然常见，因为它比 `undefined` 少几个字符。

**Q2：`typeof (() => {})` 的结果是什么？**

```js
typeof (() => {}); // "function"
```

> **为什么？** 箭头函数和普通函数一样，都是可调用的对象（callable object），内部拥有 `[[Call]]` 属性。`typeof` 检测到 `[[Call]]` 就返回 `"function"`。箭头函数没有自己的 `this`、`arguments`、`super`、`new.target`，但这不影响它被 `typeof` 识别为函数。

**Q3：`typeof class C {}` 的结果是什么？**

```js
typeof class C {}; // "function"
```

> **为什么？** `class` 是构造函数的语法糖。`class C {}` 本质上创建了一个函数（构造函数），所以 `typeof` 返回 `"function"`。可以验证：`typeof C === typeof function() {}` → `true`。

**Q4：`typeof 42n` 的结果是什么？**

```js
typeof 42n; // "bigint"
```

> **为什么？** ES2020 引入的 `BigInt` 是一种新的基本类型，`typeof` 为它返回独有的 `"bigint"` 字符串。注意 `42n` 不是 `number`，不能与 `number` 混合运算。

**Q5：`typeof Symbol.iterator` 的结果是什么？**

```js
typeof Symbol.iterator; // "symbol"
```

> **为什么？** `Symbol.iterator` 是 JavaScript 内置的 Well-Known Symbol 之一，它的值就是一个 `symbol` 类型的值。所有 `Symbol(...)` 创建的值（包括内置的 `Symbol.iterator`、`Symbol.toPrimitive` 等）的 `typeof` 结果都是 `"symbol"`。

**Q6：`typeof null` 的结果是什么？**

```js
typeof null; // "object"
```

> **为什么？** 见 1.2 节详细解释——这是 JavaScript 最古老的 Bug，源于 1995 年类型标签系统中 `null` 的空指针表示（`0x00`）与 `object` 的标签（`000`）重合。这个 Bug 永远不会被修复。

**Q7：`typeof typeof 42` 的结果是什么？**

```js
typeof typeof 42; // "string"
```

> **为什么？** 分两步理解：
> 1. `typeof 42` → `"number"`（一个字符串）
> 2. `typeof "number"` → `"string"`
>
> `typeof` 操作符 **总是返回一个字符串**，所以对 `typeof` 的结果再做 `typeof`，永远得到 `"string"`。

**Q8：`typeof NaN` 的结果是什么？**

```js
typeof NaN; // "number"
```

> **为什么？** `NaN` 的全称是 "Not a Number"，但它实际上是 `number` 类型中一个特殊值，表示"无效的数学运算结果"。把 `NaN` 理解为 "Invalid Number"（无效数字）比 "Not a Number" 更准确。详见第 2 章 2.4 节。

---

## 第二章：值

这一章深入探讨 JavaScript 中各种值的行为细节——数组的隐秘陷阱、数字的浮点精度问题、特殊值的反直觉行为，以及值传递与引用传递的真相。

### 2.1 数组

JavaScript 数组是最灵活的数据容器——它可以容纳任何类型的值，没有固定长度，也没有类型约束：

```js
var a = [1, "2", [3]];

a.length;  // 3
a[0];      // 1
a[2][0];   // 3
```

#### 稀疏数组（Sparse Arrays）

数组可以有"空位"（empty slot）：

```js
var a = [];
a[0] = 1;
// a[1] 被跳过
a[2] = 3;

a[1];      // undefined（但它不是真正的 undefined！）
a.length;  // 3
```

**空位（empty slot）和 `undefined` 的区别**：

```js
var a = [1, , 3];         // 稀疏数组，a[1] 是空位
var b = [1, undefined, 3]; // 密集数组，b[1] 是 undefined 值

// 表面上看起来一样
a[1]; // undefined
b[1]; // undefined

// 但行为不同！
a.map((v, i) => i);  // [0, empty, 2] — map 跳过空位
b.map((v, i) => i);  // [0, 1, 2]     — map 不跳过 undefined
```

**为什么 `map()` 跳过空位而 `join()` 不跳过？**

这是因为它们的内部实现逻辑不同。用伪代码说明 `join` 的行为：

```js
// join 的简化实现（fakeJoin）
function fakeJoin(arr, connector) {
  var str = "";
  for (var i = 0; i < arr.length; i++) {
    if (i > 0) {
      str += connector;
    }
    if (arr[i] !== undefined) {
      str += arr[i];
    }
    // 注意：join 只是根据 length 遍历，空位被当作 undefined 处理
  }
  return str;
}
```

`join()` 按 `length` 逐个遍历，空位当 `undefined` 处理（输出空字符串）。而 `map()`、`forEach()` 等使用 `HasProperty` 检查——空位没有对应属性，被直接跳过。

> **最佳实践**：永远不要创建稀疏数组。如果需要"占位"，显式使用 `undefined` 或 `null`。

#### 字符串键名陷阱

数组的下标本质上是字符串键名。如果传入的字符串能被强制转换为十进制数字，它就会变成数字索引：

```js
var a = [];

a["13"] = 42;

a.length; // 14（！）— "13" 被转换为数字下标 13
a[13];    // 42

a["foo"] = "bar";
a.length; // 14（不变）— "foo" 不是有效数字，变成普通属性
a.foo;    // "bar"
```

> **教训**：不要在数组上设置字符串属性。需要键值对用 `object`，需要有序列表用 `array`。

#### delete 不改变 length

```js
var a = [1, 2, 3];
delete a[1];

a.length; // 3（不变！）
a;        // [1, empty, 3] — 变成了稀疏数组
```

`delete` 只是移除了属性，不会收缩数组。如果要移除元素并改变 `length`，使用 `splice()`。

#### 类数组对象（Array-like Objects）

有些对象看起来像数组（有 `length` 属性和数字索引），但不是真正的数组：

- `arguments`（函数参数对象）
- `NodeList`（DOM 查询结果）
- 字符串

**转换为真正数组的方法**：

```js
// 方式一：Array.prototype.slice（经典黑科技）
var arr1 = Array.prototype.slice.call(arguments);

// 方式二：Array.from（ES6，推荐）
var arr2 = Array.from(arguments);

// 方式三：展开运算符（ES6，最简洁）
var arr3 = [...arguments];
```

**扩展：`Array.from` 的高级用法**

`Array.from` 不仅仅是转换工具，它的第二个参数是映射函数：

```js
// 生成序列
Array.from({ length: 5 }, (_, i) => i * 2);
// [0, 2, 4, 6, 8]

// 生成初始化数组（避免稀疏数组）
Array.from({ length: 3 }, () => []);
// [[], [], []] — 三个独立的空数组引用

// 对比陷阱
new Array(3).fill([]);
// [[], [], []] — 三个是同一个引用！修改一个全变
```

### 2.2 字符串

字符串和数组看起来很像——都有 `length` 属性、`indexOf()` 方法、`concat()` 方法：

```js
var a = "foo";
var b = ["f", "o", "o"];

a.length;                 // 3
b.length;                 // 3

a.indexOf("o");           // 1
b.indexOf("o");           // 1

a.concat("bar");          // "foobar"
b.concat(["b", "a", "r"]); // ["f","o","o","b","a","r"]
```

**但字符串不是字符数组！核心区别：字符串是不可变的（immutable），数组是可变的（mutable）。**

```js
var a = "foo";
var b = ["f", "o", "o"];

a[1] = "O";
b[1] = "O";

a; // "foo"（不变！字符串不可变）
b; // ["f", "O", "o"]（改变了）
```

#### 借用数组方法

由于字符串和数组有相似的结构，我们可以"借用"数组的一些方法。但有一条铁律：**只能借用不修改原值的（non-mutating）方法**。

```js
// 可以借用的方法：join, map（不修改原值）
Array.prototype.join.call("foo", "-");
// "f-o-o"

Array.prototype.map.call("foo", function(v) {
  return v.toUpperCase() + ".";
}).join("");
// "F.O.O."

// 不能借用的方法：reverse（修改原值）
Array.prototype.reverse.call("foo");
// TypeError! 字符串不可变，无法原地反转
```

#### 字符串反转

```js
// 基本方案（仅适用于 ASCII）
"abc".split("").reverse().join("");
// "cba"
```

**Unicode 陷阱**：

```js
// 包含组合字符的字符串
"mañana".split("").reverse().join("");
// 可能输出乱码！ñ 可能是 n + ̃（组合字符），split("") 会拆开它们

// ES6 修复方案：展开运算符按码点拆分
[..."mañana"].reverse().join("");
// "anañam"（正确处理单个码点）
```

但展开运算符仍然无法处理 **ZWJ（Zero-Width Joiner）序列** 的 emoji：

```js
// 家庭 emoji 是由多个 emoji 通过 ZWJ 连接
[..."👨‍👩‍👧‍👦"].reverse().join("");
// 乱码！ZWJ 序列被拆散了
```

**扩展：Intl.Segmenter（ES2022）— 真正的字素分割**

```js
// 正确处理任意 Unicode 字符串的方式
const segmenter = new Intl.Segmenter("zh", { granularity: "grapheme" });
const segments = [...segmenter.segment("👨‍👩‍👧‍👦mañana")].map(s => s.segment);
segments.reverse().join("");
// 正确反转！每个"可见字符"（字素簇）被当作一个整体
```

> **业务场景**：用户内容截断（如显示摘要、限制字符数）必须是 Unicode 感知的。简单的 `str.slice(0, maxLen)` 可能截断 emoji 或组合字符的中间，导致页面显示乱码。应使用 `Intl.Segmenter` 按字素簇截断。

### 2.3 数字

#### IEEE 754 双精度浮点数

JavaScript 中的"数字"只有一种类型——`number`。它遵循 IEEE 754 标准的 **双精度 64 位浮点数** 格式：

```
| 1 bit 符号位 | 11 bits 指数位 | 52 bits 尾数位 |
| (sign)       | (exponent)     | (mantissa/fraction) |
```

- **1 位符号位**：0 = 正数，1 = 负数
- **11 位指数位**：表示数量级（2 的多少次方）
- **52 位尾数位**：表示精度（有效数字）

这意味着：
- JavaScript 没有真正的"整数"类型。`42` 和 `42.0` 在内部是完全相同的表示
- `42.0 === 42` → `true`
- 所有数字都是浮点数，"整数"只是没有小数部分的浮点数

#### 数字语法

```js
// 小数点前后可以省略零
var a = 0.42;
var b = .42;   // 合法，等同于 0.42

var c = 42.0;
var d = 42.;   // 合法，等同于 42.0（但不推荐，容易看错）

// 科学计数法
5E10;   // 50000000000
1.1E6;  // 1100000

// 不同进制
0xf3;    // 十六进制 → 243
0o363;   // 八进制（ES6）→ 243
0b11110011; // 二进制（ES6）→ 243
```

**数字字面量调用方法的陷阱**：

```js
42.toFixed(3);    // SyntaxError!
// 为什么？解析器把 42. 当作数字 42.0，然后遇到 toFixed 懵了

// 正确写法
(42).toFixed(3);  // "42.000"
42..toFixed(3);   // "42.000"（第一个点是小数点，第二个是属性访问）
42 .toFixed(3);   // "42.000"（空格区分小数点和属性访问）
```

**`toFixed()` 和 `toPrecision()`**：

```js
var a = 42.59;

// toFixed(n)：固定 n 位小数，返回字符串
a.toFixed(0); // "43"（四舍五入）
a.toFixed(1); // "42.6"
a.toFixed(3); // "42.590"（不足补零）

// toPrecision(n)：总共 n 位有效数字，返回字符串
a.toPrecision(1); // "4e+1"
a.toPrecision(2); // "43"
a.toPrecision(5); // "42.590"
```

#### 0.1 + 0.2 !== 0.3 — 浮点精度问题

这是 JavaScript（以及所有使用 IEEE 754 的语言）最臭名昭著的问题：

```js
0.1 + 0.2 === 0.3; // false（！）
```

**为什么？**

`0.1` 在二进制中是无限循环小数：

```
0.1（十进制）= 0.0001100110011001100110011...（二进制，无限循环）
```

由于双精度浮点数只有 52 位尾数，`0.1` 和 `0.2` 都会被截断/舍入。两个已经不精确的数相加，结果自然不精确：

```js
0.1 + 0.2; // 0.30000000000000004
```

**解决方案：机器精度（Machine Epsilon）**

ES6 定义了 `Number.EPSILON`（值为 `2^-52`，约 `2.220446049250313e-16`），表示两个可区分浮点数之间的最小差值：

```js
// 判断两个浮点数是否"足够接近"（即相等）
function numbersCloseEnoughToEqual(n1, n2) {
  return Math.abs(n1 - n2) < Number.EPSILON;
}

numbersCloseEnoughToEqual(0.1 + 0.2, 0.3); // true
```

**业务场景：金额计算**

在真实项目中，浮点精度问题最常出现在 **金额计算** 中：

```js
// ❌ 错误做法：直接用浮点数计算金额
const price = 1.99;
const quantity = 3;
const total = price * quantity;
console.log(total); // 5.970000000000001（！）
console.log(total === 5.97); // false

// ✅ 正确做法：用"分"（整数）计算，最后转换
const priceCents = 199;       // 1.99 元 = 199 分
const quantityCents = 3;
const totalCents = priceCents * quantityCents; // 597（精确！）
const display = (totalCents / 100).toFixed(2); // "5.97"
console.log(display); // "5.97"
```

> **最佳实践**：所有涉及金额的计算，在前后端之间传递 **整数（分/厘）**，仅在展示层转换为带小数点的字符串。

#### 安全整数范围

由于 52 位尾数的限制，JavaScript 能精确表示的整数有一个范围：

```js
Number.MAX_SAFE_INTEGER; // 9007199254740991（2^53 - 1）
Number.MIN_SAFE_INTEGER; // -9007199254740991

// 超出安全范围会怎样？
9007199254740992 === 9007199254740993; // true（！）
// 两个不同的数被认为相等——精度已经丢失了
```

**32 位有限制的场景——位运算**

JavaScript 的位运算符（`|`、`&`、`~`、`^`、`<<`、`>>`、`>>>`）在内部会先将操作数转换为 **32 位有符号整数**：

```js
// a | 0 是把数字截断到 32 位整数的常见技巧
2147483647 | 0;   // 2147483647（2^31 - 1，32 位最大正整数）
2147483648 | 0;   // -2147483648（溢出！）

// 所以位运算的安全范围只有 ±2^31（约 ±21 亿）
```

**扩展：BigInt 用于大整数**

```js
// BigInt 可以表示任意大的整数
const big = 9007199254740993n; // 超过 MAX_SAFE_INTEGER
big === 9007199254740993n; // true（精确！）

// 不能与 Number 混合运算
42n + 1;   // TypeError
42n + 1n;  // 43n（必须都是 BigInt）

// JSON 不支持 BigInt
JSON.stringify({ id: 42n }); // TypeError

// 需要自定义序列化
JSON.stringify({ id: 42n }, (key, value) =>
  typeof value === "bigint" ? value.toString() : value
); // '{"id":"42"}'
```

> **业务场景**：后端数据库的自增 ID 如果超过 `2^53 - 1`（如 Twitter 的 Snowflake ID），前端 JSON.parse 后会丢失精度。解决方案：**后端以字符串形式返回大 ID**，前端始终用 `string` 类型处理。

### 2.4 特殊值

#### null vs undefined

| | `null` | `undefined` |
|------|--------|-------------|
| 含义 | **空值**——有意为空 | **缺失值**——尚未赋值 |
| 语法地位 | **关键字**（不能被重新赋值） | **标识符**（可以在非严格模式下被遮蔽！） |
| `typeof` | `"object"`（Bug） | `"undefined"` |
| 典型使用 | 明确表示"这里应该有值但现在是空的" | 变量声明后的默认值 |

```js
// undefined 不是保留字！（在非严格模式下）
function foo() {
  var undefined = 2; // 允许！在这个作用域内 undefined 被覆盖
  console.log(undefined); // 2
}
foo();

// 所以有时用 void 操作符获取"真正的" undefined
void 0 === undefined; // true（在全局作用域下）
// void <任意表达式> 永远返回 undefined
```

#### NaN — "无效数字"

`NaN` 的全称 "Not a Number" 极其误导人。**它不是"非数字"，而是"无效的数学运算结果"——它仍然是 `number` 类型。**

把它理解为 **"Invalid Number"** 更合适。

```js
var a = 2 / "foo"; // NaN

typeof a;   // "number"（NaN 是 number 类型！）
```

**`NaN` 的核心特性：它是 JavaScript 中唯一一个不等于自身的值。**

```js
NaN === NaN; // false（！！！）
NaN !== NaN; // true

// 这是 IEEE 754 标准的规定，不是 JavaScript 的 Bug
```

**检测 NaN**：

```js
// ❌ 全局 isNaN() — 有 Bug
isNaN("foo");   // true（！"foo" 不是 NaN 啊！）
// 原因：isNaN() 会先把参数转换为 Number → Number("foo") = NaN → true

isNaN(undefined); // true（同样的问题）
isNaN({});        // true

// ✅ Number.isNaN()（ES6）— 正确
Number.isNaN("foo");   // false（"foo" 不是 number 类型，直接返回 false）
Number.isNaN(NaN);     // true
Number.isNaN(undefined); // false

// ✅ 利用 NaN !== NaN 的 Polyfill（简洁而精妙！）
if (!Number.isNaN) {
  Number.isNaN = function(n) {
    return n !== n; // 只有 NaN 满足"不等于自身"
  };
}
```

> **业务场景**：用户输入验证时，`parseFloat` 的结果可能是 `NaN`：
> ```js
> var input = "abc123";
> var parsed = parseFloat(input); // NaN
>
> // ❌ 错误
> if (isNaN(parsed)) { ... } // 这里恰好能工作，但...
> if (isNaN(input)) { ... }  // 这也返回 true，但原因是错的
>
> // ✅ 正确
> if (Number.isNaN(parsed)) { ... }
> ```

#### Infinity / -Infinity

```js
1 / 0;     // Infinity
-1 / 0;    // -Infinity

Number.POSITIVE_INFINITY;  // Infinity
Number.NEGATIVE_INFINITY;  // -Infinity

// 无穷与无穷的运算
Infinity + Infinity;   // Infinity
Infinity - Infinity;   // NaN（不确定形式）
Infinity * 2;          // Infinity
Infinity / Infinity;   // NaN

// 一旦到达无穷，无法回到有限数
Infinity / 1e308;      // Infinity（不会变成有限数）
```

> JavaScript 对除以零不会报错，而是返回 `Infinity` 或 `-Infinity`。这与大多数编程语言不同。

#### -0 负零

JavaScript 有一个隐秘的特殊值：**负零**。

```js
// 产生负零的运算：乘法和除法
0 / -3;    // -0
0 * -3;    // -0

// 不会产生负零的运算：加法和减法
0 - 0;     // 0（不是 -0）
-0 + 0;    // 0
```

**负零被刻意隐藏了**：

```js
var a = -0;

a.toString();          // "0"（骗人！）
String(a);             // "0"（骗人！）
JSON.stringify(a);     // "0"（骗人！）
a + "";                // "0"（骗人！）

// 但解析方向是诚实的
JSON.parse("-0");      // -0（能正确解析回来）
Number("-0");          // -0
```

**负零的比较也在"说谎"**：

```js
-0 === 0;   // true（！）
-0 > 0;     // false
-0 < 0;     // false
0 > -0;     // false
```

**如何检测负零？**

```js
function isNegZero(n) {
  n = Number(n);
  return (n === 0) && (1 / n === -Infinity);
}

isNegZero(-0);   // true
isNegZero(0);    // false
```

**为什么需要负零？** 在某些需要同时表示"大小"和"方向"的场景中，符号位保存了方向信息：

```js
// 动画中的速度
// 值 = 方向（符号）× 速率（大小）
// 速率降为零时，-0 保留了"正在向左移动"的信息
let velocity = -0; // 速率为 0，但最后一次移动方向是"左"

// 地图应用中的移动方向
// 用户停止移动时，-0 vs +0 可以知道最后一次移动方向
```

> **业务场景**：CSS 动画中的方向控制、Canvas 绘图中的运动轨迹、地图 SDK 中的移动方向——这些场景下负零是有实际意义的。

#### Object.is()（ES2015）

`Object.is()` 专门用来处理 `===` 搞不定的两个特殊值：`NaN` 和 `-0`。

```js
// === 的两个失败场景
NaN === NaN;   // false（应该是 true）
-0 === 0;      // true（应该是 false）

// Object.is() 修正了这两个
Object.is(NaN, NaN);   // true ✅
Object.is(-0, 0);      // false ✅

// 其他情况和 === 完全相同
Object.is(42, 42);     // true
Object.is("foo", "bar"); // false
```

**Polyfill**：

```js
if (!Object.is) {
  Object.is = function(v1, v2) {
    // 处理 -0
    if (v1 === 0 && v2 === 0) {
      return 1 / v1 === 1 / v2; // 1/0 = Infinity, 1/-0 = -Infinity
    }
    // 处理 NaN
    if (v1 !== v1) {
      return v2 !== v2;
    }
    // 其他情况直接 ===
    return v1 === v2;
  };
}
```

> **使用原则**：日常比较用 `===`；只有在需要区分 `NaN` 或 `-0` 时才用 `Object.is()`。不要把 `Object.is()` 当作 `===` 的通用替代品——它的语义更严格，但在大多数场景下 `===` 更直观高效。

### 2.5 值和引用

JavaScript 中没有"指针"的概念。值的传递规则很简单，但经常被误解：

- **基本类型**（null, undefined, boolean, number, string, symbol, bigint）：**值复制**
- **对象**（object, array, function）：**引用复制**

```js
// 基本类型：值复制
var a = 2;
var b = a; // b 拿到的是 2 的一个副本
b++;
a; // 2（a 不受 b 的影响）

// 对象：引用复制
var c = [1, 2, 3];
var d = c; // d 拿到的是指向同一个数组的引用
d.push(4);
c; // [1, 2, 3, 4]（c 和 d 指向同一个数组）
```

**关键理解：引用指向的是值（对象），不是变量**

JavaScript 中没有"指向变量的指针"。一个引用只是指向某个对象的"遥控器"，你可以复制遥控器（复制引用），但所有遥控器都指向同一台电视（同一个对象）。

**经典陷阱：函数参数传递**

```js
function foo(x) {
  x.push(4);
  // x: [1, 2, 3, 4]

  x = [4, 5, 6]; // x 现在指向一个新数组！
  x.push(7);
  // x: [4, 5, 6, 7]
}

var a = [1, 2, 3];
foo(a);

a; // [1, 2, 3, 4]（不是 [4, 5, 6, 7]！）
```

**为什么？** 逐步分析：

1. 调用 `foo(a)` 时，参数 `x` 拿到了 `a` 的引用的一个**副本**——两者指向同一个数组 `[1,2,3]`
2. `x.push(4)` → 通过引用修改了共享的数组 → 此时 `a` 和 `x` 都指向 `[1,2,3,4]`
3. `x = [4,5,6]` → `x` 被重新赋值，指向一个**全新的数组** → 此时 `a` 仍指向 `[1,2,3,4]`，但 `x` 指向 `[4,5,6]`
4. `x.push(7)` → 修改的是新数组 `[4,5,6,7]`，与 `a` 无关

**如果想在函数内修改原数组的内容**：

```js
function foo(x) {
  x.push(4);
  // 不要重新赋值 x，而是在原数组上操作
  x.length = 0;          // 清空原数组
  x.push(4, 5, 6, 7);    // 往原数组里填新值
}

var a = [1, 2, 3];
foo(a);
a; // [4, 5, 6, 7]（修改了原数组）
```

**浅拷贝（Shallow Copy）方法**：

```js
var original = { a: 1, b: { c: 2 } };

// 方法一：Object.assign
var copy1 = Object.assign({}, original);

// 方法二：展开运算符
var copy2 = { ...original };

// 方法三：数组用 slice
var arrCopy = [1, 2, 3].slice();

// 注意：以上都是浅拷贝！嵌套对象仍然共享引用
copy1.b.c = 99;
original.b.c; // 99（！被影响了）
```

**扩展：structuredClone（ES2022）— 原生深拷贝**

```js
var original = {
  name: "test",
  date: new Date(),
  nested: { a: 1, b: [2, 3] },
  map: new Map([["key", "value"]]),
  set: new Set([1, 2, 3]),
};

// 添加循环引用
original.self = original;

// structuredClone 完美处理
var deep = structuredClone(original);
deep.nested.a = 999;
original.nested.a; // 1（不受影响）
deep.self === deep; // true（循环引用也正确复制）
```

`structuredClone` **能处理**的类型：

- 循环引用
- `Date`、`Map`、`Set`、`RegExp`
- `ArrayBuffer`、`TypedArray`
- `Blob`、`File`、`ImageData`

`structuredClone` **不能处理**的类型：

- 函数（`TypeError`）
- DOM 节点（`DOMException`）
- 某些 `Error` 的特殊属性

**对比 JSON 深拷贝**：

| 特性 | `JSON.parse(JSON.stringify())` | `structuredClone()` |
|------|------|------|
| 循环引用 | 报错 | 支持 |
| `Date` | 变成字符串 | 保持 Date |
| `Map` / `Set` | 丢失 | 保持 |
| `undefined` | 丢失 | 保持 |
| `Infinity` / `NaN` | 变成 `null` | 保持 |
| 函数 | 丢失 | 报错 |
| 性能 | 较慢（序列化+反序列化） | 较快（结构化克隆算法） |

> **业务场景**：Redux 中要求 state 是不可变的，每次更新都要创建新对象：
> ```js
> // Redux reducer 中的不可变更新
> function reducer(state, action) {
>   switch (action.type) {
>     case "UPDATE_USER":
>       return {
>         ...state,
>         user: {
>           ...state.user,
>           name: action.payload.name,
>         },
>       };
>     default:
>       return state;
>   }
> }
> // 展开运算符（浅拷贝）在大多数 Redux 场景下足够
> // 深层嵌套考虑 Immer 库或 structuredClone
> ```

### 2.6 第二章 Quiz 答案详解

**Q1：以下代码输出什么？**

```js
var a = [1, 2, 3];
delete a[1];
console.log(a.length);   // ?
console.log(a[1]);        // ?
```

> **答案**：`3` 和 `undefined`
>
> **为什么？** `delete` 操作符只是移除了下标 `1` 处的属性，不会改变数组的 `length`。访问被删除的位置返回 `undefined`（实际是空位，不是真正的 `undefined` 值）。数组变成了稀疏数组 `[1, empty, 3]`。

**Q2：`0.1 + 0.2 === 0.3` 的结果是什么？如何正确比较？**

```js
0.1 + 0.2 === 0.3; // false
```

> **答案**：`false`
>
> **为什么？** IEEE 754 双精度浮点数中，`0.1` 和 `0.2` 都无法精确表示（二进制无限循环小数）。`0.1 + 0.2` 的实际结果是 `0.30000000000000004`，不等于 `0.3`。
>
> **正确比较**：`Math.abs((0.1 + 0.2) - 0.3) < Number.EPSILON` → `true`

**Q3：以下代码中 `a` 的值是什么？**

```js
function foo(x) {
  x.push(4);
  x = [4, 5, 6];
  x.push(7);
}
var a = [1, 2, 3];
foo(a);
console.log(a); // ?
```

> **答案**：`[1, 2, 3, 4]`
>
> **为什么？** 详见 2.5 节分析。`x = [4, 5, 6]` 让 `x` 指向了新数组，不再影响 `a`。

**Q4：以下表达式的结果是什么？**

```js
NaN === NaN;           // ?
Number.isNaN("foo");   // ?
isNaN("foo");          // ?
```

> **答案**：`false`、`false`、`true`
>
> **为什么？**
> - `NaN === NaN` → `false`：NaN 是 JS 中唯一不等于自身的值（IEEE 754 规定）
> - `Number.isNaN("foo")` → `false`：ES6 版本先检查类型，`"foo"` 不是 `number` 类型，直接 `false`
> - `isNaN("foo")` → `true`：全局版本先做 `Number("foo")` 得到 `NaN`，再判断 → `true`（Bug）

**Q5：如何区分 `0` 和 `-0`？**

```js
-0 === 0; // true — 没法用 === 区分
```

> **答案**：两种方式
> ```js
> // 方式一：利用 1/x 的符号
> function isNegZero(n) {
>   return n === 0 && (1 / n === -Infinity);
> }
>
> // 方式二：Object.is()
> Object.is(-0, 0); // false
> ```

**Q6：以下代码输出什么？**

```js
var a = new Array(3);
var b = [undefined, undefined, undefined];

console.log(a.length);  // ?
console.log(b.length);  // ?

var a2 = a.map((v, i) => i);
var b2 = b.map((v, i) => i);

console.log(a2); // ?
console.log(b2); // ?
```

> **答案**：`3`、`3`、`[empty × 3]`、`[0, 1, 2]`
>
> **为什么？** `new Array(3)` 创建的是长度为 3 的稀疏数组（3 个空位），不是 3 个 `undefined`。`map` 会跳过空位，所以 `a2` 仍然是稀疏的。而 `b` 是密集数组，每个位置都有 `undefined` 值，`map` 正常处理。

**Q7：`structuredClone` 和 `JSON.parse(JSON.stringify())` 的关键区别？**

> **答案**：
> - `structuredClone` 能处理循环引用、`Date`、`Map`、`Set`、`undefined`、`NaN`、`Infinity`
> - JSON 方案会丢失以上所有特殊值（`Date` 变字符串，`undefined` 和函数被删除，`NaN`/`Infinity` 变 `null`，循环引用直接报错）
> - `structuredClone` 不能克隆函数和 DOM 节点（会报错），JSON 方案静默忽略函数

**Q8：以下两行代码有什么区别？**

```js
var a = "foo";
var b = new String("foo");
```

> **答案**：
> - `a` 是 **基本类型** `string`，`typeof a === "string"`
> - `b` 是 **包装对象** `String`，`typeof b === "object"`
> - `a === b` → `false`（类型不同）
> - `a == b` → `true`（自动拆箱后比较）
> - 详见第 3 章

---

## 第三章：原生函数

JavaScript 有一组内置的"原生函数"（Native Functions），也叫"内建函数"（Built-in Functions）。它们既可以当作构造函数使用（`new String("abc")`），也可以当作普通函数使用（`String(123)`）——但两种用法的结果截然不同。

### 3.1 原生函数列表与 [[Class]]

JavaScript 中最常用的原生函数：

| 原生函数 | 构造函数用法 | 推荐？ | 说明 |
|----------|-------------|--------|------|
| `String()` | `new String("abc")` | **否** | 使用字面量 `"abc"` |
| `Number()` | `new Number(42)` | **否** | 使用字面量 `42` |
| `Boolean()` | `new Boolean(true)` | **否** | 使用字面量 `true` |
| `Array()` | `new Array(1,2,3)` | **否** | 使用字面量 `[1,2,3]` |
| `Object()` | `new Object()` | **否** | 使用字面量 `{}` |
| `Function()` | `new Function("...")` | **否** | 使用 `function` 声明 |
| `RegExp()` | `new RegExp("pattern")` | **视情况** | 动态模式时有用 |
| `Date()` | `new Date()` | **是** | 唯一获取日期对象的方式 |
| `Error()` | `new Error("msg")` | **是** | 获取带堆栈的错误 |
| `Symbol()` | `Symbol("desc")` | **是** | 不能用 `new`！ |

#### Object.prototype.toString.call() — 可靠的类型检测

每个对象内部都有一个 `[[Class]]` 属性（在 ES6+ 中更准确地说是 `@@toStringTag`），`Object.prototype.toString.call()` 可以访问它：

```js
Object.prototype.toString.call(null);        // "[object Null]"
Object.prototype.toString.call(undefined);   // "[object Undefined]"
Object.prototype.toString.call("abc");       // "[object String]"
Object.prototype.toString.call(42);          // "[object Number]"
Object.prototype.toString.call(true);        // "[object Boolean]"
Object.prototype.toString.call([1,2,3]);     // "[object Array]"
Object.prototype.toString.call({});          // "[object Object]"
Object.prototype.toString.call(function(){}); // "[object Function]"
Object.prototype.toString.call(/regex/);     // "[object RegExp]"
Object.prototype.toString.call(new Date());  // "[object Date]"
Object.prototype.toString.call(new Error()); // "[object Error]"
Object.prototype.toString.call(Symbol());    // "[object Symbol]"
Object.prototype.toString.call(new Map());   // "[object Map]"
Object.prototype.toString.call(new Set());   // "[object Set]"
```

这是最可靠的类型检测方式——比 `typeof` 更全面，比 `instanceof` 更不受原型链影响。

**扩展：Symbol.toStringTag（ES6）— 自定义品牌标识**

```js
class MyCollection {
  get [Symbol.toStringTag]() {
    return "MyCollection";
  }
}

Object.prototype.toString.call(new MyCollection());
// "[object MyCollection]"
```

这让自定义类也能拥有自己的"类型标签"，在调试和类型检测时非常有用。

### 3.2 封装对象包装（Boxing）

基本类型值没有属性和方法——`"abc"` 就是一个原始字符串，不是对象。那为什么 `"abc".length` 能工作？

**自动装箱（Auto-boxing）**：当你在基本类型值上访问属性或调用方法时，JavaScript 引擎会 **自动、临时** 地将它包装成对应的封装对象：

```js
var a = "abc";

a.length;        // 3
a.toUpperCase(); // "ABC"

// 引擎内部做的事情（伪代码）：
// new String(a).length       → 3
// new String(a).toUpperCase() → "ABC"
// 然后立即丢弃包装对象
```

**不要手动预先装箱！**

```js
// ❌ 不推荐
var a = new String("abc");
var b = new Number(42);
var c = new Boolean(true);

typeof a; // "object"（不是 "string"！）
typeof b; // "object"（不是 "number"！）
typeof c; // "object"（不是 "boolean"！）
```

手动装箱没有性能优势（现代引擎对自动装箱做了优化），反而会带来类型混淆问题。

**Boolean 包装对象的致命陷阱**：

```js
var a = new Boolean(false);

if (!a) {
  console.log("这行永远不会执行！");
}
// 为什么？因为 a 是一个对象，所有对象都是 truthy 的！
// 即使它包装的值是 false，对象本身在布尔上下文中是 true
```

**为什么？** 在 JavaScript 中，**所有对象都是 truthy**。`new Boolean(false)` 创建的是一个 **对象**（恰好包装了 `false` 值），而对象永远被视为 `true`。

> **业务场景**：API 返回的布尔标志如果被错误地用 `new Boolean()` 包装，会导致所有条件判断失效：
> ```js
> // 假设后端返回 { enabled: false }
> // 某个中间件错误地做了：
> var flag = new Boolean(response.enabled); // new Boolean(false)
>
> if (flag) {
>   // 这会执行！因为 flag 是对象，是 truthy
>   enableDangerousFeature(); // Bug!
> }
>
> // 正确做法：永远不要用 new Boolean()，直接用原始值
> var flag = Boolean(response.enabled); // false（不用 new）
> // 或者
> var flag = !!response.enabled; // false
> ```

### 3.3 拆封（Unboxing）

封装对象如何变回基本类型值？通过 **`valueOf()`** 方法：

```js
var a = new String("abc");
var b = new Number(42);
var c = new Boolean(true);

a.valueOf(); // "abc"
b.valueOf(); // 42
c.valueOf(); // true
```

**隐式拆箱**：在需要基本类型值的上下文中，封装对象会自动拆箱：

```js
var a = new String("abc");

// 字符串拼接触发隐式拆箱
a + "";        // "abc"（基本类型字符串）

// 比较触发隐式拆箱
a == "abc";    // true（== 允许类型转换）
a === "abc";   // false（=== 不做类型转换，类型不同直接 false）

typeof a;      // "object"
typeof (a + ""); // "string"
```

### 3.4 构造函数陷阱

#### Array() — 参数数量决定行为

```js
// 一个参数：设置长度（不是填充元素！）
var a = new Array(3);
a.length; // 3
a[0];     // undefined（实际是空位）
a;        // [empty × 3]

// 多个参数：填充元素
var b = new Array(1, 2, 3);
b; // [1, 2, 3]
```

这种不一致的行为是 JavaScript 设计的失误。

**稀疏数组的隐患（再次强调）**：

```js
var a = new Array(3); // [empty × 3]
var b = [undefined, undefined, undefined]; // [undefined, undefined, undefined]

// 看起来一样
a.length === b.length; // true

// 行为不同
a.map((v, i) => i); // [empty × 3]（map 跳过空位）
b.map((v, i) => i); // [0, 1, 2]

a.join("-"); // "--"（join 把空位当 ""）
b.join("-"); // "--"（join 把 undefined 也当 ""）
// join 的结果恰好一样，但原因不同
```

**推荐替代方案**：

```js
// ES6：Array.from — 创建密集数组
Array.from({ length: 3 }); // [undefined, undefined, undefined]

// ES6：Array.of — 行为一致的构造
Array.of(3);     // [3]（不是 [empty × 3]！）
Array.of(1,2,3); // [1, 2, 3]

// ES6：fill — 填充数组
new Array(3).fill(0); // [0, 0, 0]
```

#### Object()、Function()、RegExp() — 使用字面量

```js
// ❌ 不推荐
var obj = new Object();
obj.a = 1;
obj.b = 2;

// ✅ 推荐
var obj = { a: 1, b: 2 };

// ❌ 不推荐（极度危险：相当于 eval）
var fn = new Function("a", "b", "return a + b");

// ✅ 推荐
var fn = function(a, b) { return a + b; };
```

**RegExp() 的例外——动态模式**：

```js
// 静态正则：用字面量
var re1 = /^hello/i;

// 动态正则：必须用 RegExp()
var name = "wuxudong";
var re2 = new RegExp("^" + name + "\\b", "i");

// ES6 模板字符串让它更清晰
var re3 = new RegExp(`^${name}\\b`, "i");
```

### 3.5 有用的构造函数

有些原生函数的构造形式是真正有用的：

#### Date() — 唯一获取日期对象的方式

```js
// 必须用构造函数创建日期对象
var now = new Date();
now.getTime(); // 毫秒时间戳

// 获取当前时间戳的快捷方式
Date.now(); // 等同于 new Date().getTime()

// ES5 polyfill
if (!Date.now) {
  Date.now = function() {
    return new Date().getTime();
  };
}

// 注意：不带 new 的 Date() 返回字符串
Date();      // "Wed Jun 04 2026 ..." （字符串）
new Date();  // Date 对象
```

#### Error() — 捕获堆栈信息

```js
// 加不加 new 效果一样
var err1 = new Error("出错了");
var err2 = Error("也出错了");

// Error 最大的价值：自动捕获调用栈
console.log(err1.stack);
// Error: 出错了
//     at <file>:<line>:<column>
//     at ...

// 常见 Error 子类型
new TypeError("类型错误");       // 类型不匹配
new RangeError("范围错误");      // 值超出范围
new ReferenceError("引用错误");  // 访问未声明的变量
new SyntaxError("语法错误");     // 代码语法问题
new URIError("URI 错误");       // URI 处理错误
```

**扩展：Error.cause（ES2022）— 错误链**

```js
// ES2022 之前：手动附加原因
try {
  connectToDatabase();
} catch (err) {
  throw new Error("连接失败: " + err.message);
  // 丢失了原始错误的堆栈信息
}

// ES2022：用 cause 选项保留错误链
try {
  connectToDatabase();
} catch (err) {
  throw new Error("连接失败", { cause: err });
  // err 完整保留在 cause 属性中
}

// 使用
try {
  initApp();
} catch (err) {
  console.error(err.message);       // "连接失败"
  console.error(err.cause);         // 原始的数据库错误
  console.error(err.cause.stack);   // 原始的堆栈信息
}
```

#### Symbol() — 不能用 new

```js
// Symbol 是唯一不能用 new 的原生函数
var sym = Symbol("描述信息");
typeof sym; // "symbol"

// 不能 new
new Symbol(); // TypeError: Symbol is not a constructor

// Symbol 的核心特性：唯一性
Symbol("foo") === Symbol("foo"); // false（！即使描述相同）

// 典型用途：对象的唯一键
var MY_KEY = Symbol("myKey");
var obj = {};
obj[MY_KEY] = "secret";

Object.keys(obj);                  // []（Symbol 键不出现在常规遍历中）
Object.getOwnPropertySymbols(obj); // [Symbol(myKey)]
```

**内置 Well-Known Symbols**：

```js
// Symbol.iterator — 定义迭代行为
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }
  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return {
      next() {
        return current <= end
          ? { value: current++, done: false }
          : { done: true };
      }
    };
  }
}

[...new Range(1, 5)]; // [1, 2, 3, 4, 5]

// Symbol.toPrimitive — 自定义类型转换
// Symbol.hasInstance — 自定义 instanceof 行为
// Symbol.toStringTag — 自定义 toString 标签（见 3.1）
```

### 3.6 原生原型

每个原生函数的 `prototype` 都是该类型的一个"默认空值"：

```js
typeof Function.prototype;          // "function" — 空函数
Function.prototype();               // undefined（可以调用，啥也不干）

Array.prototype;                    // []（空数组）
RegExp.prototype.toString();        // "/(?:)/"（空正则）
String.prototype;                   // ""（空字符串，包装对象形式）
```

有些老代码会利用这些原型作为"默认值"：

```js
// 不推荐但能看到的写法
function foo(fn, regex, arr) {
  fn = fn || Function.prototype;       // 默认空函数
  regex = regex || RegExp.prototype;   // 默认空正则
  arr = arr || Array.prototype;        // 默认空数组
}
```

> **不推荐这样做**：这种写法太"聪明"了，可读性差，而且修改原型对象会影响所有实例。直接用字面量默认值更清晰：
> ```js
> function foo(fn = () => {}, regex = /(?:)/, arr = []) {
>   // ES6 默认参数，清晰明了
> }
> ```

### 3.7 第三章 Quiz 答案详解

**Q1：以下代码输出什么？**

```js
var a = new Boolean(false);
if (a) {
  console.log("会执行吗？");
}
```

> **答案**：会输出 `"会执行吗？"`
>
> **为什么？** `new Boolean(false)` 创建的是一个 **对象**，所有对象在布尔上下文中都是 truthy。即使这个对象内部包装的值是 `false`，它本身作为对象仍然是 `true`。

**Q2：`Object.prototype.toString.call([1,2,3])` 的结果？**

```js
Object.prototype.toString.call([1,2,3]); // "[object Array]"
```

> **为什么？** `Object.prototype.toString` 访问对象内部的 `[[Class]]`（或 `@@toStringTag`），数组的内部标签是 `"Array"`。这是区分数组和普通对象最可靠的方式（比 `typeof` 和 `instanceof` 都可靠）。

**Q3：以下两种写法有什么区别？**

```js
var a = Array(3);
var b = Array.of(3);
```

> **答案**：
> - `Array(3)` → `[empty × 3]`，长度为 3 的稀疏数组
> - `Array.of(3)` → `[3]`，包含一个元素 `3` 的数组
>
> `Array.of()` 的行为始终一致：传入的所有参数都作为元素，避免了 `Array()` 单参数时"设置长度"的歧义。

**Q4：`typeof new String("abc")` 和 `typeof String("abc")` 的结果？**

```js
typeof new String("abc"); // "object"
typeof String("abc");     // "string"
```

> **为什么？**
> - `new String("abc")` 创建了一个 String **包装对象**，类型是 `"object"`
> - `String("abc")` 是普通函数调用（类型转换），返回基本类型 `"string"`
>
> **规律**：`new` + 原生函数 = 包装对象；不带 `new` = 类型转换。

**Q5：为什么 `Symbol` 不能用 `new`？**

```js
new Symbol("test"); // TypeError: Symbol is not a constructor
```

> **为什么？** Symbol 被设计为 **基本类型**，不是对象。如果允许 `new Symbol()`，返回的将是一个 Symbol 包装对象，这会导致和 `new Boolean(false)` 同样的困惑（对象是 truthy 的）。ES6 规范直接禁止了 `new Symbol()`，强制开发者只使用基本类型形式。
>
> 如果确实需要 Symbol 包装对象（极罕见），可以用 `Object(Symbol("test"))`。

**Q6：动态正则表达式该用什么方式创建？**

```js
var userInput = "hello.world";
// 需要匹配包含 userInput 的字符串
```

> **答案**：使用 `new RegExp()`
> ```js
> // 注意：用户输入需要转义正则特殊字符
> function escapeRegExp(str) {
>   return str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
> }
>
> var pattern = new RegExp(escapeRegExp(userInput), "i");
> pattern.test("say hello.world!"); // true
> ```
>
> **为什么？** 正则字面量 `/pattern/` 在编写代码时就必须确定模式，无法插入变量。`new RegExp()` 接受字符串参数，可以动态构建模式。

**Q7：以下代码的问题是什么？**

```js
var arr = new Array(3).map((v, i) => i + 1);
console.log(arr); // ?
```

> **答案**：`[empty × 3]`（不是 `[1, 2, 3]`！）
>
> **为什么？** `new Array(3)` 创建的是稀疏数组（3 个空位，不是 3 个 `undefined`）。`map` 会跳过空位，所以回调函数一次都没执行。
>
> **修复**：
> ```js
> // 方案一：Array.from
> Array.from({ length: 3 }, (_, i) => i + 1); // [1, 2, 3]
>
> // 方案二：fill 先填充
> new Array(3).fill(0).map((_, i) => i + 1); // [1, 2, 3]
>
> // 方案三：展开运算符
> [...new Array(3)].map((_, i) => i + 1); // [1, 2, 3]
> ```

---

## 附录

### A. 全部 Quiz 答案速查表

| 章 | # | 题目 | 答案 |
|----|---|------|------|
| 1 | 1 | `typeof void 0` | `"undefined"` |
| 1 | 2 | `typeof (() => {})` | `"function"` |
| 1 | 3 | `typeof class C {}` | `"function"` |
| 1 | 4 | `typeof 42n` | `"bigint"` |
| 1 | 5 | `typeof Symbol.iterator` | `"symbol"` |
| 1 | 6 | `typeof null` | `"object"` |
| 1 | 7 | `typeof typeof 42` | `"string"` |
| 1 | 8 | `typeof NaN` | `"number"` |
| 2 | 1 | `delete a[1]` 后的 length | `3`（不变） |
| 2 | 2 | `0.1 + 0.2 === 0.3` | `false` |
| 2 | 3 | `foo(a)` 后 a 的值 | `[1,2,3,4]` |
| 2 | 4 | `NaN === NaN` | `false` |
| 2 | 5 | 区分 `0` 和 `-0` | `Object.is()` 或 `1/n === -Infinity` |
| 2 | 6 | `new Array(3).map(...)` | `[empty × 3]`（跳过空位） |
| 2 | 7 | `structuredClone` vs JSON | 循环引用、Date、Map/Set、undefined、NaN |
| 2 | 8 | `"foo"` vs `new String("foo")` | 基本类型 vs 包装对象 |
| 3 | 1 | `new Boolean(false)` 在 if 中 | truthy（对象永远为真） |
| 3 | 2 | `toString.call([1,2,3])` | `"[object Array]"` |
| 3 | 3 | `Array(3)` vs `Array.of(3)` | `[empty×3]` vs `[3]` |
| 3 | 4 | `new String` vs `String` | `"object"` vs `"string"` |
| 3 | 5 | 为什么 Symbol 不能 new | 防止包装对象混淆 |
| 3 | 6 | 动态正则 | `new RegExp()` |
| 3 | 7 | `new Array(3).map(...)` | 空位被跳过 |

### B. 推荐阅读

**原书与系列**
- [You Don't Know JS（GitHub 全文）](https://github.com/getify/You-Dont-Know-JS)
- [You Don't Know JS 中文翻译](https://github.com/getify/You-Dont-Know-JS/tree/1st-ed/types%20%26%20grammar)

**规范与参考**
- [MDN typeof 操作符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/typeof)
- [MDN Number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)
- [MDN Object.is()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is)
- [MDN structuredClone()](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)

**深入专题**
- [IEEE 754 浮点数可视化工具](https://float.exposed/)
- [TC39 BigInt 提案](https://github.com/tc39/proposal-bigint)
- [Intl.Segmenter 提案](https://github.com/tc39/proposal-intl-segmenter)
- [Error.cause 提案](https://github.com/nicolo-ribaudo/proposal-error-cause)
- [The history of "typeof null"](https://2ality.com/2013/10/typeof-null.html)

**工具**
- [JavaScript Visualizer 9000](https://www.jsv9000.app/) — 可视化 JS 执行过程
- [AST Explorer](https://astexplorer.net/) — 查看 JS 代码的抽象语法树
