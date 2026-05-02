## 附录C：常用配置模板

### 附录引言

本附录提供 Hermes 在各种场景下的完整配置模板，可直接复制使用。阅读本附录后，你将能够：

- 快速搭建 React Native + Hermes 工程
- 根据场景选择合适的配置组合
- 直接复制模板并根据注释修改参数

**前置知识**：React Native 工程结构、Gradle 基础、CocoaPods 基础。

---

### C.1 React Native 0.74+ 默认配置（推荐）

RN 0.74+ 使用新的 Gradle 配置块，Hermes 默认启用。

#### Android（build.gradle）

```groovy
// android/app/build.gradle

react {
    // Hermes 已默认启用，以下为可选项

    // 禁用 Hermes（回退到 JSC）
    // hermesEnabled = false

    // 传递 Hermes 编译器标志（仅在需要自定义编译时使用）
    hermesFlags = [
        "-O",    // 最大优化
        "-g0"    // Release 构建不包含调试信息（减小包体积）
    ]
}
```

#### iOS（Podfile）

```ruby
# ios/Podfile

use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true,
  # 可选：传递编译器标志
  :hermes_flags => ["-O"]
)

target 'YourApp' do
  # ... 其他依赖
end
```

---

### C.2 RN 0.70–0.73 配置

#### Android（build.gradle）

```groovy
// android/app/build.gradle

project.ext.react = [
    enableHermes: true,                     // 启用 Hermes
    hermesFlags: ["-O", "-g0"]              // 可选：编译器标志
]

def enableHermes = project.ext.react.get("enableHermes", true)

dependencies {
    if (enableHermes) {
        def hermesPath = "../../node_modules/hermes-engine/android/"
        debugImplementation files(hermesPath + "hermes-debug.aar")
        releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
}
```

#### iOS（Podfile）

```ruby
# ios/Podfile

use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true
)
```

---

### C.3 RN 0.69 Bundled Hermes 配置

```groovy
// android/app/build.gradle

dependencies {
    if (enableHermes) {
        // 使用 Bundled Hermes（推荐，确保版本一致）
        implementation("com.facebook.react:hermes-engine:+") {
            exclude group:'com.facebook.fbjni'
        }
    } else {
        implementation jscFlavor
    }
}
```

> **重要**：切勿手动混用不同版本的 `hermes-engine`。RN 0.69+ 采用 Bundled Hermes 模型，版本由 React Native 统一管理，版本不匹配可能导致应用崩溃。

---

### C.4 自定义 Hermes 构建配置

当需要使用本地修改版 Hermes 时，通过环境变量覆盖：

```bash
# 指定自定义 Hermes 源码目录
export REACT_NATIVE_OVERRIDE_HERMES_DIR=/path/to/your/hermes/repo

# 然后正常构建
cd android && ./gradlew assembleRelease
```

#### 从源码构建 Hermes（CMake）

```bash
# Debug 构建
cmake -S hermes -B build -G Ninja
cmake --build ./build

# Release 构建
cmake -S hermes -B build_release -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build ./build_release

# 使用 AddressSanitizer（调试内存问题）
cmake -S hermes -B asan_build -G Ninja -DHERMES_ENABLE_ADDRESS_SANITIZER=ON
cmake --build ./asan_build

# 使用旧版 GenGC（不推荐，仅用于兼容性测试）
cmake -S hermes -B build -G Ninja -DHERMESVM_GCKIND=gengc
cmake --build ./build
```

---

### C.5 Metro Bundler 配置

RN 0.70+ 的 Metro 默认配置已适配 Hermes，通常无需额外配置：

```javascript
// metro.config.js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);

module.exports = mergeConfig(defaultConfig, {
  // Hermes 字节码在 Release 构建时由 Metro 自动生成
  // 无需手动配置
});
```

如需自定义 Source Map 输出路径：

```javascript
// metro.config.js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);

module.exports = mergeConfig(defaultConfig, {
  serializer: {
    // 自定义 Source Map 输出
    getModulesRunBeforeMainModule() {
      return [];
    },
  },
});
```

---

### C.6 Release 构建配置模板

#### Android Release（gradle.properties）

```properties
# android/gradle.properties

# 启用 Hermes
hermesEnabled=true

# Release 构建优化（Hermes 自动使用 -O 标志）
# 以下为额外 Gradle 优化选项
android.enableJetifier=false
android.useAndroidX=true
org.gradle.jvmargs=-Xmx4096m
org.gradle.parallel=true
org.gradle.caching=true
```

#### Release 构建命令

```bash
# Android Release
cd android && ./gradlew assembleRelease

# iOS Release（命令行）
xcodebuild -workspace ios/YourApp.xcworkspace \
  -scheme YourApp \
  -configuration Release \
  -destination generic/platform=iOS \
  -archivePath build/YourApp.xcarchive \
  archive
```

---

### C.7 GC 调优配置模板

#### C++ 层面调优（适用于自定义 Hermes 集成）

```cpp
// 默认配置（适用于大多数应用）
auto runtime = hermes::makeHermesRuntime(
    hermes::vm::RuntimeConfig::Builder().build()
);

// 初始化密集型应用（如启动时大量对象创建）
// 预提升策略：直接在老年代分配，减少 YG 提升 overhead
auto runtime = hermes::makeHermesRuntime(
    hermes::vm::RuntimeConfig::Builder()
        .withGCConfig(
            hermes::vm::GCConfig::Builder()
                .withMinHeapSize(0)
                .withInitHeapSize(1024 * 1024)       // 1MB 初始堆
                .withMaxHeapSize(1024 * 1024 * 512)   // 512MB 最大堆
                .withAllocInYoung(false)              // 预提升：直接老年代分配
                .withRevertToYGAtTTI(true)            // TTI 后回退年轻代
                .build()
        )
        .build()
);
```

#### 命令行 GC 调优

```bash
# 限制最大堆内存为 256MB
hermes -gc-max-heap 256MiB app.hbc

# 设置初始堆为 4MB
hermes -gc-init-heap 4MiB -gc-max-heap 256MiB app.hbc

# 输出 GC 统计信息
hermes -gc-print-stats app.hbc

# 先执行 Full GC 再输出统计
hermes -gc-print-stats -gc-before-stats app.hbc
```

---

### C.8 调试构建配置模板

#### 带完整调试信息的 Debug 构建

```groovy
// android/app/build.gradle
project.ext.react = [
    enableHermes: true,
    hermesFlags: [
        "-g3",              // 完整调试信息
        "-O0",              // 禁用优化（方便逐步调试）
        "-enable-eval"      // 启用 eval 支持
    ]
]
```

#### iOS Debug 构建（Podfile）

```ruby
# ios/Podfile
use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true,
  :hermes_flags => ["-g3", "-O0"]   # 完整调试信息，禁用优化
)
```

#### CMake Debug 构建（带 AddressSanitizer）

```bash
cmake -S hermes -B debug_build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DHERMES_ENABLE_ADDRESS_SANITIZER=ON

cmake --build ./debug_build
```

---

### C.9 运行时特性检测模板

```javascript
// utils/hermesDetection.js

const isHermes = () => !!global.HermesInternal;

const getHermesInfo = () => {
  if (!isHermes()) {
    return { engine: 'JSC', version: null, gc: null };
  }

  const props = HermesInternal.getRuntimeProperties();
  return {
    engine: 'Hermes',
    version: props.Release || 'unknown',
    gc: props.GC || 'unknown',
  };
};

// 条件性 polyfill
if (isHermes()) {
  // Hermes 环境下可能需要补充的全局对象
  // 例如：如果项目需要 Proxy（v0.7.0+ 已支持）
  // 例如：如果需要完整的 Intl 支持
}

export { isHermes, getHermesInfo };
```