## 附录B：Hermes 命令速查

### 附录引言

本附录提供 Hermes 工具链所有命令的快速参考，涵盖编译、执行、调试和诊断工具。阅读本附录后，你将能够：

- 快速查找 Hermes CLI 工具的正确用法和参数
- 理解 `hermesc`、`hermes`、`hdb` 等工具的区别与适用场景
- 掌握字节码分析、增量编译等高级操作的命令行操作

**前置知识**：基本的命令行操作、JavaScript/React Native 工程结构。

---

### B.1 hermesc — 字节码编译器

Hermes 的独立编译器，将 JavaScript 编译为 Hermes 字节码（`.hbc`），不执行代码。

#### 基本用法

```bash
hermesc [选项...] <输入文件...>
```

#### 常用操作速查

| 场景 | 命令 |
|------|------|
| 编译为字节码（生产） | `hermesc -O -emit-binary -out out.hbc input.js` |
| 编译为字节码（调试） | `hermesc -g -emit-binary -out out.hbc input.js` |
| 编译 CommonJS 模块 | `hermesc -commonjs -O -emit-binary -out bundle.hbc input.js` |
| 生成 Source Map | `hermesc -O -emit-binary -output-source-map -out out.hbc input.js` |
| 增量编译（基于旧字节码） | `hermesc -O -emit-binary -base-bytecode old.hbc -out new.hbc input.js` |
| 仅语法检查（不输出） | `hermesc -dump-target=DumpAST input.js` |
| 查看 IR | `hermesc -dump-target=DumpIR input.js` |
| 查看字节码 | `hermesc -dump-target=DumpBytecode -pretty input.js` |
| 编译含 JSX | `hermesc -parse-jsx -O -emit-binary -out out.hbc input.js` |
| 编译含 TypeScript | `hermesc -parse-ts -O -emit-binary -out out.hbc input.js` |
| 编译含 Flow | `hermesc -parse-flow -O -emit-binary -out out.hbc input.js` |

> **注意**：`hermesc` 不支持 `-exec` 模式，仅用于编译。

---

### B.2 hermes — 执行器与 REPL

Hermes 的主执行器，可执行字节码和 JS 源文件，也提供交互式 REPL。

#### 基本用法

```bash
hermes [选项...] <输入文件...>
      # 无参数时进入 REPL 模式
```

#### 常用操作速查

| 场景 | 命令 |
|------|------|
| 执行 JS 文件 | `hermes script.js` |
| 执行字节码文件 | `hermes script.hbc` |
| 进入 REPL | `hermes` |
| 编译并执行 | `hermes -emit-binary -out tmp.hbc script.js && hermes tmp.hbc` |
| 限制执行时间 | `hermes -time-limit 5000 script.hbc` |
| 输出 GC 统计 | `hermes -gc-print-stats script.hbc` |
| 设置最大堆内存 | `hermes -gc-max-heap 256MiB script.hbc` |
| 采样性能分析 | `hermes -sample-profiling script.hbc` |

> **注意**：`hermes` 命令同时支持编译和执行，而 `hermesc` 仅支持编译。

---

### B.3 hdb — 命令行调试器

Hermes 的命令行 JavaScript 调试器，类似 GDB 风格的操作方式。

#### 基本用法

```bash
hdb <脚本路径>
```

#### 常用调试命令

| 命令 | 说明 |
|------|------|
| `break <行号>` | 设置断点 |
| `continue` | 继续执行 |
| `step` | 单步执行（进入函数） |
| `next` | 单步执行（跳过函数） |
| `finish` | 执行到当前函数返回 |
| `backtrace` | 显示调用栈 |
| `print <表达式>` | 计算并打印表达式的值 |
| `locals` | 显示当前作用域的局部变量 |
| `delete <编号>` | 删除断点 |
| `quit` | 退出调试器 |

---

### B.4 hbcdump — 字节码反汇编器

反汇编 Hermes 字节码文件（`.hbc`），用于检查编译输出。

#### 基本用法

```bash
hbcdump <字节码文件.hbc>
```

#### 典型用途

- 检查字节码是否正确编译
- 分析函数数量和字节码体积
- 验证 Source Map 映射是否正确
- 排查编译期优化是否生效

---

### B.5 hvm — 独立虚拟机

Hermes 的独立虚拟机，仅执行字节码，不支持编译。

#### 基本用法

```bash
hvm <字节码文件.hbc>
```

与 `hermes` 命令的区别：`hvm` 更轻量，不支持 JS 源文件输入和编译功能，仅执行 `.hbc` 文件。

---

### B.6 hcdp — Chrome DevTools 协议代理

Hermes 的 CDP（Chrome DevTools Protocol）调试代理，将 Hermes 的调试协议转换为 Chrome DevTools 可识别的 CDP 格式。

#### 基本用法

```bash
hcdp <脚本路径>
```

#### 典型用途

- 在 Chrome DevTools 中调试 Hermes 脚本
- 远程调试 React Native 应用中的 Hermes 运行时
- 配合 `adb forward` 进行 Android 设备调试

---

### B.7 其他辅助工具

| 工具 | 用途 |
|------|------|
| `hbc-attribute` | 字节码属性工具 |
| `hbc-deltaprep` | 字节码增量更新准备工具 |
| `hbc-diff` | 对比两个字节码文件的差异 |
| `hbc-read-trace` | 读取/追踪字节码执行 |
| `dependency-extractor` | 提取模块依赖关系 |
| `hermes-parser` | 独立 JS/Flow/TS 解析器 |
| `hvm-bench` | VM 性能基准测试工具 |

---

### B.8 React Native 中的 Hermes 命令

在 React Native 工程中，通常不直接调用 Hermes CLI，而是通过构建系统间接使用：

| 场景 | 命令 |
|------|------|
| Android Release 构建（自动编译字节码） | `cd android && ./gradlew assembleRelease` |
| iOS Release 构建（自动编译字节码） | Xcode → Product → Archive |
| 查看 Hermes 版本 | `node -e "console.log(require('hermes-engine').version)"` |
| 手动编译 bundle 为字节码 | `hermesc -O -emit-binary -out index.hbc index.android.bundle` |
| 生成 Source Map | `hermesc -O -emit-binary -output-source-map -out index.hbc index.android.bundle` |
| 符号化 Android 崩溃 | `$ANDROID_HOME/ndk/<ver>/ndk-stack -sym ./obj/local/arm64-v8a < crash.txt` |
| 符号化 iOS 崩溃 | `atos -arch arm64 -o Hermes.framework.dSYM/Contents/Resources/DWARF/Hermes < crash.txt` |