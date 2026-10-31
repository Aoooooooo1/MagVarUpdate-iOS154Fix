# MagVarUpdate — iOS 15.4 兼容修复版（非官方）

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

## 根因

上游在 iOS / Safari **16.4 以下**完全无法加载：

```
SyntaxError: Invalid regular expression: invalid group specifier name
```

原因是**正则后行断言 `(?<!...)`**（ES2018 特性，Safari 16.4 才实现）。
正则字面量在**解析阶段**就要编译，所以整个 ES Module 加载失败，
`try/catch` 和运行时 polyfill 都无法补救。

### 它出现了 8 次，跨 4 个文件、横跨依赖链

| 文件 | 处数 | 来源 |
|---|---|---|
| `artifact/bundle.js` | 1 | 应用自身的孤立代理项清理 |
| `artifact/vendor/google-shared.js` | 1 | `@earendil-works/pi-ai@1.3.0/api/google-shared` |
| `artifact/vendor/typebox-compile.js` | 3 | `@typebox@1.3.27/build/compile` |
| `artifact/vendor/typebox-value.js` | 3 | `@typebox@1.3.27/build/value` |

> ⚠️ **只改 `bundle.js` 是没用的**——模块图里只要有一个文件解析失败，整条 import 链都会挂。
> 排查时必须**递归爬完整张模块图**（本项目共 69 个模块），不能只看第一层 import。

## 本仓库改了什么

### ① 三组正则改写（语义等价）

```diff
# bundle.js 与 google-shared.js（同一逻辑）
- /[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g
+ /[\uD800-\uDBFF][\uDC00-\uDFFF]|[\uD800-\uDFFF]/g
  配合回调 m => 2===m.length ? m : ''
  （保留成对的代理项，删除落单的）

# typebox：不能以 - 开头或结尾
- /^(?!-).*(?<!-)$/
+ /^(?!-)(?!.*-$).*$/

# typebox：邮箱域名标签不能以 - 结尾（出现 2 次）
- (?:[\p{L}\p{N}-]{0,62})(?<!-)
+ (?:[\p{L}\p{N}-]{0,61}[\p{L}\p{N}])?
```

已用固定用例 + 2000 组随机 fuzz 对拍验证，**0 差异**。

### ② 依赖本地化

npm 上的文件无法修改，因此把受影响的依赖复制到 `artifact/vendor/`，
改好正则后再把引用指向相对路径：

```diff
# bundle.js
- https://cdn.jsdelivr.net/npm/@earendil-works/pi-ai@1.3.0/+esm
+ ./vendor/pi-ai.js

- https://cdn.jsdelivr.net/npm/@earendil-works/pi-ai@1.3.0/api/google-shared/+esm
+ ./vendor/google-shared.js

# vendor/pi-ai.js
- /npm/typebox@1.3.27/+esm          → ./typebox.js
- /npm/typebox@1.3.27/compile/+esm  → ./typebox-compile.js
- /npm/typebox@1.3.27/value/+esm    → ./typebox-value.js
```

其余 49 个依赖仍走 `cdn.jsdelivr.net`，它们均不含后行断言。

> `vendor/typebox.js` 本身没有后行断言，一并本地化是为了保持引用关系自洽。

## 目录结构

```
artifact/
├── bundle.js              611,188 B   SHA256 3F485D14…51B
├── bundle.js.LICENSE.txt      107 B   第三方声明（pinia）
└── vendor/
    ├── google-shared.js    13,180 B   pi-ai 的 API 模块
    ├── pi-ai.js            46,014 B   pi-ai 入口
    ├── typebox.js          77,028 B   typebox 入口
    ├── typebox-compile.js 126,949 B   含 3 处后行断言
    └── typebox-value.js   115,411 B   含 3 处后行断言
```

## 使用

```
https://cdn.jsdelivr.net/gh/Aoooooooo1/MagVarUpdate-iOS154Fix@v0.196.0-ios154.4/artifact/bundle.js
```

> ⚠️ 不要用 `raw.githubusercontent.com` —— 返回 `Content-Type: text/plain`，
> 浏览器会拒绝加载 ES Module。

## 已验证

| 项目 | 结果 |
|---|---|
| ESM 语法校验（6 个文件） | 全部通过 |
| 正则等价性（固定用例 + 2000 组 fuzz） | 0 失败 |
| 全模块图递归扫描（69 个） | 8 处已全部修复 |
| 依赖导出名完整性 | typebox 16 个、google-shared 12 个，全部匹配 |
| **iOS 真机验证** | **模块加载成功 + 面板正常显示** |

## 关于 AbortSignal

上游 bundle **本来就自带** `AbortSignal.timeout` / `AbortSignal.any` 的垫片
（在首次调用前执行），因此本仓库**不需要**额外注入。

## 第三方声明

- 打包内含 `pinia v3.0.3`（MIT，© 2025 Eduardo San Martin Morote），
  完整声明见 [`artifact/bundle.js.LICENSE.txt`](./artifact/bundle.js.LICENSE.txt)
- `artifact/vendor/` 下的文件派生自 `@earendil-works/pi-ai@1.3.0` 与 `@typebox@1.3.27`，
  仅修改上述正则部分，未改动任何逻辑

## 反馈

本补丁由第三方制作，**问题请优先反馈原项目**。
若原作者认为本仓库不妥，请联系删除。
