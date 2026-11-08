# YYText iOS 27 适配记录

日期：2026-11-07。基线：本 fork 的 `858d28f`（tag `1.0.9`），分支 `codex/yytext-writing-direction`。本次适配提交发布为测试 tag `1.0.0`，保留已有 `1.0.9` 不变。

## 本次修改

`YYText/Component/YYTextKeyboardManager.m` 的 `_notifyAllObservers` 原来在所有系统的回调末尾执行：

```objc
_fromOrientation = app.statusBarOrientation;
```

但 `_fromOrientation` 的唯一读取位置，是同一方法内 `((int)[self _systemVersion]) == 7` 的 iPad 浮动键盘旋转补丁。因此本次仅给末尾赋值添加相同的 iOS 7 条件：

```objc
// Only the iOS 7 floating-keyboard workaround consumes this value.
if (((int)[self _systemVersion]) == 7) {
    _fromOrientation = app.statusBarOrientation;
}
```

这样 iOS 8 及以后不再从该管理器读取 `UIApplication.statusBarOrientation`。这不是将方向改为另一种推算，而是停止计算现代系统从未使用的状态。无需新增 Scene 方向读取。若未来其他逻辑确实需要方向，应从相关 view/window 的 `windowScene` 取；本机 iOS 27 SDK 中 `effectiveGeometry` 自 iOS 16 可用，`windowScene.interfaceOrientation` 自 iOS 26 废弃。

iOS 7 的两个读取及其顺序仍然保留，包括只有键盘 frame 观察事件时的末尾赋值。不能把这次赋值搬进前面的通知分支，否则会改变旧系统的状态更新行为。

本次没有修改对外 API、键盘 frame 和可见性计算、动画参数、通知时机、observer 规则、CoreText 排版和测高算法，也没有替换 `YYLabel`、改变现有 `1.0.9` 的文字方向修复或新增自定义变量。

### 用户确认后追加的空尺寸保护

在 iOS 27 模拟器中，对有文本、尺寸为 `{0, 40}` 或 `{120, 0}` 的 `YYLabel` 强制执行同步 `layer display`，均在 `YYTextAsyncLayer.m` 的 `UIGraphicsBeginImageContextWithOptions` 处复现 `NSInternalInconsistencyException`。这是零尺寸条件下可复现的历史风险，不能据此称为 iOS 27 新引入的问题，也未证明三个使用方业务已经触发。

用户确认最小方案后，在同步绘制分支中、原 `willDisplay` 之后增加：

```objc
if (self.bounds.size.width < 1 || self.bounds.size.height < 1) {
    self.contents = nil;
    if (task.didDisplay) task.didDisplay(self, YES);
    return;
}
```

这使同步路径沿用既有异步路径的空尺寸规则：不创建 bitmap，不执行 display block，清空旧内容并报告本次处理完成。宽或高小于 1 的同步绘制行为因此发生了明确变化；正常尺寸的绘图 API、颜色格式、CoreText 绘制、回调顺序和异步实现保持原样。

## 旧调用的实际触发条件

管理器的对象文件被链接并加载后，`+load` 会延迟 0.1 秒调用 `defaultManager`。非 App Extension 中，管理器注册全局键盘 WillChangeFrame 通知，并在 iOS 9 及以后同时注册 DidChangeFrame 通知。有效 frame 通知会直接调用或调度 `_notifyAllObservers`，即使没有任何 observer，原实现仍会执行末尾的旧 API 读取。

`YYTextView` 创建时会主动注册 observer；`YYTextEffectWindow` 也使用管理器。`YYLabel`、`YYTextLayout` 本身没有直接引用管理器，所以不能仅凭使用了这两个类就断言管理器一定已加载。三个使用方的可见业务源码以 `YYLabel`、`YYTextLayout` 为主；实际 App 的链接方式和运行调用栈仍决定是否触发。其他库也可能调用相同的系统 API，不能据此保证整个 App 的警告消失。

## 核查后保留的既有逻辑

- iOS 7 专属分支中的旧方向 API：不在 iOS 27 执行，保留以维持旧系统行为；编译时仍可能报告 deprecated。
- `YYTextEffectWindow` 的全局单例、`app.windows/keyWindow`、主屏尺寸缓存，以及 `YYTextView` 的全局根控制器回退：属于既有 Scene/多屏上下文限制。当前没有证据表明它们是三个使用方的 iOS 27 功能阻断。完整适配需从 host view/window 确定 Scene，再处理 effect window 生命周期、尺寸、窗口层级与截图范围，不属于完全等价的局部替换；本次未实施此项行为变更。
- `UIView+YYText` 已从实际 source/destination view 取窗口并使用公开坐标转换，本次未发现必须改写的依据。
- `UIGraphicsBeginImageContextWithOptions`：废弃不等于失效。旧上下文固定为 8 bit/DeviceRGB，而 renderer 的默认格式可涉及不同色域和动态范围，因此本次未批量替换文字绘制实现。
- `YYTextEffectWindow._updateMagnifier:` 在全局 key window 为 nil 时可能向数组加入 nil。这是静态发现的条件性历史风险，未验证三个使用方会触发，本次未修改；与已复现并经用户确认修复的空尺寸绘图问题分开记录。

上述非等价 Scene 改造若后续需要实施，应先确认具体使用场景和最小方案。YYWebImage、YYImage、YYCache、SVProgressHUD 不属于本次修改范围。

## 验证

使用 Xcode 27.0（27A266a）、iPhoneOS/iPhoneSimulator 27.0 SDK。老 Framework 工程的 deployment target 是 iOS 8；仅在验证命令中覆盖为 iOS 15，未修改工程或 podspec 的最低支持版本。

- Framework Debug，generic iOS Simulator：编译通过。
- Framework Release，generic iOS Device，关闭签名：编译通过；这不是设备安装或真机运行验证。
- `git diff --check`：通过。
- iOS 27.0、iPhone 18 Pro 模拟器：最终 31 项针对性检查全部通过，详见下表。

运行验证使用一次性 Scene App，加载完整 YYText framework；通过替换旧方向 getter 的实现计数、发送合成键盘通知，以及测试代码调用内部方法检查分支语义。

| 范围 | 实际结果 |
| --- | --- |
| 自动注册 | 没有创建 YYTextView，也未添加 observer 时，`+load` 注册的通知仍会更新管理器状态；旧方向 getter 调用为 0 |
| 键盘转场 | Will/Did 通知的 from/to frame、显示/隐藏状态、时长、曲线、重复 frame 去重、移除 observer 后继续更新内部状态均符合预期 |
| 坐标转换 | 已知 window/view 的往返转换，以及 CGRectNull/CGRectInfinite 保留均通过；未覆盖多 Scene/外接屏 |
| 常规文本 | 中文、英文、阿拉伯文、emoji 多行测高、CoreText 绘制、YYLabel 同步/异步绘制均通过；未进行逐像素比对 |
| 空尺寸保护 | `{0,40}`、`{120,0}`、`{0.5,40}`、`{120,0.5}` 均不再抛异常；正常绘制后缩小会清空旧内容；每次 `willDisplay=1`、`display=0`、`didDisplay=1` 且 `finished=YES` |
| 旧分支语义 | 测试子类将 `_systemVersion` 覆盖为 7，验证通知旋转补丁及仅 frame 观察路径仍更新前一方向；这不等于 iOS 7 设备或运行时验证 |

这轮测试没有真实操作系统键盘，没有完成三个使用方的端到端或真机回归，也未覆盖 YYTextView 编辑、选区、放大镜和设备旋转。运行结果只证明上述测试条件成立。测试 App 已卸载，本次启动的模拟器已关闭；临时测试源码、构建输出和诊断文件在记录结果后清理，不向旧工程加入测试 target。

可复现编译命令（将临时输出目录换为自己的路径）：

```sh
xcodebuild -project Framework/YYText.xcodeproj -scheme YYText \
  -configuration Debug -sdk iphonesimulator \
  -destination 'generic/platform=iOS Simulator' \
  -derivedDataPath /tmp/yytext-check-simulator \
  IPHONEOS_DEPLOYMENT_TARGET=15.0 CODE_SIGNING_ALLOWED=NO build

xcodebuild -project Framework/YYText.xcodeproj -scheme YYText \
  -configuration Release -sdk iphoneos \
  -destination 'generic/platform=iOS' \
  -derivedDataPath /tmp/yytext-check-device \
  IPHONEOS_DEPLOYMENT_TARGET=15.0 CODE_SIGNING_ALLOWED=NO build
```

编译通过不代表没有告警。现有的 deprecated、旧函数原型等告警仍在；本次没有将它们全部归为 iOS 27 必改项。

## 三个使用方验收与后续统一引用

只读核实 RMS、ww-carcCuplet、electricity-IOS 的 Podfile 和 lock 均指向 `https://github.com/pzz819816/YYText.git` 的 tag `1.0.9`。本次未修改这些文件，也未执行使用方的 pod update。

验收时先在单独的验证副本中将 YYText 临时指向本地 fork，再做下列真机检查：

1. 打开原来使用 YYLabel/YYTextLayout 的协议文本、弹窗、套餐或列表页面，核对换行、高度、截断、富文本附件和复用滚动。
2. 在普通 UITextField/UITextView 页面反复显示、隐藏和切换键盘；对 `-[UIApplication statusBarOrientation]` 设符号断点，核对是否仍有 YYText 调用栈。
3. 若业务实际使用 YYTextView，补测编辑、选择、复制、拖动光标、放大镜、键盘避让，以及设备旋转；iPad 按需补测浮动/分离键盘与窗口变化。
4. 若实际使用多 Scene/外接屏，单独验证选区、放大镜和键盘坐标，不能用单窗口结果覆盖这些场景。

本次不推送、不发布 release、不移动 `1.0.9`。用户验收后，可将修复提交并推送，再选择未使用的新 tag（例如届时可用的 `1.0.10`，同步 podspec 版本）或精确 commit；三个项目统一引用同一版本/commit，分别更新 YYText 并提交各自 lock。继续锁定 `1.0.9` 不会自动取得本地修复。

## 参考

- [Apple：UIApplication.statusBarOrientation](https://developer.apple.com/documentation/uikit/uiapplication/statusbarorientation)
- [Apple：UIWindowScene.effectiveGeometry](https://developer.apple.com/documentation/uikit/uiwindowscene/effectivegeometry)
- [Apple：UIWindowScene.interfaceOrientation 的替代说明](https://developer.apple.com/documentation/uikit/uiwindowscene/interfaceorientation)

实际可用性同时核对了本机 iOS 27 SDK 的 `UIApplication.h`、`UIWindowScene.h`、`UIWindowSceneGeometry.h` 和绘图头文件；用户提供的 no-op 日志用于定位，未把交接报告当作真机验证结论。
