# Troubleshooting

按"哪一步报错"排序。

---

## Step 1: 装依赖失败

**症状**：`apt-get install` 某个包失败、或报 `E: Unable to locate package`。

**修复**：
```bash
sudo apt-get update
sudo apt-get install -f
./build.sh --deps-only
```

非 Ubuntu 系统请参考 [dependencies.md](./dependencies.md) 手工装。

---

## Step 2: clone 宿主树失败

**症状 A**：`Repository not found`

说明宿主树 URL 在你 clone 的那一刻无法访问。试试换源：

```bash
# 方案1: ImmortalWrt 官方 mt798x fork
HOST_TREE_URL=https://github.com/immortalwrt-mt798x/immortalwrt.git \
HOST_TREE_BRANCH=master \
./build.sh

# 方案2: ImmortalWrt 主线（官方，但 MT3600BE 支持可能滞后）
HOST_TREE_URL=https://github.com/immortalwrt/immortalwrt.git \
HOST_TREE_BRANCH=master \
./build.sh
```

**症状 B**：克隆很慢或超时

国内云服务器 + GitHub 的老问题。几个选择：
- 用 GitHub 加速器（比如 `ghproxy.com`）改 URL
- 用 Gitee 镜像（搜 "immortalwrt-mt798x gitee 镜像"）
- 增加 clone 深度让它一次拉完：`HOST_TREE_DEPTH=100 ./build.sh`

---

## Step 3: verify_device_support 失败

**症状**：
```
MT3600BE DTS not found: openwrt/target/linux/mediatek/dts/mt7987a-glinet-gl-mt3600be.dts
```

**说明**：你换了宿主树或 branch，新的树没有 MT3600BE 适配。

**修复**：
- 换回默认的 chasey-dev + 25.12 分支
- 或者去 `openwrt/target/linux/mediatek/dts/` 看有什么设备，改 `build.sh` 里的 `DEVICE_PROFILE` 测别的板子

---

## Step 4: feeds/defconfig 阶段出警告

**典型信息**：
```
WARNING: Makefile 'package/feeds/packages/xxx/Makefile' has a build depends on 'yyy', which does not exist
```

**一般无需处理**——这是 feeds 里某些包互相依赖的非致命警告。只要 `.config` 能正常生成、`CONFIG_TARGET_DEVICE_glinet_gl-mt3600be=y` 在里面，就可以继续。

**检查 .config 是不是正确选了设备**：
```bash
grep DEVICE_glinet_gl-mt3600be openwrt/.config
# 应该看到：CONFIG_TARGET_mediatek_filogic_DEVICE_glinet_gl-mt3600be=y
```

没看到的话 `./build.sh --menuconfig` 手动勾：`Target Profile → GL.iNet GL-MT3600BE`。

---

## Step 5: 编译失败

`build.sh` 会自动在失败时跑一次 `make -j1 V=s` 并输出最后 50 行到控制台。完整日志在 `logs/verbose-*.log`。

**最常见的错误类型**：

### 5.1 网络原因 download 失败

`download-*.log` 里有 `wget: unable to resolve host` 或 `HTTP 403`：
```bash
# 重试通常能解决
cd openwrt && make download -j4
cd .. && ./build.sh --resume
```

### 5.2 某个包编译失败

定位是哪个包：
```bash
grep -E "ERROR: |^make.*Error" logs/build-*.log | head -5
```

比如报 `package/feeds/packages/foo/Makefile: error`，就去 menuconfig 关掉它：
```bash
./build.sh --menuconfig
# 进 menuconfig 用 `/` 搜 foo，按空格反选
./build.sh --resume
```

### 5.3 内存不够（OOM）

大内存包（gcc/kernel）链接时可能吃 >4GB。16G 以下机器建议：
```bash
./build.sh --jobs 4   # 减少并行数
# 或加 swap
sudo dd if=/dev/zero of=/swapfile bs=1G count=4
sudo mkswap /swapfile && sudo swapon /swapfile
```

### 5.4 磁盘满

```bash
df -h .
# 清 ccache
ccache -C
# 清构建中间产物（但保留 dl/ 避免重下）
cd openwrt && make clean
```

---

## Step 6: 编出来了但刷不进

**症状**：`sysupgrade -T` 报 "Invalid image type" 或 "Magic mismatch"

**说明**：大概率是你从原厂固件直接刷 ImmortalWrt 时校验不过。

**修复**：
1. 先进 GL.iNet 原厂 U-Boot（断电，按住 reset 通电，`http://192.168.1.1`）
2. 通过 U-Boot Web UI 上传固件，不走 sysupgrade

**GL.iNet 官方 U-Boot 教程**：<https://docs.gl-inet.com/router/en/4/tutorials/uboot/>

---

## 通用技巧

**看完整编译日志的最后 200 行**：
```bash
tail -200 logs/build-*.log | less
```

**只重编某个 package**：
```bash
cd openwrt
make package/mt7990-firmware/{clean,compile} V=s
```

**两棵树对比**（如果想借鉴 ChuranNeko 的适配）：
```bash
git clone --depth=1 https://github.com/ChuranNeko/openwrt-snapshot-gl-mt3600be.git /tmp/ref
diff -ruN /tmp/ref/target/linux/mediatek/ openwrt/target/linux/mediatek/ | less
```

**追 MT3600BE 在宿主树里的历史**：
```bash
cd openwrt
git log --all --oneline -- target/linux/mediatek/dts/mt7987a-glinet-gl-mt3600be.dts
```
