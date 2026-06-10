# moon build --target native 因 core bundle 重复定义而失败

[![CI: core-bundle double-definition](https://github.com/conglinyizhi/moonbit-bug-core-bundle/actions/workflows/ci.yml/badge.svg)](https://github.com/conglinyizhi/moonbit-bug-core-bundle/actions/workflows/ci.yml)

## 环境

MoonBit 0.1.20260608 (60bc8c3 2026-06-08) / moonc v0.10.0，Linux x86_64

## 问题现象

`moon check --target native` 可以通过，但 `moon build --target native` 会报 GCC 链接错误。

```text
moon check --target native  # ✅ 通过
moon build --target native  # ❌ 失败
```

具体错误：

```
/home/abc/.moon/lib/core/abort/abort.mbt:54:9: error: redefinition of '_M0FPC15abort5abortGuE'
/home/abc/.moon/lib/core/abort/abort.mbt:49:19: error: '_M0L3msgS1' undeclared (did you mean '_M0L3msgS2'?)
```

## 怎么发现的

在把 cirrinx-cli（一个 GitHub issue/PR 管理工具）从 Go 重写到 MoonBit 时，auth 和 gh 两个包都在 `moon check --target native` 下零错误通过了类型检查，但 `moon build` 始终报错。一开始以为是自己的代码写错了，排查了很久才发现是工具链层面的问题。

逐步排除后确定：一个只 `println("hello")` 的最小项目，仅依赖 `moonbitlang/async@0.19.2`，就能稳定复现。

## 根因

MoonBit 的 native build 流程中，编译器把项目所有依赖**合进一个** `main.c` 文件（~176K 行），其中包括了 `~/.moon/lib/core/` 下 core 库的 `.mbt` 源码。

但同时，链接阶段还会带上预编译的 core bundle `libmoonbitrun.o`。

**两个来源都包含了相同的函数定义**（`abort`、`panic_impl` 及其各种泛型变体），GCC 在链接时报 `redefinition` 错误。

`moon check` 走的是 Moon IR 层（类型检查），不涉及 C 编译和链接，所以不受影响。

## CI 结果

GitHub Actions 每次提交会自动运行以下指令，以验证问题是否仍然存在：

```yaml
- moon check --target native   # 应 ✅
- moon build --target native   # 应 ❌（但 CI 容忍失败）
- moon build --target wasm     # 应 ✅
```

详细 workflow 见 `.github/workflows/ci.yml`。

## 关联仓库

- [moonbit-bug-strconv-path](https://github.com/conglinyizhi/moonbit-bug-strconv-path) — core 库 `internal/strconv` 旧路径残留导致的符号名冲突，与此 bug 同根同源，都是在 core 源码被重复编译到 `main.c` 的基础上多出来的连带问题

## 项目结构

```
moon.mod              # 仅依赖 moonbitlang/async@0.19.2
moon.pkg              # 空
cmd/main/
├── moon.pkg          # 导入 moonbitlang/async/http
└── main.mbt          # 一行 println("hello")
.github/workflows/
└── ci.yml            # CI 自动验证
```

## 不受影响的目标

- `moon build --target wasm` 不受影响（wasm 不走 `libmoonbitrun.o`）
- `moon check --target native` 不受影响

## 临时绕过

类型检查用 `moon check --target native`，native 构建暂时无法使用。
