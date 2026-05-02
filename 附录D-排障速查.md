## 附录D：排障速查

### 附录引言

本附录汇总 Hermes 常见问题的快速诊断与解决方案，按场景分类，便于定位问题。阅读本附录后，你将能够：

- 快速识别 Hermes 相关的构建错误和运行时错误
- 针对 JSC → Hermes 迁移中的兼容性问题找到解决方案
- 使用 Hermes 提供的调试工具定位性能和内存问题

**前置知识**：React Native 构建流程、基本调试方法。

---

### D.1 构建错误 — Android

#### D.1.1 "hermes is not recognized" 或找不到 hermesc

| 项目 | 内容 |
|------|------|
| **错误信息** | `hermes is not recognized as an internal or external command` |
| **原因** | Hermes 二进制文件不在 PATH 中，或 `node_modules/hermes-engine` 未正确安装 |
| **解决方案** | 1. 运行 `npm install` 或 `yarn install` 确保依赖安装完整<br>2. 检查 `node_modules/hermes-engine/{platform}/bin/hermesc` 是否存在<br>3. 清除缓存后重试：`cd android && ./gradlew clean` |

#### D.1.2 Release 构建字节码编译失败

| 项目 | 内容 |
|------|------|
| **错误信息** | `hermesc` 编译阶段报错，提示语法不支持 |
| **原因** | JS 代码包含 Hermes 不支持的语法（如 `with` 语句、某些 `eval` 模式） |
| **解决方案** | 1. 查看 `hermesc` 输出中的具体错误行号<br>2. 本地用 `hermesc -emit-binary -out test.hbc bundle.js` 复现问题<br>3. 移除不支持的语法（参见 D.5 特性限制表）<br>4. 如使用了 `with` 语句，必须重写为常规作用域访问 |

#### D.1.3 NDK 构建失败（New Architecture）

| 项目 | 内容 |
|------|------|
| **错误信息** | CMake/Ninja 相关的 NDK 构建错误 |
| **原因** | 缺少 CMake 或 Ninja 构建工具 |
| **解决方案** | 1. 确保已安装 Android NDK<br>2. Windows：通过 Chocolatey 安装 CMake；确保 Visual Studio Build Tools 已安装 C++ 桌面开发组件<br>3. macOS/Linux：`brew install cmake ninja` |

#### D.1.4 Hermes 版本不匹配崩溃

| 项目 | 内容 |
|------|------|
| **错误信息** | 应用启动后立即崩溃，日志显示 Hermes 版本不匹配 |
| **原因** | hermes-engine 版本与 React Native 版本不兼容 |
| **解决方案** | 1. **使用 Bundled Hermes**（RN 0.69+）：删除 `package.json` 中手动指定的 `hermes-engine` 版本，改用 RN 自带版本<br>2. **切勿手动混用**不同版本的 hermes-engine<br>3. 运行 `npx react-native clean` 后重新安装依赖 |

---

### D.2 构建错误 — iOS

#### D.2.1 "Hermes is enabled in Podfile, but isn't enabled"

| 项目 | 内容 |
|------|------|
| **错误信息** | `Hermes is enabled in Podfile, but isn't enabled in app` |
| **原因** | 构建缓存或 Pod 安装不完整 |
| **解决方案** | 1. Xcode 中 Clean Build Folder（⇧⌘K）<br>2. 重新安装 Pods：`cd ios && pod deintegrate && pod install`<br>3. 删除 `ios/build` 目录后重新构建 |

#### D.2.2 Hermes 预编译包下载失败

| 项目 | 内容 |
|------|------|
| **错误信息** | `pod install` 时 Hermes tarball 下载超时或失败 |
| **原因** | 网络问题或当前 RN 版本无对应预编译包 |
| **解决方案** | 1. 从源码构建：`BUILD_FROM_SOURCE=true bundle exec pod install`<br>2. 检查网络连接和代理设置<br>3. 如持续失败，在 Podfile 中添加：`pod 'hermes-engine', :podspec => '../node_modules/hermes-engine/hermes-engine.podspec'` |

#### D.2.3 缺少 dSYM 调试符号

| 项目 | 内容 |
|------|------|
| **错误信息** | 崩溃栈无符号化信息 |
| **原因** | 预编译的 Hermes 不包含 dSYM 文件 |
| **解决方案** | 1. 从源码构建以获取 `Hermes.framework.dSYM`：`BUILD_FROM_SOURCE=true bundle exec pod install`<br>2. RN 0.71+：调试符号已发布到 Maven Central，`ndk-stack` 可自动符号化 |

---

### D.3 运行时错误

#### D.3.1 ReferenceError: Property 'xxx' doesn't exist

| 项目 | 内容 |
|------|------|
| **错误信息** | `ReferenceError: Property 'xxx' doesn't exist` |
| **原因** | JSC 提供了某些全局对象/方法，但 Hermes 未实现 |
| **解决方案** | 1. 在入口文件添加 polyfill：`global.xxx = require('xxx-polyfill')`<br>2. 检查是否使用了 Hermes 不支持的 API（参见 D.5）<br>3. 使用 `!!global.HermesInternal` 判断运行时环境，条件加载 polyfill |

#### D.3.2 Function.prototype.toString() 返回 [bytecode]

| 项目 | 内容 |
|------|------|
| **错误信息** | `Function.prototype.toString()` 返回 `[bytecode]` 而非源码 |
| **原因** | Hermes 设计如此——从字节码执行，不保留源码文本 |
| **解决方案** | 1. **不要依赖** `Function.prototype.toString()` 返回源码<br>2. 某些库（如依赖注入框架）会使用此方法，需要找到替代方案<br>3. 如库强依赖此行为，考虑使用 JSC 替代 Hermes |

#### D.3.3 Symbol() 描述返回空字符串

| 项目 | 内容 |
|------|------|
| **错误信息** | `Symbol().description` 返回 `''` 而非 `undefined` |
| **原因** | Hermes 已知的不兼容行为（ documented in Features.md） |
| **解决方案** | 此为已知 bug，关注后续版本更新。临时规避：`sym.description || undefined` |

---

### D.4 性能问题

#### D.4.1 启动速度仍然慢

| 项目 | 内容 |
|------|------|
| **症状** | 启用了 Hermes 但应用启动速度没有明显改善 |
| **诊断** | 1. 确认 Hermes 实际在运行：`console.log(!!global.HermesInternal)`<br>2. 确认 Release 构建使用了字节码：检查 APK/IPA 中是否包含 `.hbc` 文件<br>3. Debug 模式下 Hermes 不编译为字节码，性能差异不明显是正常的 |
| **解决方案** | 1. 使用 Release 模式测试性能<br>2. 确保构建命令使用了 Release 配置<br>3. Android：`./gradlew assembleRelease`；iOS：Xcode Release 模式 |

#### D.4.2 GC 暂停导致卡顿

| 项目 | 内容 |
|------|------|
| **症状** | 运行时偶发性帧率下降 |
| **诊断** | 1. 检查 GC 类型：`HermesInternal.getRuntimeProperties().GC`<br>2. 64 位设备应显示 `hades (concurrent)`<br>3. 32 位设备显示 `hades (incremental)` 为正常 |
| **解决方案** | 1. 确保使用 Hades GC（RN 0.70+ 默认）<br>2. 32 位设备卡顿更明显，属已知限制<br>3. 考虑调整 GC 参数：减少初始堆大小或设置最大堆限制 |

#### D.4.3 初始化时内存占用高

| 项目 | 内容 |
|------|------|
| **症状** | 应用启动后内存占用显著高于预期 |
| **诊断** | 大量临时对象在年轻代（YG）创建后被提升到老年代（OG），产生 promotion 开销 |
| **解决方案** | 1. 使用预提升策略（C++ API）：`withAllocInYoung(false)` + `withRevertToYGAtTTI(true)`<br>2. 设置合理的堆大小：`-gc-init-heap 4MiB -gc-max-heap 256MiB`<br>3. 参见 C.7 GC 调优配置模板 |

---

### D.5 特性限制速查表

以下为 Hermes **不支持或部分支持** 的 JavaScript 特性，从 JSC 迁移时务必检查：

| 特性 | 状态 | 替代方案 / 说明 |
|------|------|----------------|
| `with` 语句 | ❌ 永不支持 | 重写为常规作用域访问 |
| `eval()` 引入局部变量 | ❌ 不支持 | 避免使用 `eval`，或仅使用全局 `eval` |
| `Symbol.species` | ❌ 不支持 | 不依赖此特性 |
| `Symbol.unscopables` | ❌ 不支持（因无 `with`） | 不影响正常代码 |
| `Function.prototype.toString()` | ⚠️ 返回 `[bytecode]` | 不依赖函数源码文本 |
| Realms | ❌ 不支持 | 不使用 Realm API |
| `let`/`const` TDZ | ⚠️ 不完全符合规范 | 后续版本改进中 |
| ES6 Class | ⚠️ 实验性（`-es6-class`） | 默认使用构造函数 + 原型 |
| `WeakRef` | ⚠️ 开发中 | 关注版本更新 |
| `async`/`await` | ⚠️ 开发中（较新版本已支持） | 确认 RN 版本对应的 Hermes 版本 |
| ES Modules | ⚠️ 开发中 | 使用 CommonJS |
| `Proxy` / `Reflect` | ✅ v0.7.0+ 支持 | 可能有边缘情况不一致 |
| `Promise` | ✅ 支持（内部为字节码 polyfill） | 行为可能与原生实现有微小差异 |
| `Intl` API | ⚠️ 平台相关 | Android/iOS 行为可能不同 |

---

### D.6 JSC → Hermes 迁移检查清单

从 JSC 迁移到 Hermes 时，按以下步骤逐项检查：

1. **启用 Hermes**
   - Android：`hermesEnabled = true`（gradle.properties 或 build.gradle）
   - iOS：`:hermes_enabled => true`（Podfile）

2. **移除 JSC 依赖**
   - 删除 `package.json` 中的 `jsc-android` 依赖
   - 删除手动安装的 `hermes-engine` npm 包（使用 Bundled Hermes）

3. **清理构建缓存**
   ```bash
   # Android
   cd android && ./gradlew clean

   # iOS
   cd ios && pod deintegrate && pod install
   rm -rf ios/build

   # 清除 Metro 缓存
   npm start -- --reset-cache
   ```

4. **代码兼容性检查**
   - 全局搜索 `with (` → 必须移除
   - 全局搜索 `Function.prototype.toString` → 替换方案
   - 全局搜索 `eval(` → 确认不依赖局部变量引入
   - 检查 `Proxy` 用法 → 确认 Hermes 版本 ≥ 0.7.0
   - 检查 `Intl` 日期/数字格式 → 需要跨平台测试

5. **验证 Hermes 生效**
   ```javascript
   // 在应用中添加验证代码
   console.log('Hermes enabled:', !!global.HermesInternal);
   if (global.HermesInternal) {
     console.log('GC:', HermesInternal.getRuntimeProperties().GC);
   }
   ```

---

### D.7 崩溃符号化

#### Android 崩溃符号化

```bash
# RN 0.71+（自动符号化）
$ANDROID_HOME/ndk/<version>/ndk-stack \
  -sym ./android/app/build/intermediates/merged_native_libs/release/out/lib/arm64-v8a \
  < crash.txt

# RN 0.69–0.70（需下载符号文件）
# 从 React Native GitHub Release 下载 hermes-native-symbols-v{version}.zip
$ANDROID_HOME/ndk/<version>/ndk-stack -sym ./obj/local/arm64-v8a < crash.txt

# 实时符号化
adb logcat | $ANDROID_HOME/ndk/<version>/ndk-stack -sym ./obj/local/arm64-v8a
```

#### iOS 崩溃符号化

```bash
# 需要从源码构建以获取 dSYM
BUILD_FROM_SOURCE=true bundle exec pod install

# 使用 atos 符号化
atos -arch arm64 \
  -o Hermes.framework.dSYM/Contents/Resources/DWARF/Hermes \
  < crash_addresses.txt
```

---

### D.8 堆快照分析

#### 从 Chrome DevTools 采集

1. 连接 Chrome DevTools 到 Hermes 调试端口
2. 打开 Memory 面板 → 选择 "Heap snapshot"
3. 可选：点击垃圾桶图标回收不可达对象
4. 点击 "Take snapshot"

#### 从 JS 代码采集

```javascript
// 输出到标准输出
createHeapSnapshot();

// 保存到文件
createHeapSnapshot("/tmp/snapshot.heapsnapshot");
```

#### 从 C++ 代码采集

```cpp
void takeSnapshot(jsi::Runtime &rt) {
    rt.instrumentation().createSnapshotToFile("/tmp/filename.heapsnapshot");
}
```

#### 堆快照根分类

| 分类 | 说明 |
|------|------|
| `(Registers)` | JS 调用栈上的局部变量 |
| `(IdentifierTable)` | 属性名和 Symbol 描述 |
| `(GCScopes)` | 原生代码的 JS 值句柄 |
| `(Prototypes)` | 系统对象原型 |
| `(Custom)` | Hermes 外部的 JSI 值根 |
| `(SymbolRegistry)` | `Symbol.for()` 表 |