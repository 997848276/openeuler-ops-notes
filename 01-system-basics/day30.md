# day30 · systemd 单元编写实战：从救 unit 到生 unit（2026-10-09 · 128 oe-base · systemd 255）

## 一 侦察：三层目录与 unit 生态

```bash
systemctl --version | head -2             # systemd 255 (255.58-1.oe2403) +SELINUX +PAM +AUDIT
ls /usr/lib/systemd/system | wc -l        # 347  ← vendor 层（软件包自带）
ls -A /run/systemd/system | wc -l         # 0    ← 运行层 tmpfs（瞬态 unit 才点亮）
ls -A /etc/systemd/system/                # 管理层：少量 + 一堆 *.wants/
ls -l /etc/systemd/system/multi-user.target.wants/
# xxx.service -> /usr/lib/systemd/system/xxx.service   ← enable 的铁证：软链
# remote-cryptsetup.target -> ...（红=断链，unit 文件已不存在）
```

- 三层优先级 /usr/lib < /run < /etc：底座 347 个，顶层说了算。
- ★enabled ≠ 有效：断链软链还在、目标没了，systemd 容忍空链（快捷方式还在、软件已卸载）。

## 二 模板解剖：nginx 的 unit

```bash
sudo systemctl cat nginx | head -40
# Type=forking + PIDFile=/run/nginx.pid   ← fork 型主进程要 PIDFile 报身份证
# ExecStartPre=... nginx -t               ← 起跑前体检，配置错不上场
# ExecReload=/bin/kill -s HUP $MAINPID / PrivateTmp=true
# [Install] WantedBy=multi-user.target    ← enable 的挂载点地图
```

- Type 选型：simple（前台常驻最常用）/ forking（要 PIDFile）/ oneshot（跑完就退）/ notify（主动报活）。

## 三 主菜：手写 hello-lab.service

```bash
sudo mkdir -p /opt/hello-lab
sudo tee /opt/hello-lab/counter.sh >/dev/null <<'EOF'   # 每 5s 心跳写日志
#!/bin/bash
count=0
while true; do
    count=$((count+1))
    echo "$(date '+%F %T') heartbeat #$count" >> /opt/hello-lab/heartbeat.log
    sleep 5
done
EOF
sudo chmod +x /opt/hello-lab/counter.sh

sudo tee /etc/systemd/system/hello-lab.service >/dev/null <<'EOF'
[Unit]
Description=Hello Lab - my first self-written service
After=network.target
[Service]
Type=simple
ExecStart=/bin/bash /opt/hello-lab/counter.sh
Restart=on-failure
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
systemctl list-unit-files | grep hello-lab   # hello-lab.service  disabled  disabled
sudo systemctl start hello-lab
systemctl status hello-lab --no-pager -l
# Active: active (running) / Main PID: 2773 (bash)
# CGroup: /system.slice/hello-lab.service → 2773 bash → 2783 sleep 5
sudo cat /opt/hello-lab/heartbeat.log        # 每 5s 一条精准如钟
```

- ★坑：写文件用 `sudo tee`，`sudo cat > file` 的重定向会被自己的 shell 吃掉（root 没参与）。
- ★enabled（自启）与 active（在跑）是正交两轴：status 里 disabled + active (running) 同屏。
- 改 unit 必须 daemon-reload，否则 systemd 用内存里的旧版本。

## 四 enable/disable：软链的两端

```bash
sudo systemctl enable hello-lab
# Created symlink .../multi-user.target.wants/hello-lab.service → /etc/systemd/system/hello-lab.service.
sudo systemctl disable --now hello-lab
# Removed "/etc/systemd/system/multi-user.target.wants/hello-lab.service".
```

- §三十三"enable=建软链"拿到亲手回执；--now = 自启开关+当前开关一键双切。

## 五 drop-in 覆盖：不动原文贴协议

```bash
sudo mkdir -p /etc/systemd/system/hello-lab.service.d
sudo tee /etc/systemd/system/hello-lab.service.d/override.conf >/dev/null <<'EOF'
[Service]
RestartSec=1
EOF
sudo systemctl daemon-reload
sudo systemctl cat hello-lab    # 两段式：原 unit + override.conf
systemctl status hello-lab      # Drop-In: ... └─override.conf
```

- drop-in=不改原合同贴补充协议：同名键后者生效，升级包覆盖 unit 也不丢修改。
- ⚠️ show -p RestartSec 空回显留疑，但——

## 六 自愈测试：SIGKILL 斩首与替补上场

```bash
sudo systemctl kill --signal=SIGKILL --kill-who=main hello-lab
sleep 4; systemctl status hello-lab    # Main PID 2773 → 3179，active (running)
sudo journalctl -u hello-lab -n 8 --no-pager
# 08:20:15 Main process exited, code=killed, status=9/KILL
# 08:20:15 Failed with result 'signal'.
# 08:20:16 Scheduled restart job, restart counter is at 1.   ← 1 秒档！
# 08:20:16 Started Hello Lab ...
```

- ★drop-in 生效的铁证在时间差：kill(08:20:15)→restart(08:20:16)=1s（原 RestartSec=3 应 3 秒档）——回显可以骗人，时间戳不会。
- Restart 选型：on-failure（只管非正常死）/ always（连正常退出也拉起）。

## 七 状态落盘 vs 进程内存：接缝两枚 #1

```bash
sudo grep -n "heartbeat #1$" /opt/hello-lab/heartbeat.log
# 1:  08:16:36 heartbeat #1   ← 第一次人生
# 45: 08:20:16 heartbeat #1   ← 第二次人生（第一生 44 拍被斩断）
```

- ★假阴性反转：tail 见 #18~#21 似"没归零"，算术揭穿：08:20:16+17×5s=08:21:41=#18 时间戳（没归零应是 #62）——归零真发生，是 tail 看的"最近"不是"接缝"。
- ★坑：验证窗口选错=假阴性；找"重新开始"去接缝 grep，别 tail。
- 设计启示：进程内存状态重启即清零，服务状态必须落盘。

## 八 坑位速查

| 坑 | 正解 |
|---|---|
| sudo cat > file 写 unit 失败 | 重定向被自己 shell 吃；用 sudo tee / sudo sh -c |
| 改 unit 没反应 | daemon-reload |
| start rc=0 但没起来 | Type=simple 竞态，status 验活 |
| fork 型 MAIN PID 不对 | PIDFile 必配 |
| 怕升级覆盖 unit 修改 | drop-in override.conf |
| drop-in 生效没 | systemctl cat 合并视图 + status Drop-In 行 + 时间差铁证 |
| 重启后状态丢了 | 状态落盘，别揣进程内存 |
| 判断"有没有重新开始" | grep 接缝找 #1，tail 只能看最近 |

## 面试一句话

> "systemd 服务生命周期我全程手动走过：unit 三层目录 /etc 管理层优先级最高，写完 daemon-reload；enable 本质是往 wants/ 建软链（disable 拆链，完全对称）；改配置用 .service.d/override.conf drop-in 贴补充协议，升级覆盖也不丢。两个实测细节：enabled 和 active 是正交两轴，disabled 照样 active (running)，排障分开看；Restart=on-failure 我用 SIGKILL 斩首实测自愈，journal 从 'Failed with result signal' 到 'Scheduled restart job' 一秒拉起。方法论：验证重启后状态有没有重置，tail 最近几条会假阴性——计数明明归零了，要 grep 日志接缝找两枚第一拍。看的位置不对，显示再清楚也白看。"
