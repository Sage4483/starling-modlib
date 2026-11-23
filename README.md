# starling-modlib

Starling（Minecraft Java 版手机启动器/客户端 0.9.0）的**模组打包构建仓库**。
这里不放业务代码，只放两样东西：**补丁 kit**（`kit*.zip`）和**GitHub Actions 工作流**。
推一次代码 / 手动点一次 Run，CI 会把上游 0.9.0 源码拉下来、打上补丁、编译出 arm64-v8a 的 APK，挂在 Artifacts 里。

> 仅个人自用与学习交流。上游 Starling 的代码与素材版权归原作者，请遵守其许可；本仓库的补丁仅为适配手机端所做的移植改动。

## 产物一览

| 工作流 | 触发 | 产出 | 说明 |
|---|---|---|---|
| `build-090-mods.yml` | 手动 `workflow_dispatch` | `apk-mods-090` | **全模组包**：11 个模组一起编进内核，日常主用 |
| `build-090-schematic.yml` | 手动 | 单投影包 | 只带 `schematic` 投影模组，用来单独验证投影改动 |
| `build-090-stock.yml` | 手动 | `Starling Dev` | 上游原版 0.9.0，不打任何补丁，包名 `…starling.dev`，可与上面两个共存，用来做对照 |
| `build-mod.yml` | 手动 | 单模组库 | 只编 `modlib`（模组宿主），轻量验证 |
| `build-090-mods-push.yml` | **push 自动** | 同全模组包 | 往本仓库上传任何文件都会自动跑一次，约 25–27 分钟，注意别误触 |

一次构建大概 25–30 分钟（要现编 137 MB 的 mod host，CI 机器内存不够，工作流里用 `fallocate` 挂了 10 GB swap 才编得动），所以别指望几分钟出包。

## 目录

```
.github/workflows/          # 上面那 5 个工作流
mods/                       # 11 个 .hnmod，按需合进包里
  schematic.hnmod           #   投影（自建 Rust 实现，仿 Litematica）
  minihud.hnmod             #   HUD 信息
  waymap.hnmod              #   路径点/地图
  minimap-zh.hnmod          #   小地图（汉化）
  jei-zh.hnmod              #   物品配方查询（汉化）
  appleskin-zh.hnmod        #   饱食度显示（汉化）
  lookat-zh.hnmod           #   看向方块信息（汉化）
  shulker_peek-zh.hnmod     #   潜影盒预览（汉化）
  freecam_plus.hnmod        #   自由视角
  gamma.hnmod               #   亮度解锁
  player_bot.hnmod          #   假人
kit.zip / kit090.zip        # 早期补丁包
kit090full.zip              # 当前全量 kit（全模组包用，构建脚本读这个）
kit090schem.zip             # 投影单包用
kitnew2.zip                 # 迭代中的备用 kit
```

`kit090full.zip` 里是构建真正需要的东西：

- `full090.patch` —— 手机端适配总补丁（触屏、UI、渲染、启动参数等）
- `schematic090.patch` —— 投影模组补丁（`schematics_core.rs` / `schematic_litematic.rs` 等）
- `CJK-Regular.ttf` / `CJK-Bold.ttf` / `CJK-Medium.ttf` —— 中文字体，塞进 `horizon-theme/fonts/`
- `starling.keystore` —— 签名用

## 自己跑一遍

1. Fork 本仓库（**建议设成 Private**：kit 里带签名 keystore，公开等于把签名私钥挂出来）。
2. Actions 页面选 `build-090-mods`，点 `Run workflow`。
3. 跑完在该次 run 页面底部下载 `apk-mods-090`，解压得到 `starling-0.9.0-arm64-v8a.apk`，直接装。

构建逻辑（`build-090-mods*.yml` 里的 `prepare` 步骤）：

```bash
git apply full090.patch      # 先打总补丁
git apply schematic090.patch # 再打投影补丁
# 然后把 mods/ 下的 .hnmod 一起打进去，共 11 个
```

补丁顺序不能反，`schematic090.patch` 是基于打完 `full090.patch` 的树生成的。

## 安装注意

- 只出 **arm64-v8a**，老 32 位机器装不上。
- 签名用的固定 keystore，所以覆盖安装不会掉存档；但和官方版签名不同，**不能直接覆盖官方 Starling**，要先卸载官方版（存档在游戏目录里，卸载前自己备份）。
- 装完首次启动会解包资源，慢一点是正常的。

## 常见问题

- **构建卡在 `build` 步骤半小时**：正常，mod host 编译就是慢；看日志里 `Finished release` 出现才算快好了。
- **`No space left on device` / 被 OOM kill**：工作流已经挂了 swap，如果还挂，把 `CARGO_BUILD_JOBS` 再调小。
- **一上传文件就自动开始构建**：`build-090-mods-push.yml` 是 `on: [push]`，改 README 也算 push。只想改文档不想构建，就把这个工作流的触发改成只留 `workflow_dispatch`。
- **APK 装上了但模组没生效**：确认装的是 `apk-mods-090` 那个包（11 模组），单投影包只有 `schematic`。
