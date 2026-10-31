# MagVar Update — iOS 15.4 兼容修复版（非官方）

> ⚠️ **本仓库不是官方项目**，仅提供兼容性补丁。版权归原作者所有。

## 基于哪个版本

| 项 | 值 |
|---|---|
| 上游版本 | **v0.196.0** |
| tag → commit | `d03b7dc8a6ebf9bd611c7e2dc91f3b35462deebc` |
| CHANGELOG 日期 | 2026-10-19 |
| 上游分支 | `beta` |
| 上游 bundle.js | 611,598 B · SHA256 `A5D6166D…4B27` |

> **上游一更新，本补丁就需要重新打。** 核对命令：
> ```bash
> curl -s https://raw.githubusercontent.com/MagicalAstrogy/MagVarUpdate/beta/artifact/bundle.js | sha256sum
> ```

## 来源

| | |
|---|---|
| 原作者 | MagicalAstrogy & StageDog |
| 原项目 | https://github.com/MagicalAstrogy/MagVarUpdate |
| 许可证 | MIT（见 [`LICENSE`](./LICENSE)） |

## 根因（比想象的更深）

原版在 iOS / Safari **16.4 以下**完全无法加载。问题是 **正则后行断言 `(?<!...)`**——这是 ES2018 特性，
Safari 直到 **16.4** 才实现，而正则字面量在**解析阶段**就要编译，抛：

```
SyntaxError: Invalid regular expression: invalid group specifier name
```

由于是 ES Module，解析失败 = 整个模块不加载，`try/catch` 和 polyfill 都救不了。

**关键：这条正则出现了两次，分布在两个文件里。**

| # | 位置 | 说明 |
|---|---|---|
| 1 | `bundle.js`（`artifact/bundle.js`） | 应用自身的孤立代理项清理 |
| 2 | **`@earendil-works/pi-ai@1.3.0/api/google-shared/+esm`** | npm 上的依赖，被 bundle 静态 import |

**只修第 1 处没有用**——模块图里只要有一个文件解析失败，整条 import 链都会挂。
所以本仓库把第 2 处依赖也修好，并**改指向本地副本**。

## 本仓库改了什么

**① 两处后行断言，同一套等价改写**

```diff
- /[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g
+ /[\uD800-\uDBFF][\uDC00-\uDFFF]|[\uD800-\uDFFF]/g
```

配合回调 `m => 2===m.length ? m : ''`：保留成对代理项、删除落单的，与原式语义完全等价
（已用固定用例 + 2000 组随机 fuzz 对拍，0 差异）。

**② 把有问题的依赖本地化**

```diff
- from 'https://testingcf.jsdelivr.net/npm/@earendil-works/pi-ai@1.3.0/api/google-shared/+esm'
+ from './vendor/google-shared.js'
```

`artifact/vendor/google-shared.js` 是原依赖的逐字节副本，**只改了那一处正则**，
12 个导出名与原版完全一致，未改动任何逻辑。

> 之所以要本地化，是因为 npm 上的文件我们改不了。改成相对路径后，它会从本仓库同目录加载。

## 目录结构

```
artifact/
├── bundle.js              611,532 B  应用主体（正则修复 + import 重定向）
├── bundle.js.LICENSE.txt      107 B  第三方声明（pinia）
└── vendor/
    └── google-shared.js    13,180 B  pi-ai 依赖副本（正则修复）
```

## 使用

```
https://cdn.jsdelivr.net/gh/Aoooooooo1/MagVarUpdate-iOS154Fix@<tag>/artifact/bundle.js
```

> ⚠️ 不要用 `raw.githubusercontent.com` —— 那边返回 `Content-Type: text/plain`，
> 浏览器会拒绝加载 ES Module。

## 已验证

| 项目 | 结果 |
|---|---|
| ESM 语法校验（两个文件） | 通过 |
| 正则等价性（含 2000 组 fuzz） | 0 失败 |
| 全模块图递归扫描（51 个依赖） | 仅 1 处，已修复 |
| 依赖导出名完整性（12 个） | 全部匹配 |
| iOS 15.4 兼容性复查 | 0 解析期致命 |

## 关于 AbortSignal

上游 bundle **本来就自带** `AbortSignal.timeout` / `AbortSignal.any` 的垫片
（在首次调用前执行），因此本仓库**不需要**额外注入，只做最小改动。

## 第三方声明

打包内含 `pinia v3.0.3`（MIT，© 2025 Eduardo San Martin Morote），
完整声明见 [`artifact/bundle.js.LICENSE.txt`](./artifact/bundle.js.LICENSE.txt)。

`artifact/vendor/google-shared.js` 派生自 `@earendil-works/pi-ai@1.3.0`。

## 反馈

本补丁由第三方制作，**问题请优先反馈原项目**。
若原作者认为本仓库不妥，请联系删除。
