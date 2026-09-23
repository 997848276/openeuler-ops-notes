# DAY11-12 · iSula 容器引擎（对照 Docker 的国产线）

> openEuler 24.03 LTS / 128（oe-base）· 2026-09-23 下午
> 目标：装好欧拉自家的轻量容器引擎 iSulad，跑通「装 → 拉 → 跑 → 验」，产出实测版 **iSula vs Docker 对照表**。

## 任务速览

| 任务 | 结果 | 关键词 |
|---|---|---|
| A 组：安装与服务启动 | ✅ | runc 同款底座 / socket 组权限 / 加组重连 |
| B 组上半场：镜像获取 | ✅ | 短名不补前缀 / mirrors 不生效 / 绕道直连 |
| B 组下半场：容器运行 | ✅ | `-p` 接受不执行 / CNI 只管 CRI Pod / `--net=host` |
| 附加案：200 OK 假阳性 | ✅ 破案 | 宿主 nginx 抢答 / Server 版本验身份 |
| C 组：外部验证 | ✅ | Windows 浏览器 + Ctrl+F5 见官方页 |

## 1. 安装与服务启动

```bash
dnf list available iSulad           # 验源先行：update 源有 iSulad-2.1.5-18.oe2403
sudo dnf install -y iSulad          # 12 包 16M；依赖 = 架构图
sudo systemctl enable --now isulad  # 写操作必 sudo
sudo usermod -aG isula devops       # socket 组授权 + 退出重连 SSH
isula version                       # Client + Server 双段
systemctl status isulad             # Memory 9.0M / booted in 0.062s
```

**要点**：
- 依赖清单里 **runc 1.1.8 与 Docker 同款**（OCI 运行时同源），另有 lib-shim-v2（欧拉 shim，接 kata）——iSulad 是 **C 实现的轻量引擎**，差别在引擎外壳，不在集装箱标准。
- **Memory 9.0M + 0.062s 启动** = "轻量"从话术变实测数字（Docker daemon 通常 100MB+）。
- socket 权限 `srw-rw---- root:isula` 的源头是 daemon.json 里白纸黑字的 `"group": "isula"`——配置文件才是真相源。

**坑**：
1. `systemctl` 写操作忘 sudo → `Interactive authentication required`（polkit）。读免 sudo、写必 sudo。
2. `isula version` 报 `Can not connect` 不代表 daemon 死了——socket 组权限拒绝被掩成同一条文案。鉴别：`sudo isula version` 能通 = 权限问题。
3. `usermod -aG` 后**必须退出重连 SSH**：进程组凭据登录时定型。

## 2. 镜像获取：三道关

```bash
isula pull nginx                                      # ✗ Invalid image name, no host found
isula pull docker.io/library/nginx:latest             # 命名关过，21s connect timeout
sudo vi /etc/isulad/daemon.json                       # 配 registry-mirrors 三源
sudo python3 -m json.tool /etc/isulad/daemon.json     # 校验（逮住漏逗号 line 39）
# restart 后 pull 仍直连 registry-1.docker.io —— mirrors 没生效
curl -sI -m 5 https://docker.m.daocloud.io/v2/        # 401 = 源活着
sudo journalctl -u isulad -n 15                       # 全程零 daocloud 踪影
isula pull docker.m.daocloud.io/library/nginx:latest  # ★绕道直连，10 层全绿
```

**要点**：
- iSula **不自动补 `docker.io/library/` 前缀**（Docker 会补）——全限定名是硬要求。
- mirrors 配置语法正确、源活着、json 合法，但 pull 依旧直连官方源 → 定性 **iSulad 2.1.5 mirrors 特性 bug**（`"group"` 键实际生效反证配置文件在读，排除"没加载"）。
- **绕道法**：把镜像源当 registry 直连（`docker.m.daocloud.io/library/nginx:latest`）——通用逃生通道，且镜像直接到手。

## 3. 容器网络暗战（本日最硬）

```bash
isula run -d --name web -p 8080:80 docker.m.daocloud.io/library/nginx:latest
isula ps                # 显示 0.0.0.0:8080->80/tcp —— 但这是"配置显示"
curl -I 127.0.0.1:8080  # ✗ 0ms 拒连 = 没人听
isula inspect web | grep -iE 'ipaddress|gateway'   # 全空 = 容器没进过网络
# 排查修复三连（全部无效）：
#   ① dnf install containernetworking-plugins（二进制实际在 /usr/libexec/cni）
#   ② daemon.json 三键：network-plugin=cni + cni-bin-dir + cni-conf-dir + bridge conf
#   ③ systemd drop-in CLI 直传同样三参
# 终局（三方证据）：官方 CNI 文档通篇 Pod/sandbox 语境 + 社区"-p 不支持" →
isula run -d --name web --net=host docker.m.daocloud.io/library/nginx:latest   # ★正解
```

**结论**：
- **iSulad 2.1.5 的 CNI 三件套是 K8s/CRI 路线的门票，不是普通 `isula run` 容器的网络开关**——普通容器只有 lo 网卡。
- `-p` 被静默接受但不执行：`isula ps` 照样显示端口映射，实际零监听。正解 `--net=host`（容器用宿主网络栈，代价 = 无端口隔离）。

**坑**：
- daemon.json 里 `"cni-bin-dir": ""` 是**显式空串，会顶掉内置默认路径**——与"键不存在=用默认"完全两码事。
- 插件落点**先 `rpm -ql` 查再猜**：openEuler 在 `/usr/libexec/cni`，不是 Fedora 习惯的 `/opt/cni/bin`。
- vi 手编 JSON 漏逗号 → `python3 -m json.tool` 精确报行列；批量改键用 python json 读写。

## 4. 200 OK 假阳性案（验收纪律课）

```bash
isula ps -a        # web Exited (1)
isula logs web     # bind() to 0.0.0.0:80 failed (98: Address already in use) → still could not bind()
pgrep -a nginx     # 宿主 nginx master 1225（DAY6 遗留）一直占着 80
sudo ss -tlnp | grep ':80 '   # users:(("nginx",pid=1225...)) 凶手点名
sudo systemctl stop nginx && isula rm -f web && isula run -d --name web --net=host ...
curl -sI 127.0.0.1:80 | grep -i server   # Server: nginx/1.31.6 = 容器亲手答的 200
```

**教训**：第一次 `--net=host` 后 curl 也 200 OK——但 `Server: nginx/1.24.0` 是宿主源版本（容器是 1.31.6），**应答者是宿主 nginx 抢答**。`--net=host` 容器 bind 80 撞车直接 Exited(1)。

**200 OK ≠ 你的容器在应答。** 应答者身份三判据：
1. `Server` 头版本号比对；
2. `sudo ss -tlnp` 看进程名（不带 sudo 连进程列都没有）；
3. 应用日志（bind 98 重试三连）。

**彩蛋**：Windows 浏览器访问显示的还是宿主 openEuler 默认页——浏览器启发式缓存（无 Cache-Control + Last-Modified 4 个月前 → ~12 天新鲜度，直接吃本地缓存不发请求）。**Ctrl+F5 强刷**立刻见容器吐的官方页。

## 5. iSula vs Docker 对照表（全实测）

| 维度 | Docker（CE 26.1.3） | iSula（2.1.5） |
|---|---|---|
| 定位 | 全功能容器引擎 | 轻量引擎 + 原生 CRI（可当 K8s 运行时） |
| 实现语言/开销 | Go，daemon 100MB+ | C，daemon **9.0M**，启动 0.062s |
| OCI 运行时 | runc | runc 1.1.8（同款） |
| 镜像短名 | 自动补 docker.io/library/ | 不补，必须全限定名 |
| 镜像加速 | daemon.json mirrors 生效 | mirrors 配了不生效（2.1.5 bug）→ 绕道直连 |
| 普通容器网络 | 内置 bridge+iptables 开箱即用 | 只有 lo；CNI 三件套只服务 CRI Pod |
| 端口发布 | `-p` 真映射 | `-p` 接受不执行 → `--net=host` |
| socket 权限 | docker 组 | isula 组（同样 ≈root 等价，慎放） |
| 命令形态 | docker ps/images/run/... | isula 同形，学习成本低 |

一句话总结：**镜像与运行时是行业标准件，差异全在引擎外壳与网络模型**——选型要验证关键路径真的能跑，而不是看文档承诺。

## 面试一句话

> "我做过 iSulad 和 Docker 的同机对照：两者共享 OCI 镜像标准和 runc 运行时，iSulad 用 C 实现、daemon 只占 9M、原生 CRI 可当 K8s 运行时；但实测发现它 2.1.5 的镜像加速不生效、`-p` 静默不执行（CNI 只服务 CRI Pod），我用全限定名直连 + host 网络跑通了完整链路。而且验证时我不看 curl 200 就收工——用 Server 版本号、sudo ss 进程名、容器日志三层证据确认'到底是谁在应答'，期间还抓住宿主 nginx 抢答的假阳性。"
