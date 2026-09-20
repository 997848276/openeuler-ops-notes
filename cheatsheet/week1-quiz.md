# openEuler 第 1 周自测题与答案（DAY1~DAY6 复盘）

> 答对 6/8；错题集中在"归因判断"（症状 → 成因），弱项已定位。
> 用法：面试前扫一遍题干，先心里默答，再看答案。

---

## 题目

**题 1**：`nmcli device reapply ens33` 和 `nmcli connection up ens33` 有什么区别？为什么改网卡配置推荐用前者？

**题 2**：openEuler 上网卡的配置文件存在哪？手改完必须执行哪两条命令？为什么老教程里的 `systemctl restart network` 不行？

**题 3**：`sudo firewall-cmd --add-port=8080/tcp`（不加 `--permanent`）之后，服务器重启会发生什么？

**题 4**：客户端 curl 报 `No route to host`，**1 毫秒**就失败，说明发生了什么？为什么"1 毫秒"这个细节很关键？

**题 5**：`lvextend -r -L +5G` 报空间不够（就差 1 个 extent），最正确的处理是？

**题 6**：systemd 里 `status=203/EXEC` 是什么意思？

**题 7**：进程退出码 `137` 和 `143` 分别代表什么？

**题 8**：nginx 报 `bind() to 0.0.0.0:9001 failed (13: Permission denied)`，但 `nginx -t` 显示 syntax is ok —— 根本原因是什么？

---

## 答案与解析

**题 1** ✅
`reapply` = 把配置**推给内核但不重启连接**（热生效，SSH 不断线）；`con up` = **重启整条连接**（会短暂断）。
> 类比：`reapply` 是**行驶中换轮胎**，`con up` 是**靠边停车重新出发**。

**题 2** ✅
`/etc/sysconfig/network-scripts/ifcfg-ens33`（openEuler 的 NetworkManager 走 **ifcfg-rh 插件**，不是 keyfile）。
改完必须：`nmcli connection reload`（重读文件）+ `nmcli connection up ens33`（重新激活）—— **只 reload 不 up = 白改**。
`systemctl restart network` 失效是因为 openEuler 24.03 已经没有 `network.service`，已由 NetworkManager 取代。

**题 3** ❌ → 正解：**规则丢失，8080 又不通**（不是"端口还在等手动 up"）
- `--add-port` 不带 `--permanent` = **只写运行时内存**，**立刻生效但不持久**
- `reload` / 重启 → 丢弃内存，从**永久账本**重建 → 8080 消失，**而且不会自己回来**
- ⚠️ 概念串线点：**firewalld 没有"up"这个动作**。"改完要 up"是 nmcli 的逻辑（生效问题）；firewalld 的轴是**持久问题**：

| | nmcli | firewalld |
|---|---|---|
| 改完立刻生效？ | ❌ 要 reapply/up | ✅ 立刻生效 |
| 会持久化？ | ✅ 写 ifcfg | ❌ 看有没有 `--permanent` |

**题 4** ✅
**毫秒级失败 = 被 REJECT（拒绝并回话）**，具体是 firewalld public 区默认的 `icmp-host-prohibited`（ICMP type 3 code 10）。
如果是 `DROP`，ICMP 也不回，客户端只能**傻等超时**（`Connection timed out`）——**沉默是 DROP 的特征，回话是 REJECT 的特征**。

**题 5** ✅
**先 `vgextend` 扩池**（往 VG 里加新盘），再 `lvextend`。
"差 1 个 extent"是**余粮卡在 VG 边界**，不是 `-L` 写法错 —— **判据在余粮，不在写法**。

**题 6** ✅
**systemd 执行 `ExecStart` 失败（脚本压根没跑）**。
常见原因：**SELinux 标签不对**（如 `mv` 把 `admin_home_t` 带到了 `/usr/local/bin`）或没有执行权限。
修法：`restorecon -v <脚本>` → `ls -Z` 复核 → `systemctl daemon-reload` 重试。
> 对照：`209/STDOUT` = systemd 写不了 stdout 日志路径；`1` = 脚本自己跑挂。**`2xx` = systemd 没铺好摊子，`1` = 脚本自己挂。**

**题 7** ✅
- `137` = 128 + **9 = SIGKILL**（不可捕获；OOM 被杀典型）
- `143` = 128 + **15 = SIGTERM**（可捕获；`pkill` 默认发的就是 15）
> 口诀：**退出码减 128 = 凶手信号号**。

**题 8** ❌ → 正解：**SELinux 端口类型白名单拒绝**（不是配置文件权限）
两个反证：
1. **看动作**：报错动作是 `bind()`（绑定端口系统调用），对象是 `IP:端口` —— **跟读文件无关**。文件权限问题会报 `open() ... Permission denied`。
2. **`nginx -t` 已排除**：能读到配置、能解析语法 ⇒ **配置文件的权限和内容都没问题**。
剩下唯一的环节就是 worker 真正去 `bind()` 的那一刻 → **SELinux**（9001 属于 `tor_port_t`，而 nginx 在 `httpd_t` 域，无权绑）。
确认手段：`ausearch -m avc -ts recent`（AVC 审计记录）或 `setenforce 0` 对照实验（**验完必须 `setenforce 1` 开回来**）。

---

## 复盘结论

| 强项 | 弱项 |
|---|---|
| **操作流程**记得牢：reapply、vgextend、203、137/143、nmcli 落盘位置 全对 | **归因判断**：看到 `Permission denied` → 想到文件权限；看到"要生效" → 套 nmcli 的 up |

**改进动作**：把「症状 → 成因」表贴在显示器边（见 `openEuler速查表-第1周.md` 第一节），每次排障先对表、再动手。
