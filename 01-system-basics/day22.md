# day-openeuler-0930b · 加练：dnf 本地离线源 + Shell 巡检脚本 + 国密 SM2/SM3/SM4

> 日期：2026-09-30（周三）上午 · 机器 128 / oe-base（openEuler 24.03）
> 性质：加练（非主线 DAY）。同日上午的 **LVM** 已在 `day-openeuler-0930.md`（推仓 day21）中；本文承接后三项。
> 推 git：**day22.md**（延后，推前 `gops log` 确认最新序号 day21=6a01353）

---

## 一、dnf 本地仓库：从零搭内网离线源

**场景**：内网无外网 → 下载包（含依赖）到本地 → 生成 repodata → 配成本地源（`file://` 自用 / `http://` 分发）。

### 步骤链
1. 建仓 + 拉包（含依赖）：`dnf download --resolve --destdir=/opt/localrepo/Packages httpd tree`（6 个包，1.6M）
2. 生成元数据：`dnf install -y createrepo_c` → `createrepo_c /opt/localrepo`（生成 `repodata/`，`.xml.zst` + 哈希前缀命名）
3. 配源：`/etc/yum.repos.d/local.repo`（`[local] baseurl=file:///opt/localrepo enabled=1 gpgcheck=0`）→ `dnf clean all; dnf repolist`
4. **隔离验证**：`dnf --disablerepo='*' --enablerepo='local' install -y httpd` → 五包 `Repository` 全 = `local` ✅
5. 发布 HTTP 源：SELinux 贴标 `semanage fcontext -a -t httpd_sys_content_t "/opt/localrepo(/.*)?"` + `restorecon -Rv` → `ln -s /opt/localrepo /usr/share/nginx/html/localrepo`（借 80 默认站）→ `curl` 得 200
6. HTTP 源验证：`reinstall -y tree`（强制重装才作数）→ 跨机 254 建 `repo128.repo` → `dnf list available` 出五包 ✅

### 关键坑
- `dnf download` 需 `dnf-plugins-core`；下载不加 `--resolve` → 依赖全缺。
- `file://` 三斜杠；HTTP 源 403 = `/opt` 没贴 `httpd_sys_content_t`（SELinux 标签，非文件权限）。
- **验证靶子要用未装包/`reinstall`**：已装包会 `Nothing to do` 骗过你。
- 彩蛋：`openEuler.repo.rpmnew` = RPM 保护本地配置（见 `.rpmnew` 说明你改过配置）。

### 面试一句话
内网装包靠 `dnf download --resolve` + `createrepo_c` + `baseurl` 本地源；HTTP 源发布卡的是 SELinux 标签 `httpd_sys_content_t`；验证用 `--disablerepo='*' --enablerepo=xxx` 隔离 + `reinstall` 真搬包。

---

## 二、Shell 巡检脚本 `inspect.sh`（v1→v5）

**目标**：一键出系统体检报告，能判阈值、能报警、能写日志、能定时、退出码能被监控消费。

### 演进
- v1 骨架：shebang + `$(命令替换)` + `chmod +x`
- v2 告警：`df|awk $5|tr -d '%'` 提数 + `if [ -gt ]` + ANSI 颜色 + `echo -e`
- v3 工程化：函数 `check_xx`/`local` + `log()` 用 `>>` 追加时间戳 + `exit $ALERT`（退出码=告警条数）
- v4 定时：`inspect.service`（`Type=oneshot`、`User=devops` → HOME=/home/devops 脚本不用改）+ `inspect.timer`（`OnCalendar=*:0/2`、`Persistent=true`）
- v5 故障演练：`sed` 降 `WARN` 制造告警；`/bin/false` 造必失败服务

### ★★ 自我吞噬陷阱（本日最值钱）
`exit 非0` 在 **cron/监控** 眼里=「发现问题」，在 **systemd** 眼里=「unit 坏掉」——同一退出码两种语义。脚本发现告警 `exit 1` → systemd 判 `inspect.service` 失败 → 下次巡检 `systemctl --failed` 把**脚本自己**数进失败 → 死循环。
- **修法**：① 脚本失败服务检查排除自身（`systemctl --failed | grep -v 'inspect.service'`）；② unit 用 wrapper `ExecStart=/bin/sh -c '/usr/local/bin/inspect.sh; exit 0'` 统一退出码（`SuccessExitStatus=0 1` 在该版本实测**未**白名单化）；或保留非 0 配 `OnFailure=` 做真告警通道。
- 本质：**把"巡检结论"和"单元健康"解耦**；告警信号留在 `inspect.log`。

### 面试一句话
Shell 巡检 = 采数 + awk 提数 + 阈值判断 + 颜色报警 + 函数封装 + 日志落盘 + 退出码；套 systemd timer 自动跑。坑：`echo -e`、`>>`、oneshot 的 `inactive(dead)` 是正常态，以及**退出码语义冲突导致自我吞噬**，靠 wrapper 解耦。

---

## 三、国密 SM2/SM3/SM4（openssl 3.0.12 软件层）

| 算法 | 类型 | 对标 | 命令要点 |
|---|---|---|---|
| **SM3** | 哈希 256 位 | SHA-256 | `echo -n "abc" \| openssl dgst -sm3` |
| **SM4** | 对称 128 位 | AES | `openssl enc -sm4-cbc -salt -pbkdf2 -pass pass:…` |
| **SM2** | 非对称 | ECC/RSA | `genpkey -algorithm SM2` + `dgst -sm3 -sign/-verify` + `pkeyutl` |

### 实测要点
- **SM3("abc")** = `66c7f0f4…8f4ba8e0`（命中金标准向量）；`echo -n` 的 `-n` 是命门（多算换行摘要全变）。
- **SM4**：密文头 `Salted__`（8+8 字节头）；解密必须带同样的 `-pbkdf2`，否则 `bad decrypt`；密钥 16 字节 = 32 hex。
- **SM2 签名**：`Verified OK` / 篡改 `Verification failure`（因果闭环）；签名是 DER 的 (r,s) 两大整数。
- **SM2 加密**：C1C3C2 格式；明文长度有限 → 生产做混合加密（SM2 传 SM4 密钥）。
- **SM2 自签国密证书**：`req -new -x509 -key sm2.key -sm3 -days 365 -subj "/C=CN/…"` → `Signature Algorithm: SM2-with-SM3`；`openssl verify -CAfile sm2.crt sm2.crt` = OK。
- **OID**：`1.2.156.10197.1.401`(SM3)/`…104.x`(SM4)/`…301`(SM2)；**`1.2.156` = 中国 OID 前缀**。

### 选型认知
- **签名 = 私钥签、公钥验**（证明"谁发的"）；**加密 = 公钥加、私钥解**（保护"谁能看"）——**方向相反**。
- `dgst -sm3 -sign` = 先 SM3 摘要再 SM2 签名；`-sm3` 也决定证书签名摘要（漏了变非纯国密）。
- 普通 nginx 不支持 SM2 TLS 套件，国密 HTTPS 需专门 nginx（GmSSL/Tongsuo）。

### 面试一句话
国密用 openssl 落地：SM3 摘要、SM4 对称、SM2 签名+公钥加密；签名私钥签公钥验/加密公钥加私钥解方向相反，生产做混合加密 + 国密双证书；认 OID 前缀 `1.2.156`。

---

## 四、今日坑位速查（合并）

| 现象 | 正解 |
|---|---|
| `dnf download` 不存在 | 装 `dnf-plugins-core` |
| HTTP 源 curl 403 | `/opt` 贴 `httpd_sys_content_t` 标签 |
| `install` 返回 `Nothing to do` | 靶子已装 → 换 `reinstall` |
| 脚本颜色不显示 | `echo` 改 `echo -e` |
| 巡检脚本自己进 failed | `exit 非0` 被 systemd 误读 → 排除自身 + wrapper |
| SM4 解密 `bad decrypt` | 解密未带同样的 `-pbkdf2` / 口令错 |
| SM3 摘要对不上向量 | 少写 `-n` |
| 验签 `Verification failure` | 原文被改——正确行为 |

## 五、总面试一句话
一个上午打通**信创运维加练包**：LVM（存储池/在线扩容/快照）→ dnf 本地离线源（内网装包）→ Shell 巡检脚本（自动化/退出码陷阱）→ 国密 SM2/3/4（信创加密 + 国密证书）；**共同的硬核是"SELinux 标签、退出码语义、输入一分不差"**。
