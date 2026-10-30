# box64 SAVE_MEM + BAD_SIGNAL 编译仓库

为 RK3566 LoongOS 掌机（MiniLoong Pocket One）运行 Unity 游戏《王国：两位君主》，
交叉编译一个带 **`-DSAVE_MEM=ON`** + **`-DBAD_SIGNAL=ON`** 的 box64 arm64 二进制。

## 为什么需要这个版本

官方/第三方（ryanfortner/box64-debs）的预编译 `box64-android` 只带
`-DBAD_SIGNAL=ON`（修 RK 系列内核 si_code 不一致导致的 dynarec 崩溃），
**不带 `-DSAVE_MEM=ON`**。

`-DSAVE_MEM=ON` 的作用（box64 官方 COMPILE.md）：

- 用**五级跳表**替代四级跳表，减少未使用的地址空间分配
- **默认开启 `BOX64_DYNAREC_SEP=1`**（secondary entry points），减少重复 DynaRec 块

代价：跳块时多一次内存读，部分负载性能略降。对 956MB 内存、靠 swap 硬扛的
RK3566 掌机，省下来的几十~一两百 MB 比那点性能损失更值。

## 编译参数

```
cmake .. \
  -DBAD_SIGNAL=ON \
  -DSAVE_MEM=ON \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=aarch64-linux-gnu-gcc-11 \
  -DARM_DYNAREC=ON \
  -DBOX32=1 \
  -DBOX32_BINFMT=1
```

> 与 ryanfortner/box64-debs 的 `box64-android`（已验证能在 RK3566 上跑）相比，
> 唯一差异是多了一个 `-DSAVE_MEM=ON`。其余参数（BAD_SIGNAL / ARM_DYNAREC /
> BOX32 / RelWithDebInfo / gcc-11 交叉编译）完全一致，保证兼容性。

## 使用方法（3 步）

### 第 1 步：上传到 GitHub 并触发编译

**方式 A（推荐，最快）**：GitHub 网页「Import repository」
1. 打开 https://github.com/new/import
2. 把本目录整个打包上传，仓库名随意（如 `box64-savemem-build`）
3. 导入后到 **Actions** 标签页 → 点 `Build box64 (SAVE_MEM + BAD_SIGNAL)` →
   右侧 **Run workflow** → 绿色 **Run workflow** 按钮

**方式 B**：本地 git 推送
```sh
cd box64-savemem-build
git init && git add . && git commit -m "box64 SAVE_MEM + BAD_SIGNAL build"
git branch -M main
git remote add origin https://github.com/<你的用户名>/box64-savemem-build.git
git push -u origin main
# push 后 workflow 会自动触发（on: push）
```

### 第 2 步：下载产物

Actions 跑完后（约 5~10 分钟），点进那次 run → 底部 **Artifacts** →
下载 `box64-savemem-<commit>` → 解压得到 `box64-savemem-arm64.tar.gz` → 再解压得到
`box64-savemem`（这就是编译好的 arm64 二进制）。

### 第 3 步：替换掌机上的 box64

```sh
# 1) 传到掌机（走 SSH，几 MB 没问题，别走 WiFi 传大文件）
scp -P 2222 -i ~/.ssh/id_loong box64-savemem root@192.168.3.7:/tmp/box64-savemem

# 2) 备份旧版 + 替换（掌机上执行）
ssh -p 2222 -i ~/.ssh/id_loong root@192.168.3.7 '
  D=/mnt/sdcard/roms/ports/kingdom_two_crowns
  cp "$D/box64" "$D/box64.old-nosavemem"        # 备份当前 BAD_SIGNAL 版
  cp /tmp/box64-savemem "$D/box64"
  chmod +x "$D/box64"
  "$D/box64" --version                          # 应输出 "with Dynarec"
'
```

> 注意：`start.sh` 的 eMMC 加速逻辑会把游戏数据切到 `/userdata/kingdom`，
> 但 box64 始终用的是 `$GAMEDIR/box64`（TF 卡这份），所以替换 TF 卡这份即可。
> 若你的设备是「只更新 TF 卡」模式，不要动 `/userdata/kingdom`。

## 验证

```sh
ssh -p 2222 -i ~/.ssh/id_loong root@192.168.3.7 \
  "/mnt/sdcard/roms/ports/kingdom_two_crowns/box64 --version"
# 期望输出含 "with Dynarec"

# 实测内存收益：启动游戏后对比 swap 使用量
ssh -p 2222 -i ~/.ssh/id_loong root@192.168.3.7 "cat /proc/swaps"
# SAVE_MEM 版应比旧版 swap Used 更低（box64 自身 DynaRec 块占用下降）
```

## 常见误区（重要）

网上一堆 CSDN/博客会写 `export BOX64_SAVE_MEM=1` 或 `BOX64_DYNAREC_NOARCH=1`
这类**运行时环境变量**——**这些变量不存在，是编造的**。SAVE_MEM 是**编译期**
cmake 选项（`-DSAVE_MEM=ON`），只能重编译获得。这也是本仓库存在的原因。
