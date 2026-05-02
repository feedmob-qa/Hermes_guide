## 附录A：Hermes 配置详解

### 附录引言

本附录详细列出 Hermes JavaScript 引擎的所有配置选项，涵盖编译器标志、运行时参数、CMake 构建选项以及 React Native 集成配置。阅读本附录后，你将能够：

- 理解每项 Hermes 配置的作用与默认值
- 正确配置 Android/iOS 工程中的 Hermes 选项
- 通过 CMake 和 C++ API 定制 Hermes 构建与运行时行为

**前置知识**：基本的 React Native 工程结构、Gradle 和 CocoaPods 基础。

---

### A.1 编译器标志（hermesc / hermes CLI）

Hermes 编译器通过命令行标志控制编译行为。以下按类别分组说明。

#### A.1.1 输出与优化标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `-O` | 启用 | 最大优化级别，等价于 `-Omax` |
| `-O0` | — | 禁用所有优化 |
| `-commonjs` | false | 启用 CommonJS 模块模式 |
| `-emit-binary` | Execute | 输出字节码文件（`hermesc` 必须指定此项） |
| `-out <filename>` | 无 | 指定输出文件名 |
| `-output-source-map` | false | 生成 Source Map（输出到 `<filename>.map`） |
| `-base-bytecode <filename>` | 无 | 输入基础字节码，用于增量优化 |
| `-bytecode-format` | `HBC` | 字节码格式，当前仅支持 `HBC` |
| `-g` / `-g0` / `-g1` / `-g2` / `-g3` | `-g1` | 调试信息级别：0=无，1=仅源位置，2=局部变量，3=完整 |
| `-pretty` | true | 美化输出（JSON、JS、反汇编字节码） |

**示例**：编译一个生产环境字节码文件：

```bash
hermesc -O -emit-binary -output-source-map -out bundle.hbc bundle.js
```

#### A.1.2 语法与语言特性标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `-parse-jsx` | false | 解析 JSX 语法 |
| `-parse-flow` | false | 解析 Flow 类型注解 |
| `-parse-flow-component-syntax` | false | 解析 Flow Component 语法 |
| `-parse-flow-match` | false | 解析 Flow match 语句/表达式 |
| `-parse-ts` | false | 解析 TypeScript 注解 |
| `-strict` | false | 启用严格模式 |
| `-non-strict` | false | 启用非严格模式 |
| `-block-scoping` | false | 启用块级作用域（let/const） |
| `-es6-class` | false | 启用 ES6 Class 支持（实验性） |
| `-Xes6-promise` | 平台默认 | 启用 ES6 Promise |
| `-Xes6-proxy` | 平台默认 | 启用 ES6 Proxy |
| `-Xintl` | 平台默认 | 启用 ECMA-402 Intl API |
| `-Xmicrotask-queue` | 平台默认 | 启用微任务队列 |

**示例**：编译包含 JSX 和 TypeScript 的文件：

```bash
hermesc -parse-jsx -parse-ts -O -emit-binary -out app.hbc app.js
```

#### A.1.3 运行时与调优标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `-lazy` | false | 强制全懒编译 |
| `-eager` | false | 强制全急编译 |
| `-enable-eval` | true | 启用 `eval()` 支持 |
| `-optimized-eval` | false | 对 `eval()` 启用编译器优化 |
| `-static-builtins` | AutoDetect | 内建函数静态优化：`AutoDetect`/`On`/`Off` |
| `-reuse-prop-cache` | true | 复用属性缓存条目 |
| `-pad-function-bodies-percent <N>` | 0 | 按百分比填充函数体（用于增量优化） |
| `-error-limit <N>` | 20 | 最大错误数（0=不限制） |
| `-Werror` | 按类别 | 将警告视为错误 |
| `-Wunused` | true | 报告未使用的变量 |

#### A.1.4 调试与分析标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `-dump-target <kind>` | Execute | 输出目标：`Execute`、`DumpAST`、`DumpBytecode`、`DumpIR` 等 |
| `-instrument-ir` | false | 插桩代码用于动态分析 |
| `-basic-block-profiling` | false | 启用基本块性能分析 |
| `-EmitAsyncBreakCheck` | false | 插入异步中断检查指令 |
| `-VerifyIR` | Debug=true, Release=false | 创建 IR 后进行验证 |
| `-input-source-map <file>` | 无 | 指定输入 JS 对应的 Source Map |

**示例**：反编译并查看字节码：

```bash
hermesc -dump-target=DumpBytecode -pretty input.js
```

---

### A.2 GC（垃圾回收）配置

#### A.2.1 命令行 GC 标志

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `-gc-min-heap <size>` | 0 | 最小堆大小（格式：`{K,M,G}{iB}`，如 `4MiB`） |
| `-gc-init-heap <size>` | 1MiB | 初始堆大小 |
| `-gc-max-heap <size>` | 1GiB | 最大堆大小 |
| `-gc-alloc-young` | true | 初始在年轻代分配 |
| `-gc-revert-to-yg-at-tti` | false | 在 TTI 后回退到年轻代分配 |
| `-gc-print-stats` | false | 退出时输出 GC 统计信息 |
| `-gc-before-stats` | false | 打印统计前先执行 Full GC |
| `-gc-sanitize-handles <rate>` | 0.0 | 句柄消毒概率（0.0-1.0） |
| `-occupancy-target <fraction>` | 默认 | 堆中存活数据的占比目标 |

**示例**：设置堆大小限制为 256MB：

```bash
hermes -gc-max-heap 256MiB app.hbc
```

#### A.2.2 GC 类型

Hermes 支持两种 GC 实现：

| GC 类型 | 说明 | 适用场景 |
|---------|------|----------|
| **Hades** | 现代并发/增量 GC（默认） | 64 位设备使用并发模式，32 位设备使用增量模式 |
| **GenGC** | 旧版分代 GC | 向后兼容，不推荐新项目使用 |

**运行时检测当前 GC**：

```javascript
const gcName = HermesInternal.getRuntimeProperties().GC;
// 返回 "hades (concurrent)" 或 "hades (incremental)"
```

#### A.2.3 C++ API 配置 GC

```cpp
hermes::vm::GCConfig::Builder gcConfigBuilder{};
gcConfigBuilder
    .withAllocInYoung(false)       // 预提升：直接在老年代分配
    .withRevertToYGAtTTI(true);    // TTI 后切回年轻代分配

auto runtime = hermes::makeHermesRuntime(
    hermes::vm::RuntimeConfig::Builder()
        .withGCConfig(gcConfigBuilder.build())
        .build()
);
```

---

### A.3 CMake 构建配置

从源码构建 Hermes 时可用的 CMake 选项：

| CMake 变量 | 默认值 | 说明 |
|------------|--------|------|
| `CMAKE_BUILD_TYPE` | Debug | 构建类型：`Release` 或 `Debug` |
| `HERMESVM_GCKIND` | Hades | GC 类型：`hades`（并发，推荐）或 `gengc`（旧版） |
| `HERMESVM_HEAP_SEGMENT_SIZE_KB` | 4096 | 堆段大小（KB），默认 4MiB |
| `HERMES_ENABLE_ADDRESS_SANITIZER` | OFF | 启用 AddressSanitizer |
| `HERMESVM_SANITIZE_HANDLES` | OFF | 启用句柄消毒 |

**示例**：从源码构建 Release 版本：

```bash
cmake -S hermes -B build_release -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build ./build_release
```

---

### A.4 React Native 集成配置

#### A.4.1 Android 配置

**RN 0.74+（推荐写法）**：

```groovy
// android/app/build.gradle
react {
    hermesEnabled = true   // 默认已启用，设为 false 可禁用
}
```

**RN 0.70–0.73**：

```groovy
// android/app/build.gradle
project.ext.react = [
    enableHermes: true,   // true=启用 Hermes, false=使用 JSC
    hermesFlags: ["-O", "-g0"]  // 可选：传递编译器标志
]
```

**RN 0.69（Bundled Hermes）**：

```groovy
// android/app/build.gradle
dependencies {
    if (enableHermes) {
        implementation("com.facebook.react:hermes-engine:+") {
            exclude group:'com.facebook.fbjni'
        }
    } else {
        implementation jscFlavor
    }
}
```

**gradle.properties**：

```properties
# android/gradle.properties
hermesEnabled=true
```

#### A.4.2 iOS 配置

**RN 0.70+**：

```ruby
# ios/Podfile
use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true,
  :hermes_flags => ["-O"]   # 可选：传递编译器标志
)
```

**禁用 Hermes**：

```ruby
use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => false
)
```

**从源码构建 Hermes**（当预编译包下载失败时）：

```bash
BUILD_FROM_SOURCE=true bundle exec pod install
```

**使用自定义 Hermes 构建**：

```bash
export REACT_NATIVE_OVERRIDE_HERMES_DIR=/path/to/your/hermes/repo
```

> **注意**：版本不匹配可能导致应用立即崩溃。使用 Bundled Hermes（RN 0.69+）可确保版本一致性，切勿手动混用不同版本的 hermes-engine。

#### A.4.3 运行时特性检测

```javascript
// 检测是否在 Hermes 上运行
const isHermes = !!global.HermesInternal;

if (isHermes) {
  // Hermes 特定逻辑
  const props = HermesInternal.getRuntimeProperties();
  console.log('GC:', props.GC);
  console.log('Hermes Version:', props.Release);
}
```

---

### A.5 其他运行时标志

以下标志通过 `hermes` 命令（而非 `hermesc`）使用，控制运行时行为：

| 标志 | 默认值 | 说明 |
|------|--------|------|
| `-time-limit <ms>` | 0（无限） | JS 执行超时时间（毫秒） |
| `-Xrepeat <N>` | 1 | 重复执行 N 次 |
| `-sample-profiling` | false | 启用采样性能分析 |
| `-track-io` | false | 追踪字节码 I/O |
| `-stop-after-module-init` | false | 模块加载完成后退出 |
| `-enable-hermes-internal` | true | 启用 HermesInternal 对象 |
| `-Xheap-timeline` | false | 追踪堆分配栈 |
| `-Xvm-experiment-flags <N>` | 0 | VM 实验标志（隐藏） |

**示例**：限制脚本执行时间为 5 秒：

```bash
hermes -time-limit 5000 app.hbc
```