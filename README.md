# mt3600be-immortalwrt

> **一键在云服务器上为 GL-MT3600BE (Beryl 7) 编译 ImmortalWrt 固件。**

基于 [chasey-dev/immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase) 官方树，MT3600BE 设备适配由 ImmortalWrt 核心维护者 [@Tianling Shen](https://github.com/immortalwrt/immortalwrt) 直接提供。本仓库只负责把编译流程打包成一条命令。

## 快速开始

**云服务器要求**：Ubuntu 22.04/24.04 LTS，x86_64，≥4C8G，≥50GB 磁盘。

```bash
git clone https://github.com/kyya/mt3600be-immortalwrt.git
cd mt3600be-immortalwrt

tmux new -s build    # 强烈建议在 tmux 里跑，防 SSH 断线

./build.sh 2>&1 | tee logs/first-run.log
```

首次全量编译约 1–2 小时。成功后固件在：
```
openwrt/bin/targets/mediatek/filogic/*mt3600be*sysupgrade.bin
```

## 常用子命令

```bash
./build.sh                 # 完整流水线
./build.sh --deps-only     # 只装构建依赖
./build.sh --config        # 停在 feeds + defconfig 之后
./build.sh --menuconfig    # 合完 config 后开 menuconfig 自定义
./build.sh --resume        # 改完代码后继续编（跳过 clone）
./build.sh --clean         # 清空 ./openwrt/ 从头来
./build.sh --jobs 4        # 手动指定 -j 数量
```

## 可选的环境变量

```bash
# 换一个宿主树（比如想试 ImmortalWrt 主线）
HOST_TREE_URL=https://github.com/immortalwrt-mt798x/immortalwrt.git ./build.sh

# 换 branch
HOST_TREE_BRANCH=main ./build.sh

# 增加 clone 深度（想看完整历史）
HOST_TREE_DEPTH=100 ./build.sh

# 调整 ccache 大小
CCACHE_SIZE=40G ./build.sh
```

## 目录结构

```
.
├── build.sh                 # 一键构建入口
├── .github/workflows/
│   └── build.yml            # GitHub Actions 云端编译（推 tag 触发）
├── docs/
│   └── dependencies.md      # 各发行版依赖清单
├── logs/                    # 构建日志（运行后生成，已 gitignore）
└── openwrt/                 # 宿主树（运行后生成，已 gitignore）
```

## 刷机

```bash
# 以 GL.iNet 默认管理地址为例（OpenWrt 默认是 192.168.1.1）
FIRMWARE=openwrt/bin/targets/mediatek/filogic/openwrt-mediatek-filogic-glinet_gl-mt3600be-squashfs-sysupgrade.bin

scp $FIRMWARE root@192.168.8.1:/tmp/
ssh root@192.168.8.1 "sysupgrade -T /tmp/$(basename $FIRMWARE)"   # 校验
ssh root@192.168.8.1 "sysupgrade -n /tmp/$(basename $FIRMWARE)"   # 刷（-n = 不保留配置）
```

**如果刷进去启动不了**：断电，按住 reset 通电进 GL.iNet 原厂 U-Boot (`http://192.168.1.1`)，上传原厂固件恢复即可。**不会真变砖**——只要你不刷 U-Boot 本身。

## License

This project is licensed under **GPL-2.0-only**, matching the upstream OpenWrt / ImmortalWrt ecosystem. See [`LICENSE`](./LICENSE).

SPDX short identifier: `GPL-2.0-only`

## Credits

- [@Tianling Shen](https://github.com/cnsztl) (cnsztl@immortalwrt.org) — MT3600BE 官方适配的作者
- [ImmortalWrt](https://immortalwrt.org) / [OpenWrt](https://openwrt.org) — 基础框架
- [chasey-dev](https://github.com/chasey-dev) — mt798x-rebase 宿主树维护
- [@ChuranNeko](https://github.com/ChuranNeko) — 独立 OpenWrt snapshot fork（本仓库未用，但是很好的参考实现）
