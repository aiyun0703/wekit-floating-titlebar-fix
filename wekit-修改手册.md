# WeKit 悬浮标题栏/悬浮输入框 修改手册

> 适用范围：dev.ujhhgtg.wekit（Zygisk/LSPosed 模块）悬浮标题栏 + 悬浮输入框
> 覆盖版本：原版 app-release.apk → v1 → v2 → v3 → v3fix2 → v3fix4（最终）
> 用途：作者更新新版本后，按本手册**重新应用全部修改**，修复：状态栏下沉、进入闪一下、转场闪帧、防撤回冲突卡死、长期卡顿

---

## 0. 版本结果速览

| 版本 | 修复内容 | 状态 |
|---|---|---|
| v1 | 从聊天列表进入时标题栏下沉一个状态栏高度 | ✅ 已合入 |
| v2 | 点进聊天窗口状态栏闪一下 | ✅ 已合入 |
| v3 | 滑入转场时状态栏变透明时机晚（闪帧） | ✅ 已合入 |
| v3fix | 防撤回冲突（错误基线版，弃用） | ❌ 类型过滤导致闪回归 |
| v3fix2 | 防闪 + 防撤回兼得（正确基线=用户 v3 (1)） | ✅ 已合入 |
| v3fix3 | 防卡顿缓解（statusInited 误用 sget/sput 崩溃，弃用） | ❌ VerifyError |
| **v3fix4** | 防卡顿缓解（修复 boolean 指令） | ✅ **最终版** |

**最终基线**：用户上传的 `wekit-floating-titlebar-fix-v3 (1).apk`（含状态栏防闪，代码与工作目录 v3 一致）。
**最终产物**：`wekit-floating-titlebar-fix-v3fix4.apk`（= v3fix2 + 防卡顿两项优化，全部修复合入）。

---

## 1. 工具链（完整命令）

```bash
export PATH="/home/user/Doubao/chats/720540752001026/build_env/jdk-11.0.32.1+1/bin:$PATH"
cd /home/user/Doubao/chats/720540752001026/wekit_work

# ① 反编译（apktool 必须用 2.11.1 的 apktool_new.jar，2.10.0 对 invoke-polymorphic 报错）
java -jar apktool_new.jar d -r -f <输入.apk> -o <输出目录>

# ② 编译
java -jar apktool_new.jar b <工作树> -o <out>-unsigned.apk

# ③ 对齐 + 签名
build_env/android-14/zipalign -f 4 <out>-unsigned.apk <out>-aligned.apk
build_env/android-14/apksigner sign --ks release.keystore --ks-key-alias wekit \
  --ks-pass pass:wekit2026 --key-pass pass:wekit2026 \
  --out <最终.apk> <out>-aligned.apk
build_env/android-14/apksigner verify <最终.apk>

# ④ 验证修改确实进包（jadx 从最终 APK 反编译核对）
wekit_work/jadx/bin/jadx -d <验证目录> --no-res --no-debug-info -j 4 <最终.apk>
```

**关键资源路径**：
- keystore：`wekit_work/release.keystore`（alias `wekit`，密码 `wekit2026`）
- apktool 2.11.1：`wekit_work/apktool_new.jar`
- jadx：`wekit_work/jadx/bin/jadx`

---

## 2. 类与功能地图（类名可能随版本变化，用功能定位）

| 类 | 作用 | 定位方法 |
|---|---|---|
| `zt3` | 悬浮标题栏主类（FloatingChatHeader） | 搜功能 ID 常量表 |
| `qo` | onPreDraw 每帧回调（核心修改点） | 实现 `ViewTreeObserver$OnPreDrawListener` |
| `jt3` | hook 分发，case 9 = setStatusBarStrategy | 搜 `ALWAYS_HIDE` |
| `gt3` | hook 注册表（case 10=ChattingUILayout 构造等） | — |
| `ft3` | 悬浮输入框 ChatFooter 样式 | — |
| `ct3` | 输入框 onPreDraw 回调 | — |

**功能 ID 常量（p64 表）**：
- 悬浮标题栏 = `2ff3c096d16c3f4ee12f51729b081665`
- 悬浮输入框 = `7ef3c6f7c943ff68aa730099157b4f42`

**新版本定位策略**：类名/方法名可能被混淆改名。用**特征字符串**搜索：
- `jc6.B`、`setStatusBarColor`、`ALWAYS_HIDE`、`EdgeToEdgeWrapperLayout`、`getRootWindowInsets`、`getLocationOnScreen`、`onPreDraw`

---

## 3. 修改总览（全部只改 3 个文件，零新增方法）

| # | 文件 | 位置 | 修改 | 目的 |
|---|---|---|---|---|
| 1 | `zt3.smali` | `r0()` | 删除 `jc6.B` 的 WeakHashMap 缓存判断，每次强制沉浸 | v1 修下沉 |
| 2 | `zt3.smali` | `R()` | 类名相等判断 `m5b.o(...)` → `getName().contains(...)` | v1 修下沉 |
| 3 | `jt3.smali` | case 9 | 删除缓存命中判断，无条件强制 ALWAYS_HIDE | v2 修进入闪 |
| 4 | `qo.smali` | `onPreDraw` 开头 | 插入父链遍历 padding 兜底 + return false | v3 修转场闪帧 |
| 5 | `qo.smali` | 同上 | 同值缓存跳过（edgeCache WeakHashMap） | v3fix2 修防撤回卡死 |
| 6 | `qo.smali` | 同上 | 每 30 帧重置 edgeCache（frameCounter） | v3fix2 自动恢复沉浸 |
| 7 | `qo.smali` | cond_4 段 | 状态栏高度 30 帧节流缓存 | v3fix4 防卡顿 |
| 8 | `qo.smali` | goto_4 段 | int[2] 数组复用（tmpArr） | v3fix4 防卡顿 |

**硬约束**（见第 5 节踩坑）：**所有修改只能加字段、只能内联进现有方法体，禁止新增任何方法**（method_ids 已满 65536）；**寄存器只用 v0-v15**。

---

## 4. 逐处修改详解（含 smali 片段）

### 4.1 【v1】zt3.r0()：强制沉浸（修下沉）

**位置**：`zt3.smali` 的 `r0(Landroid/view/View;)V` 方法内，找特征 `invoke-static {v0, v3}, Ljc6;->B(Landroid/view/Window;Z)V`（约 8925 行）。

**原逻辑（bug）**：
```smali
# 受 y:WeakHashMap 缓存保护——微信复用窗口后缓存命中，不再强制沉浸
sget-object v4, Lzt3;->y:Ljava/util/WeakHashMap;
invoke-virtual {v4, v0}, ... get(...)
# 若缓存已有 → 跳过 jc6.B 强制沉浸
```

**修改后**：**删除缓存判断，每次无条件强制**：
```smali
invoke-static {v0, v3}, Ljc6;->B(Landroid/view/Window;Z)V   # 每次强制沉浸
invoke-virtual {v0, v3}, Landroid/view/Window;->setStatusBarColor(I)V  # 同时置透明
```

**注意**：同一文件 21774 行附近还有一处 `jc6.B` + `setStatusBarColor(0)`，是 r0 的强制沉浸入口，保留。

### 4.2 【v1】zt3.R()：类名 contains 匹配（修下沉）

**位置**：`zt3.smali` 的 `R(Landroid/view/View;)Z` 方法内，特征 `"EdgeToEdgeWrapperLayout"`（约 4216 行）。

**原逻辑（bug）**：`m5b.o(view, "com.tencent.mm.ui.widget.EdgeToEdgeWrapperLayout")` 用**严格类名相等**匹配，微信子类匹配不到 → 返回 false，不修 offset。

**修改后**：
```smali
const-string v2, "EdgeToEdgeWrapperLayout"
invoke-virtual {v1, v2}, Ljava/lang/String;->contains(Ljava/lang/CharSequence;)Z   # 子类也匹配
move-result v1
if-eqz v1, :cond_0
```

### 4.3 【v2】jt3 case 9：无条件 ALWAYS_HIDE（修进入闪一下）

**位置**：`jt3.smali` 的 pswitch case 9，特征 `"ALWAYS_HIDE"`（约 2097 行）。

**原逻辑（bug）**：仅当某缓存命中才强制 `setStatusBarStrategy(ALWAYS_HIDE)`，否则放行微信默认策略 → 进入聊天页时状态栏短暂非沉浸（闪一下）。

**修改后**：**删除外层缓存判断，无条件强制**。case 9 的完整修改形态：
```smali
:pswitch_8   # case 9 入口
invoke-interface {p1}, ...getArgs()[Ljava/lang/Object;
...
const-string v1, "ALWAYS_HIDE"
invoke-static {v0, v1}, Lg55;->n(Ljava/lang/Object;Ljava/lang/Object;)Z   # equals TRUE 检查
move-result v0
if-eqz v0, :cond_1d          # 非 ALWAYS_HIDE 才走 else
...强制路径（原缓存判断删除后无条件执行）...
```

### 4.4 【v3】qo.onPreDraw()：父链 padding 兜底（修滑入转场闪帧）

**位置**：`qo.smali` 的 `onPreDraw()Z` 方法开头，`iget-object p0, p0, Lqo;->p:Landroid/view/View;` 之后。

**背景**：转场（窗口滑入）期间微信给父容器设 paddingTop，标题栏/状态栏在滑动快结束时才变透明 → 闪帧。

**插入代码（原版此处只有 `const/4 v6, 0x0` 后直接进入后面的逻辑）**：
```smali
iget-object p0, p0, Lqo;->p:Landroid/view/View;
invoke-virtual {p0}, Landroid/view/View;->getParent()Landroid/view/ViewParent;
move-result-object v4
const/4 v6, 0x0
:goto_0
if-eqz v4, :cond_1
instance-of v5, v4, Landroid/view/View;
if-eqz v5, :cond_1
check-cast v4, Landroid/view/View;
invoke-virtual {v4}, Landroid/view/View;->getPaddingTop()I
move-result v5
if-eqz v5, :cond_0
# （v3fix2 注入：同值缓存，见 4.5）
...setPadding 清理 + v6=1...
:cond_0
invoke-virtual {v4}, Landroid/view/View;->getParent()Landroid/view/ViewParent;
move-result-object v4
goto :goto_0
:cond_1
if-eqz v6, :cond_2
invoke-virtual {p0}, Landroid/view/View;->requestLayout()V
const/4 v1, 0x0
return v1
:cond_2
```

### 4.5 【v3fix2】qo.onPreDraw()：同值缓存（修防撤回冲突卡死）

**背景**：防撤回模块插入「xxx撤回了一条消息」item 后触发微信反复重设祖先 paddingTop → 每帧清理+取消帧+重排 → 无限 layout 循环卡死；滚动到该消息也卡。

**位置**：4.4 的 `if-eqz v5, :cond_0` 之后、`getPaddingLeft` 之前插入：

```smali
# === 防撤回冲突修复：同值缓存跳过（保留无差别清理以维持状态栏防闪） ===
sget-object v3, Lqo;->edgeCache:Ljava/util/WeakHashMap;
if-nez v3, :cond_einit
new-instance v3, Ljava/util/WeakHashMap;
invoke-direct {v3}, Ljava/util/WeakHashMap;-><init>()V
sput-object v3, Lqo;->edgeCache:Ljava/util/WeakHashMap;
:cond_einit
invoke-virtual {v3, v4}, Ljava/util/WeakHashMap;->get(Ljava/lang/Object;)Ljava/lang/Object;
move-result-object v3
if-eqz v3, :cond_eclean
check-cast v3, Ljava/lang/Integer;
invoke-virtual {v3}, Ljava/lang/Integer;->intValue()I
move-result v3
if-eq v3, v5, :cond_0          # 同值 → 跳过（不清理、不重排）
:cond_eclean
move v10, v5
invoke-virtual {v4}, Landroid/view/View;->getPaddingLeft()I
move-result v5
const/4 v9, 0x0
invoke-virtual {v4}, Landroid/view/View;->getPaddingRight()I
move-result v7
invoke-virtual {v4}, Landroid/view/View;->getPaddingBottom()I
move-result v8
invoke-virtual {v4, v5, v9, v7, v8}, Landroid/view/View;->setPadding(IIII)V
sget-object v3, Lqo;->edgeCache:Ljava/util/WeakHashMap;
invoke-static {v10}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;
move-result-object v5
invoke-virtual {v3, v4, v5}, Ljava/util/WeakHashMap;->put(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
const/4 v6, 0x1
:cond_0
```

**新增字段**：`.field public static edgeCache:Ljava/util/WeakHashMap;`

**重要经验**：**不要加类型过滤（只清 EdgeToEdge）**——那会让非 EdgeToEdge 容器漏清，转场闪回归（v3fix 的教训）。

### 4.6 【v3fix2】qo.onPreDraw()：30 帧重置缓存（自动恢复沉浸）

**位置**：`getParent` 赋值 v4、`const/4 v6, 0x0` 之后、`:goto_0` 之前插入：

```smali
# === 每 30 帧重置同值缓存：防撤回 item 结束后自动恢复沉浸 ===
sget v10, Lqo;->frameCounter:I
add-int/lit8 v10, v10, 0x1
sput v10, Lqo;->frameCounter:I
const/16 v10, 0x1e
sget v9, Lqo;->frameCounter:I
if-lt v9, v10, :cond_fc_skip
sget-object v9, Lqo;->edgeCache:Ljava/util/WeakHashMap;
if-eqz v9, :cond_fc_skip
invoke-virtual {v9}, Ljava/util/WeakHashMap;->clear()V
const/4 v9, 0x0
sput v9, Lqo;->frameCounter:I
:cond_fc_skip
:goto_0
```

**新增字段**：`.field public static frameCounter:I`

**作用**：防撤回 item 动画结束、微信停止重设 padding 后，缓存自动失效，下一次遇到非零 padding 会再清一次 → 恢复沉浸；动画期间最多每 30 帧重排一次，不卡死。

### 4.7 【v3fix4】qo.onPreDraw()：状态栏高度 30 帧节流（防卡顿）

**位置**：`cond_2` 之后的 `:cond_4` 段（特征 `getRootWindowInsets`）。替换原来每帧的 insets 查询：

```smali
:cond_4
# === 防卡顿：状态栏高度 30 帧节流缓存 ===
sget-boolean v10, Lqo;->statusInited:Z
if-nez v10, :cond_sb_check
# 未初始化 -> 刷新
:cond_sb_refresh
invoke-virtual {p0}, Landroid/view/View;->getRootWindowInsets()Landroid/view/WindowInsets;
move-result-object v3
if-eqz v3, :cond_3
invoke-static {}, Lzl9;->b()I
move-result v4
invoke-static {v3, v4}, Lh4;->x(Landroid/view/WindowInsets;I)Landroid/graphics/Insets;
move-result-object v3
if-eqz v3, :cond_3
invoke-static {v3}, Lo63;->s(Landroid/graphics/Insets;)I
move-result v3
sput v3, Lqo;->statusTop:I
sget v10, Lqo;->frameCounter:I
sput v10, Lqo;->lastRefresh:I
const/4 v10, 0x1
sput-boolean v10, Lqo;->statusInited:Z
goto :goto_1
:cond_sb_check
sget v4, Lqo;->frameCounter:I
sget v10, Lqo;->lastRefresh:I
sub-int v4, v4, v10
const/16 v10, 0x1e
if-ge v4, v10, :cond_sb_refresh
sget v3, Lqo;->statusTop:I
:goto_1
```

**新增字段**：
```smali
.field public static statusTop:I
.field public static lastRefresh:I
.field public static statusInited:Z
```

**⚠️ 关键**：`statusInited` 是 **boolean**，必须用 **`sget-boolean` / `sput-boolean`**！用 `sget`/`sput` 会 `VerifyError: expected field boolean ... but found 'Z'`（v3fix3 崩溃教训）。

### 4.8 【v3fix4】qo.onPreDraw()：int[2] 数组复用（防卡顿）

**位置**：`getLocationOnScreen` 前原 `new-array v6, v6, [I` 处替换：

```smali
# === 防卡顿：int[2] 数组复用，避免每帧分配 ===
sget-object v6, Lqo;->tmpArr:[I
if-nez v6, :cond_arr
const/4 v6, 0x2
new-array v6, v6, [I
sput-object v6, Lqo;->tmpArr:[I
:cond_arr
invoke-virtual {v5, v6}, Landroid/view/View;->getLocationOnScreen([I)V
```

**新增字段**：`.field public static tmpArr:[I`

**注意**：qo.onPreDraw 是同步方法，静态数组复用无并发风险。onPreDraw 内另一处 pswitch_0 分支的 `getLocationOnScreen` 用的是 `Lro;->f:[I` / `Lro;->e:[I`（已有复用数组），**不要动**。

### 4.9 qo.smali 最终新增字段汇总

```smali
# static fields
.field public static edgeCache:Ljava/util/WeakHashMap;
.field public static frameCounter:I
.field public static tmpArr:[I
.field public static statusTop:I
.field public static lastRefresh:I
.field public static statusInited:Z
```

`.locals 11` 保持（v0-v10 足够，全部新增逻辑只用 v0-v15）。

---

## 5. 踩坑记录（务必遵守）

1. **dex method_ids 已满 65536**：任何新增方法（含静态方法）编译报 `Unsigned short value out of range: 65537`。**只能加字段、只能内联进现有方法体**。
2. **apktool 汇编器寄存器限制**：普通指令只接受 v0-v15；v16+ 必须用 `/16` 变体。注入代码保持现有 `.locals`，用 v0-v15 内复用。
3. **boolean 字段必须用 `sget-boolean`/`sput-boolean`**：`sget`/`sput` 是 int 专用，boolean 会 ART VerifyError（运行时崩溃，v3fix3 教训）。
4. **apktool 必须用 2.11.1（apktool_new.jar）**：2.10.0 对 `invoke-polymorphic` 报错。
5. **`un3.smali`（4 处）、`z55.smali`（2 处）含 invoke-polymorphic**（Kotlin 反射 MethodHandle）：反编译后必须改写为 `invokeWithArguments` 等义指令才能回编译——**永远不要"修复"或改回**。
6. **防撤回修复不要用类型过滤**（只清 EdgeToEdge）：会让非 EdgeToEdge 容器漏清，转场闪回归（v3fix 教训）。正确做法 = 无差别清理 + 同值缓存。
7. **已验证失败的方案（勿再试）**：
   - v4：hook `Window.setStatusBarColor` 前置变 0 → 用户判坏，废弃
   - 液态玻璃 v5/v6/v7（模糊/半透明注入）→ 用户否决，废弃
8. **WorkTree 注意事项**：`wekit_work/liquid_glass_work/v5_smali/` 是液态玻璃残留树（含 v5/v6/v7 全部注入），**严禁作为基线**。正确基线树 = `wekit_work/v3fix_work/build2/`（v3fix4 工作树）。

---

## 6. 新版本应用流程（step by step）

作者更新新版本后：

1. **反编译新 APK**：`java -jar apktool_new.jar d -r -f <新版本.apk> -o <新树>`
2. **定位 3 个目标类**：用特征字符串搜索
   - `grep -rn "ALWAYS_HIDE" <新树>/smali/` → jt3 case 9
   - `grep -rn "EdgeToEdgeWrapperLayout" <新树>/smali/` → zt3.R
   - `grep -rln "OnPreDrawListener" <新树>/smali/` 中 p 字段为 View 的 → qo（对照 zt3 里 `new-instance` 注册的类）
   - `grep -rn "jc6;->B" <新树>/smali/` → zt3.r0 强制沉浸点
3. **逐个应用 4.1-4.8 的修改**（按 4.x 小节，每处都要在**新代码上下文**里重新定位，不能直接搬行号）
4. **编译 + 签名 + verify**（第 1 节命令）
5. **jadx 从最终 APK 反编译验证**：确认 qo 的 6 个静态字段 + zt3 的 contains + jt3 的 ALWAYS_HIDE 无条件都在包里
6. **交付用户实测**：来回切换聊天页（不闪不沉）+ 群里触发消息撤回（不卡死）+ 长时间使用（滑动流畅）

**改动只涉及 3 个文件**：`zt3.smali`（2 处）、`jt3.smali`（1 处）、`qo.smali`（4 处 + 6 字段）。如果新版本这三个类被重构/改名，按功能 ID（悬浮标题栏 `2ff3c096d16c3f4ee12f51729b081665`）和特征字符串重新定位。

---

## 7. 验证清单

- [ ] apktool 编译无报错
- [ ] apksigner verify 通过
- [ ] jadx 反编译最终 APK：qo 字段齐全（edgeCache/frameCounter/tmpArr/statusTop/lastRefresh/statusInited）
- [ ] zt3.R 是 contains 匹配（非 equals）
- [ ] jt3 case 9 无条件 ALWAYS_HIDE
- [ ] 用户实测：进聊天不沉不闪、撤回不卡、长时间不卡

---

## 8. 版本产物对照

| 文件 | 说明 |
|---|---|
| `app-release.apk`（附件） | 原版，反编译源 |
| `wekit-floating-titlebar-fix.apk` | v1 |
| `wekit-floating-titlebar-fix-v2.apk` | v2 |
| `wekit-floating-titlebar-fix-v3.apk` | v3（15:02，与用户 v3 (1) 代码一致） |
| `wekit-floating-titlebar-fix-v3 (1).apk`（用户附件） | **正确基线**（用户实测含防闪） |
| `wekit-floating-titlebar-fix-v3fix.apk` | 错误基线防撤回版（弃用） |
| `wekit-floating-titlebar-fix-v3fix2.apk` | 防闪+防撤回 ✓ |
| `wekit-floating-titlebar-fix-v3fix3.apk` | 防卡顿（崩溃，弃用） |
| `wekit-floating-titlebar-fix-v3fix4.apk` | **最终版（全部修复）** |
| `wekit_work/v3fix_work/build2/` | v3fix4 工作树（含全部修改的 smali） |
| `wekit_work/v3fix_work/v3base/` | v3 反编译树（对照基线） |
