# WeKit 修复悬浮标题栏

WeKit（dev.ujhhgtg.wekit）悬浮标题栏 / 悬浮输入框 bug 修复修改手册 + 最新修复版 APK。

## 修复内容（原版 → v3fix4）
- v1：从聊天列表进入时标题栏下沉一个状态栏高度
- v2：点进聊天窗口状态栏闪一下
- v3：滑入转场时状态栏变透明时机晚（闪帧）
- v3fix2：防撤回冲突（消息被撤回时聊天框卡死）
- v3fix4：长期使用卡顿（onPreDraw 每帧开销缓解）+ 修复 boolean 指令崩溃

## 下载（Releases）
最新修复版 APK 通过 GitHub Releases 分发（仓库内不保存 APK 二进制）：

👉 [**wekit-floating-titlebar-fix-v3fix4.apk**](https://github.com/aiyun0703/wekit-floating-titlebar-fix/releases/tag/v3fix4)（LSPosed 模块，直接替换安装）

历史版本/更新说明见 [Releases 页面](https://github.com/aiyun0703/wekit-floating-titlebar-fix/releases)。

## 文件说明
| 文件 | 说明 |
|---|---|
| [wekit-修改手册.md](wekit-修改手册.md) | 全部改动明细（smali 片段、工具链、踩坑记录、新版本应用流程） |

## 使用
1. 从 Releases 下载 APK → LSPosed 管理器 → 模块 → 替换安装 → 启用
2. 完全杀掉微信进程重新打开
3. 测试：来回切换聊天页（不闪不沉）、群里消息被撤回（不卡死）、长时间使用（滑动流畅）

> 反编译树不随仓库发布：原项目开源，修改手册中已含全部改动 smali 片段，配合源码可完整复现。
