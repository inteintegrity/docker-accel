# docker-accel

一条命令给设备配好容器镜像加速——三条路径，换设备不用重新踩坑。

设备刚刷好系统时，Docker 守护进程既没有代理也没有镜像站，`docker pull` 只能
走直连，从 `ghcr.io` 拉取就非常慢。这个脚本会测出当前网络下哪条路真的快、
把它配上，并且能一键还原。

| 路径 | 原理 | 实测速度（reComputer 在国内局域网） | 依赖 |
| --- | --- | --- | --- |
| `direct` | 直连 | 30–70 KB/s | — |
| `proxy` | Docker 守护进程走 HTTP 代理（`/etc/systemd/system/docker.service.d/http-proxy.conf`） | ~544 KB/s | 代理主机要在同一局域网（如开了 allow-lan 的 Clash） |
| `mirror` | 把镜像地址重写到镜像站（`ghcr.nju.edu.cn`）拉取后再 retag 回原名 | ~260 KB/s | 无 |

以上是实际拉取 Seeed GHCR 镜像时测到的数字；脚本会在设备上重新测量，所以你会
看到自己网络下的结果。

## 安装

把脚本拷到设备并加上可执行权限：

```bash
scp docker-accel <user>@<开发板>:/tmp/
ssh <user>@<开发板> 'sudo install -m 0755 /tmp/docker-accel /usr/local/bin/docker-accel'
```

或者从仓库安装：

```bash
git clone https://github.com/inteintegrity/docker-accel.git
sudo install -m 0755 docker-accel/docker-accel /usr/local/bin/docker-accel
```

只依赖 `bash` 和 `curl`（Raspberry Pi OS 自带）。

## 用法

```bash
# 1. 先看当前网络下哪条路快（给了代理地址就一并测代理）
docker-accel probe 192.168.3.114:6789

# 2a. 最快：让 Docker 守护进程走局域网代理
sudo docker-accel proxy 192.168.3.114:6789
docker pull ghcr.io/seeed-projects/recomputer-hailo10h-cv/yolov8_pose:latest

# 2b. 或者完全不依赖代理：走镜像站
docker-accel mirror ghcr.nju.edu.cn
docker-accel pull ghcr.io/seeed-projects/recomputer-hailo10h-cv/yolov8_pose:latest
```

`docker-accel pull` 会按最近一次探测结果自动选路，选中的路径失败时自动回退到
直连；无论走哪条路，最后都会把镜像 retag 回原始名字，脚本和 compose 文件不用
改镜像名前缀。

其他命令：

```bash
docker-accel status        # 当前代理/镜像配置、上次探测、上次拉取
docker-accel proxy off     # 移除守护进程代理
docker-accel off           # 还原到修改前的 docker 配置
docker-accel -n proxy ...  # --dry-run：只打印将要执行的动作
```

## 安全性

- 所有会被修改的文件，改动前都备份到 `/etc/docker-accel/backup/<时间戳>/`。
- 重启 docker 失败或 `docker info` 校验不通过时，会自动还原上一份配置。
- `--dry-run` 在任何平台（包括笔记本）都能跑，只打印动作、不执行。
- 配置只写在设备本地，不会向外发送任何数据。

## 代理侧检查清单（运行 Clash 的 Windows/macOS 主机）

走代理路径需要主机允许局域网连接：

1. 打开 `allow_lan`（Clash for Windows 勾选 *Allow LAN*；mihomo API 方式：
   `curl -X PATCH http://127.0.0.1:9097/configs -H "Authorization: Bearer <secret>" -d '{"allow-lan": true}'`）。
2. 放行代理端口：
   `New-NetFirewallRule -DisplayName "Allow LAN proxy 6789" -Direction Inbound -Protocol TCP -LocalPort 6789 -Action Allow -Profile Any`。
3. 确认主机 IP 没变：`docker-accel probe <主机IP>:6789` 应报告代理路径可达。

## 说明

- `/etc/docker/daemon.json` 里的 `registry-mirrors` **只对 Docker Hub（`docker.io`）生效**，
  不能加速 `ghcr.io`。所以镜像路径做成 pull 包装器，而不是守护进程配置。
- 镜像路径只加速通过 `docker-accel pull` 发起的拉取；直接执行
  `docker pull ghcr.io/...` 仍然走直连（除非配了代理）。
- 测速参考文件是 Hailo Model Zoo S3 上固定的一段 4 MiB 数据，它衡量的是线路
  快慢而非镜像站本身；端到端的真实速度请看 `pull` 输出里那行计时。