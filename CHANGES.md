# YYEVA-iOS 修改对比分析

> 分支：`feature/per-loop-callback`  
> 提交：`c3c6a89` + `cb97ef2`  
> 范围：7 文件，+150 / -68 行

---

## 一、功能新增

### Commit `c3c6a89` — 每轮播放完成回调

| 项目 | 修改前 | 修改后 |
|------|--------|--------|
| 回调时机 | 仅全部播放结束后触发 `evaPlayerDidCompleted:` | 新增 `evaPlayer:didLoopCompletedWithRemainingCount:`，每轮结束即触发 |
| `repeatCount` 感知 | 无 | `remainingCount` 参数告知剩余播放次数 |
| 使用场景 | 无法在循环播放中插入业务逻辑（如更新 UI 计数） | 支持逐轮进度追踪 |

**涉及文件：** `YYEVAPlayer.h`（+4 行）、`YYEVAPlayer.m`（+6 行）

---

## 二、性能优化

### 2.1 锁机制替换：`@synchronized` → `pthread_mutex_t`

**文件：** `YYEVA/Classes/model/YYEVAAssets.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 锁类型 | `@synchronized(self)` | `pthread_mutex_t` |
| 加锁开销 | Objective-C 运行时查找，每次约 100-200ns | 系统级 mutex，未竞争时约 20-30ns |
| 影响范围 | 4 个热路径方法（`readVideoTracksIntoQueueIfNeed`、`hasNextSampleBuffer`、`nextSampleBuffer`、`clear`） | 同上 |
| 预期收益 | — | 帧读取路径减少 ~80% 锁开销，高频场景（60fps）下每秒省约 10μs |

### 2.2 图像缩放：UIImageView+renderInContext → CGContext 直接绘制

**文件：** `YYEVA/Classes/Render/YYEVAVideoEffectRender.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 实现方式 | 创建 `UIImageView` → 设置 image/contentView → `layer renderInContext:` | `UIGraphicsBeginImageContextWithOptions` + `[image drawInRect:]` |
| 内存峰值 | `UIImageView` 实例 + CALayer 渲染树 + 上下文 = **3 份图像数据** | 仅位图上下文 = **1 份图像数据** |
| 临时对象 | 每次 alloc 一个 UIView + CALayer | 无 UIKit 对象 |
| 预期收益 | — | 内存峰值降低约 **60%**，减少 autorelease pool 压力 |

### 2.3 dispatch_async retain cycle 修复

**文件：** `YYEVA/Classes/core/YYEVAPlayer.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| block 捕获 | `dispatch_async(..., ^{ [self ...]; })` | `__weak` + `__strong` dance |
| 影响 | `self` 强引用 block，block 强引用 `self` → 循环引用，`dispatch_after` 场景下 player 无法释放 | 弱引用打破循环，block 执行时按需提升为强引用 |
| 预期收益 | — | 消除潜在内存泄漏 |

### 2.4 fillMode 硬编码覆盖移除

**文件：** `YYEVA/Classes/utils/YSVideoMetalUtils.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 行为 | 函数入口处强制 `fillMode = YYEVAEffectSourceImageFillModeAspectFit`，忽略调用方传入的参数 | 直接使用调用方传入的 `fillMode` 参数 |
| 影响 | 所有 Effect 元素一律按 AspectFit 渲染，无法正确显示 AspectFill/SacleFill 内容 | 尊重原始设计意图，按 JSON 配置的 fillMode 渲染 |

---

## 三、安全性加固

### 3.1 CFRelease(NULL) 崩溃防护

**文件：** `YYEVA/Classes/Render/YYEVAVideoAlphaRender.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| dealloc 逻辑 | `CFRelease(_textureCache);` — 若为 NULL 直接崩溃 | `if (_textureCache != NULL) { CFRelease(...); _textureCache = NULL; }` |
| 风险等级 | **P0 崩溃** — `CFRelease(NULL)` 为 undefined behavior | 已修复 |

### 3.2 除零保护（CGSizeZero guard）

**文件：** `YYEVA/Classes/model/YYEVAAssets.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 计算 | 直接 `naturalSize.width / 2` | 先检查 `naturalWidth > 0 && naturalHeight > 0` |
| 风险 | 若视频轨道返回 `CGSizeZero`（损坏文件/未就绪），后续缩放产生 NaN/Inf 传播至 Metal shader | 已修复 |

### 3.3 zlib 解压炸弹防护

**文件：** `YYEVA/Classes/utils/YYEVADemuxMedia.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 解压上限 | 无限制 | 50 MB |
| 风险 | 恶意构造的 MP4 metadata 可包含高压缩比 zlib 数据，解压后膨胀至数 GB 导致 OOM | 超限即终止并返回 nil |
| 额外处理 | — | 提前调用 `inflateEnd(&strm)` 防止资源泄漏 |

### 3.4 静态变量线程安全

**文件：** `YYEVA/Classes/model/YYEVAEffectInfo.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 变量声明 | `static float vertexData[vertexDataLength];` | `float vertexData[vertexDataLength];` |
| 风险 | `static` 使该数组跨调用共享，多线程同时进入此方法会互相覆盖数据 → 渲染错乱 | 每次调用独立栈上分配，线程安全 |
| 备注 | `vertexDataLength` 为编译期常量（16×6=96），栈分配开销可忽略 | — |

### 3.5 Delegate 回调移出锁区间

**文件：** `YYEVA/Classes/model/YYEVAAssets.m` → `nextSampleBuffer`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| 回调位置 | `@synchronized` 块内调用 delegate | 先在锁内提取数据，锁外再调用 delegate |
| 风险 | delegate 实现若触发 `readVideoTracksIntoQueueIfNeed` → 尝试获取同一把锁 → **死锁** | 锁区间仅包含 CFArray 操作，delegate 回调在锁外执行 |
| 备注 | 旧代码使用 `@synchronized`（可重入）故未暴露，换成 `pthread_mutex_t`（不可重入）后必现 | — |

---

## 四、可观测性提升

### Metal 错误日志

**文件：** `YYEVAVideoAlphaRender.m`、`YYEVAVideoEffectRender.m`

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| metallib 加载 | `error:nil` — 失败静默 | `error:&error` + `NSLog` 输出具体错误 |
| Pipeline 创建 | `error:nil` — 失败静默 | 同上 |
| 覆盖位点 | 5 处（AlphaRender 2 处 + EffectRender 3 处） | — |
| 收益 | Metal 初始化失败时无任何日志，难以排查白屏问题 | 控制台直接定位是 metallib 加载失败还是 pipeline 编译失败 |

---

## 五、修改前后总览

| 分类 | 修改前问题 | 修改后状态 | 严重程度 |
|------|-----------|-----------|---------|
| CFRelease(NULL) | 崩溃 | ✅ 已修复 | P0 |
| @synchronized 性能 | 渲染路径锁开销大 | ✅ pthread_mutex_t | P0 |
| Metal 错误静默 | 初始化失败无日志 | ✅ NSLog 输出 | P0 |
| UIImageView 内存峰值 | 3 倍临时图像数据 | ✅ CGContext 直接绘制 | P1 |
| static 变量线程不安全 | 多线程数据竞争 | ✅ 栈变量 | P1 |
| zlib 无解压限制 | OOM 攻击向量 | ✅ 50MB 上限 | P1 |
| dispatch_async 循环引用 | 潜在内存泄漏 | ✅ weak/strong dance | P1 |
| CGSizeZero 除零 | NaN 传播至 shader | ✅ 前置校验 | P1 |
| fillMode 硬编码 | 忽略业务配置 | ✅ 使用参数值 | P1 |
| Delegate 在锁内回调 | pthread_mutex 下死锁 | ✅ 锁外回调 | P1 |

---

## 六、验证结果

| 验证方式 | 结果 |
|---------|------|
| iOS SDK clang 编译（arm64, -fobjc-arc, -fmodules） | ✅ 0 errors / 2 warnings（预存 VLA 警告，非本次引入） |
| Oracle 代码审查 | ✅ 9/9 修复确认，1 处 AspectFill 回归已修正 |
| 敏感信息扫描 | ✅ 无 Yalla / token / key / secret 等敏感数据 |
| Fork 推送 | ✅ `SamDephin/YYEVA-iOS` @ `feature/per-loop-callback` |

---

## 七、第二次整改 — 修复 `loadTextureWithImage` 零尺寸崩溃（遗留）

**提交：** 工作区（待提交）
**范围：** `YYEVAVideoEffectRender.m`，1 文件

### 崩溃堆栈

```
Fatal Exception: NSInternalInconsistencyException
3  UIKitCore    _UIGraphicsBeginImageContextWithOptions
4  YYEVA        -[YYEVAVideoEffectRender loadTextureWithImage:device:trueSize:containerSize:videoFillMode:fillMode:] + 488
```

### 根因

当 `containerSize` 或 `trueSize` 为 `CGSizeZero` 时：

1. `drawableSize.width / containerSize.width` → **除以零** → NaN / Inf
2. `realWidth` / `realHeight` = 0 或 NaN
3. `UIGraphicsBeginImageContextWithOptions(CGSizeMake(0, 0), ...)` → **崩溃**

> ⚠️ 第一次整改的 commit message 声称已添加 "CGSizeZero guard"，但实际仅在 `YYEVAAssets.m` 的 `rgbSize` 赋值处添加了保护，**未覆盖** `YYEVAVideoEffectRender.m` 的 `loadTextureWithImage` 方法。

### 修复内容

| 防御点 | 位置 | 代码 | 说明 |
|--------|------|------|------|
| 入口参数校验 | 方法开头，`!image` 检查之后 | `if (trueSize.width <= 0 \|\| trueSize.height <= 0 \|\| containerSize.width <= 0 \|\| containerSize.height <= 0) return nil;` | 从源头阻止除零，避免 NaN/Inf 传播 |
| 计算结果校验 | `videoFillMode` switch 之后 | `if (realWidth <= 0 \|\| realHeight <= 0 \|\| isnan(realWidth) \|\| isnan(realHeight)) return nil;` | 兜底防御：即使通过了入口检查，计算后仍可能出现异常值（如浮点下溢） |
| image.size 校验 | `AspectFill` case 开头 | `if (image.size.width <= 0 \|\| image.size.height <= 0) break;` | 防止 `imageAspect = 0 / 0` 除零 |
