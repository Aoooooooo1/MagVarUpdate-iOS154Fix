# MagVar Update — iOS 15.4 兼容修复版（非官方）

> ⚠️ **本仓库不是官方项目**，仅提供兼容性补丁。版权归原作者所有。

## 基于哪个版本

| 项 | 值 |
|---|---|
| 上游版本 | **v0.196.0** |
| tag → commit | `d03b7dc8a6ebf9bd611c7e2dc91f3b35462deebc` |
| CHANGELOG 日期 | 2026-10-19 |
| 上游分支 | `beta` |
| 原始 bundle.js | 611,598 字节 · SHA256 `A5D6166D43A78C7FAD05669864EFD76852F58EEA905814DCD89F8E1B29FA4B27` |
| 本仓库 bundle.js | 612,438 字节 · SHA256 `005AC5D694218EAE1CE60A710950EFB15D4DB27B8827A010036375BDE55D8846` |

> **上游一更新，本补丁就需要重新打。** 判断是否过期：看上游 bundle.js 的 SHA256
> 是否还是 `A5D6166D…4B27`，变了就说明代码有改动。
>
> 核对命令：
> ```bash
> curl -s https://raw.githubusercontent.com/MagicalAstrogy/MagVarUpdate/beta/artifact/bundle.js | sha256sum
> ```

## 来源

| | |
|---|---|
| 原作者 | MagicalAstrogy & StageDog |
| 原项目 | https://github.com/MagicalAstrogy/MagVarUpdate |
| 许可证 | MIT（见 [`LICENSE`](./LICENSE)） |

## 为什么需要这个补丁

原版在 iOS / Safari **16.4 以下**完全无法加载，原因有两个层次：

**1. 解析期就崩（整个文件不加载）**

`bundle.js` 中有 1 处正则后行断言 `(?<!...)`：

```js
d.replace(/[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g, '')
```

后行断言是 ES2018 特性，Safari 直到 **16.4** 才实现。正则字面量在**解析阶段**就要编译，
所以会直接抛：

```
SyntaxError: Invalid regular expression: invalid group specifier name
```

由于这是 ES Module，解析失败意味着整个模块不加载 —— `try/catch` 和运行时 polyfill 都救不了。

**2. 修好解析后仍会炸（运行时 API 缺失）**

| API | 需要 Safari | 影响 |
|---|---|---|
| `AbortSignal.timeout` | 16.0 | 超时控制失效 |
| `AbortSignal.any` | 17.4 | **所有 API 请求直接抛 TypeError** |

## 本仓库改了什么

仅两处，其余逻辑未改动任何一行：

**① 去除后行断言**

```diff
- /[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g
+ /[\uD800-\uDBFF][\uDC00-\uDFFF]|[\uD800-\uDFFF]/g
```

配合回调保留合法代理对、删除落单代理项，语义与原式完全等价
（已用固定用例 + 2000 组随机 fuzz 对拍验证，0 差异）。

**② 注入 `AbortSignal` 垫片**

在 51 条 `import` 之后、业务代码之前插入约 840 字符的 shim：

- `AbortSignal.timeout(ms)` —— 到期后 abort，`reason.name === 'TimeoutError'`
- `AbortSignal.any(signals)` —— 任一子信号 abort 则合并信号 abort，并透传 `reason`

两者都用 `typeof` 检测，**在原生支持的浏览器上完全不生效**，只做兜底。

## 使用

```
https://cdn.jsdelivr.net/gh/Aoooooooo1/MagVarUpdate-iOS154Fix@main/artifact/bundle.js
```

> ⚠️ 请勿使用 `raw.githubusercontent.com` —— 那边返回 `Content-Type: text/plain`，
> 浏览器会拒绝加载 ES Module。

## 已验证

| 项目 | 结果 |
|---|---|
| ESM 语法校验 | 通过 |
| 正则等价性（含 fuzz） | 0 失败 |
| 垫片功能测试 | 15 / 15 通过 |
| iOS 15.4 兼容性复查 | 0 解析期致命 / 0 运行时阻断 |
| jsDelivr 返回 | `application/javascript` + CORS ✅ |

## 第三方声明

打包内含 `pinia v3.0.3`（MIT，© 2025 Eduardo San Martin Morote），
完整声明见 [`artifact/bundle.js.LICENSE.txt`](./artifact/bundle.js.LICENSE.txt)。

## 反馈

本补丁由第三方制作，**问题请优先反馈原项目**。
若原作者认为本仓库不妥，请联系删除。
