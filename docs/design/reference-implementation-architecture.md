# Skiloom Reference Implementation Architecture v0

状态：Accepted

对应：Skiloom Core v0 Source Spec #16

对应 ADR：[`0017-node-reference-implementation-with-native-helpers.md`](../adr/0017-node-reference-implementation-with-native-helpers.md)

本文件固定 **Skiloom reference implementation** 的技术栈、代码组织和 native helper 边界。它不改变 Skiloom Core 协议；独立实现可以使用其他语言，只要通过同一版本的 P/R/A conformance fixtures。

## 1. 首要目标

Reference implementation 优先优化：

1. 用户通过 npm 低摩擦安装和试用；
2. `npm install -g skiloom` / `npx skiloom` 可以直接进入产品；
3. 协议语义与 CLI/runtime side effects 分离；
4. 普通开发不要求 Rust/C/C++ toolchain；
5. 对确实需要高性能或底层能力的热点，允许使用预编译 native helper；
6. native helper 不能形成第二套协议 authority。

因此 v0 不采用“Rust 主程序 + npm wrapper”，也不要求纯 JavaScript 实现所有算法。总体形态是：

```text
npm / Node.js / TypeScript control plane
        +
optional prebuilt native compute helpers
```

## 2. Node.js / TypeScript 基线

Reference implementation 固定：

```text
runtime:            Node.js >= 22
primary dev line:   Node.js 24 LTS
language:           TypeScript
TypeScript mode:    strict
module system:      ESM
package manager:    npm
lockfile:           package-lock.json
public npm package: skiloom
public executable:  skiloom
```

Node 22 与 24 当前都属于受支持 LTS line；开发与 release CI 以 Node 24 LTS 为主，并至少验证最低支持线 Node 22。

生产发布使用编译后的 JavaScript；最终用户不需要安装 TypeScript compiler。

初始构建 SHOULD 使用 TypeScript compiler 直接生成 `dist/`，不因“CLI 项目通常会 bundle”而提前引入 bundler。未来若 bundling 对启动速度、package size 或 supply-chain surface 有实测收益，可作为 release engineering 变化引入，只要不改变公开行为。

## 3. 单一公开 npm package

v0 默认只发布一个用户需要理解的 package：

```text
skiloom
```

内部 Module 不因为代码边界而自动变成多个 npm package。特别是 v0 不提前发布：

```text
@skiloom/core
@skiloom/runtime
@skiloom/source-github
```

避免过早承担多 package versioning、export map、发布顺序与 public library compatibility。

当确实出现第二个独立发布物（例如 platform-specific native binary package 或 independent conformance consumer）时，repository MAY 启用 npm workspaces。Workspace 是 repository/release tooling，不是 Skiloom Core 协议。

## 4. 代码结构

目标结构：

```text
skiloom/
├── package.json
├── package-lock.json
├── tsconfig.json
├── src/
│   ├── core/
│   │   ├── package/
│   │   ├── snapshot/
│   │   ├── requirement/
│   │   ├── resolver/
│   │   ├── project/
│   │   ├── lock/
│   │   ├── activation-plan/
│   │   └── errors/
│   ├── source/
│   │   └── github/
│   ├── runtime/
│   │   ├── project/
│   │   ├── source-cache/
│   │   ├── store/
│   │   ├── activation/
│   │   └── filesystem/
│   ├── native/
│   │   └── ... explicit helper bridges only when needed
│   └── cli/
│       └── ... thin command surface
├── conformance/
│   ├── fixtures/
│   │   ├── p/
│   │   ├── r/
│   │   └── a/
│   └── runner/
├── test/
├── native/
│   └── ... native source trees only when a real helper exists
└── docs/
```

目录表达 ownership，不要求每个子目录都对应 class/package。优先形成 deep Module，而不是一文件一 abstraction。

## 5. 依赖方向

Reference implementation 的逻辑依赖方向固定为：

```text
CLI
 ↓
runtime orchestration
 ├──────────────→ GitHub source adapter
 ↓
Core interface
 ↓
protocol-semantic implementation
```

`src/core/` 是主要协议语义面。它不能直接拥有：

```text
process.argv / process.exit
interactive prompt
console-oriented UX
GitHub credential discovery
live HTTP request creation
cache/store absolute path policy
foreign project file deletion
host package installation
```

Source/runtime/CLI 把明确输入交给 Core，并消费 Core result/error。

## 6. Node 标准库优先

用户安装依赖面 SHOULD 保持小。

可直接使用 Node 标准能力的地方不额外增加 runtime dependency，例如：

```text
HTTP                 -> built-in fetch
SHA-256              -> node:crypto
filesystem/path      -> node:fs / node:path
native subprocess    -> node:child_process
basic CLI arg parsing -> built-in capability when sufficient
```

第三方依赖只用于标准库明显不应该自行实现的协议，例如 TOML/YAML parser，或经过评估确实能明显降低复杂度的能力。

Skiloom Release Requirement 不能直接把 npm `semver` grammar 当协议 oracle，因为 Core 已定义自己的 Cargo-style requirement profile。

## 7. Native helper 的角色

Native helper 可以使用 Rust、C、C++ 或其他可生成可移植 standalone executable 的实现语言。

适合 native 化的候选包括但不限于：

```text
大规模 deterministic dependency search
Git object / pack processing
超大 repository snapshot enumeration / hashing
高吞吐 archive/tree processing
```

是否 native 化必须由复杂度、性能或底层能力需求驱动；不能只因为“Rust/C++ 更快”就复制一套实现。

### 7.1 Native helper 不是 plugin/provider framework

v0 不建立通用：

```text
NativeProvider
AlgorithmPlugin
BackendRegistry
```

每个真实 native helper 在出现时建立一个最窄的内部 seam。没有第二个 implementation/真实变化点时，不预造抽象层。

### 7.2 Control plane 始终属于 Node

Native helper MUST NOT 独立拥有：

- GitHub/network access；
- credentials/secrets discovery；
- user prompts/authorization；
- Project Intent/Lock write policy；
- `.agents/skills/` destructive mutation；
- Package Store ownership policy；
- CLI rendering/exit policy。

Node runtime 负责取得/验证外部事实，并向 helper 提供明确 deterministic input。Helper 返回 deterministic result/error；Node 决定如何呈现、接受或执行 side effect。

Helper MAY 读取 Node 明确提供的 immutable input/file/stream，并 MAY 写到 Node 明确分配的 temporary/output target；它不得自行遍历任意 project/home/network 状态来补充隐式输入。

### 7.3 Protocol authority

Native code 可以实现 Core 算法，但它不是协议 authority。

Authority 顺序是：

```text
Source Spec / Accepted protocol
        ↓
versioned conformance fixtures
        ↓
TypeScript or native implementation
```

如果 TypeScript path 与 native path 对同一 fixture 给出不同结果，这是 implementation bug，不是“两个 backend 都合法”。

一个 Core algorithm 若只有 native implementation，也必须独立通过对应 conformance fixture；不要求为了形式上的 fallback 再维护一份完整 TypeScript duplicate implementation。

## 8. Native process seam

Standalone executable 是 v0 native helper 的首选形态，而不是 Node native addon。

理由：

- Rust/C/C++ 都能复用相同进程 seam；
- 不把 helper 绑定到 Node/V8 addon ABI；
- 用户机器无需 compiler/toolchain；
- helper crash/timeout 可以由 Node process boundary 隔离和报告；
- 发布物可以独立做平台签名/校验。

Node SHOULD 使用 direct spawn（不经过 shell）启动 helper，并显式控制 argv、cwd、environment、timeout/cancellation 与 stdin/stdout/stderr。

Helper IPC 必须 versioned。v0 不在没有真实 helper 前过早固定 JSON/MessagePack/二进制 framing；创建首个 helper 的 ticket 必须同时固定其最小 request/response schema 与 test vectors。

Node-API/native addon MAY 在未来针对明确性能问题采用，但不是 v0 默认 native seam。

## 9. npm 分发 native binary

Native helper 不应在用户执行 `npm install` 时现场编译。

首选分发方式：

1. CI/release pipeline 为支持的平台预编译 standalone binary；
2. platform package 使用 npm `os` / `cpu`，Linux 必要时使用 `libc` metadata；
3. 主 `skiloom` package 通过 `optionalDependencies` 声明可选 platform helper packages；
4. Node runtime 检测已安装的匹配 helper，并在有对应 implementation 时使用；
5. `npm install -g skiloom` / `npx skiloom` 仍是用户唯一需要理解的入口。

具体 platform package 名称是 release engineering 名称，不属于 Core protocol。在正式创建 npm scope/package 前必须重新验证 registry availability。

因为 npm 允许用户用 `--omit=optional` 跳过 optional dependencies，所以 **optional native accelerator 的缺失不能让一个原本有 TypeScript implementation 的 Core capability变得错误**。

若未来某项 mandatory capability 只提供 native implementation，则必须在采用前显式修改 reference implementation support policy：声明 supported platform matrix、安装行为与清晰的 `UnsupportedPlatformCapability` 类产品错误；不能借 optional dependency 的偶然存在偷偷变成 mandatory。

## 10. GitHub source implementation

v0 GitHub source adapter优先使用 Node built-in HTTP capabilities和 GitHub API/source transport，不要求用户机器安装 system `git` executable。

Core source semantics仍是：

```text
GitHub metadata/ref/tag
  -> exact commit
  -> exact repository snapshot facts
```

是否未来用 native Git object helper优化 acquisition 属于 implementation；不能改变 Source Spec 的 coordinate/redirect/tag/commit/discovery/digest语义。

## 11. Testing architecture

最高测试 seam继续是 Source Spec 已接受的：

```text
versioned fixture
  -> implementation under test
  -> canonical observable result/error
```

Reference implementation tests分三层：

1. **Conformance fixtures**：验证 P/R/A协议结果，不依赖 live GitHub；
2. **Module tests**：验证 source/runtime/native adapter自己的实现细节和错误映射；
3. **Thin end-to-end CLI tests**：验证 npm executable能够把输入交给 runtime并正确渲染/退出，不重复测试 resolver内部算法。

如果某个算法同时有 TypeScript 与 native implementation，同一 conformance fixture suite MUST 对两条 implementation path运行。

Native helper自身 MAY 使用其语言生态的 unit/property tests，但这些不能代替 repository-level conformance fixtures。

## 12. 发布与平台原则

Reference implementation的主要传播渠道是 npm，而不是 GitHub binary download。

目标用户体验：

```text
npm install -g skiloom
# 或
npx skiloom ...
```

预编译 native helper只是 npm dependency graph内部的实现细节。用户不应该为了使用 Skiloom而安装 Rust、C/C++ compiler、CMake 或 platform SDK。

Node.js v0 support floor为 22；开发/release主线使用 Node 24 LTS。未来提高 Node support floor属于 reference implementation release policy，不需要修改 Core protocol version，除非它改变 portable artifact/conformance语义。

## 13. Non-Goals

本 architecture v0 不定义：

- Skiloom Core 对其他实现语言的限制；
- public `@skiloom/core` library API；
- 通用 native plugin/provider framework；
- 用户现场编译 native helper；
- runtime 自动下载任意未通过 npm/release provenance管理的 executable；
- 将 native helper作为 Package/Skill可执行 extension机制；
- 通过 native helper绕过 Core host-side-effect boundary；
- 为没有性能证据的算法同时维护 TS/native duplicate implementation。

## 14. 一句话架构

```text
Skiloom reference implementation
= npm-distributed Node.js/TypeScript control plane
+ small deep Core interfaces
+ GitHub/runtime side-effect adapters
+ language-neutral conformance fixtures
+ optional prebuilt native compute helpers behind narrow versioned seams.
```
