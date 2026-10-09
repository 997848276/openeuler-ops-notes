# day29 · SELinux 排障：从 avc denied 到放行（2026-10-08 · 128 oe-base · openEuler 24.03 targeted/Enforcing）

## 一 侦察：战场档案 + 一笔旧账

```bash
sudo getenforce                       # Enforcing
sudo sestatus | head -8               # loaded policy: targeted / config file: enforcing
sudo semanage port -l | grep -E 'ssh_port_t|http_port_t'
# ssh_port_t   tcp  2222, 22          ← 2222 在册（当年加固配过）
# http_port_t  tcp  80, 81, 443, 488, 8008, 8009, 8443, 9000   ← 没有 8080！
sudo semanage port -l -C              # 只显本地修改 → ssh_port_t tcp 2222（旧账结案）
sudo ausearch -m avc -ts today        # 开机自带活体 avc
```

- ssh 换端口在 Enforcing 下必须注册：`semanage port -a -t ssh_port_t -p tcp 2222`。
- 类比：SELinux 端口类型=小区门禁授权表，开新门牌先去物业登记。

## 二 8080 之谜：httpd_t 的钥匙串不止一串

```bash
ps -eZ | grep nginx                   # 三进程全在 system_u:system_r:httpd_t:s0
sudo semanage port -l | grep 8080     # http_cache_port_t  tcp 8080, 8118, 8123, 10001-10010
```

- http_port_t 没有 8080，nginx 却合法监听 → **8080 在 http_cache_port_t（Squid 缓存族）**，httpd_t 对缓存端口有绑定许可。
- 一个端口可属不同类型；httpd_t 的钥匙串挂着好几串。

## 三 主菜：亲手制造一条 avc（独立 nginx 实例，零碰主服务）

```bash
# 独立配置 /etc/nginx/lab8888.conf（listen 8888 + 独立 pid/log）+ 临时 unit nginx-lab.service
sudo systemctl start nginx-lab; echo "rc=$?"   # rc=0 ⚠️ 但——
systemctl status nginx-lab                      # Active: failed (Duration: 8ms)
sudo journalctl -u nginx-lab | tail -5
# nginx: [emerg] bind() to 0.0.0.0:8888 failed (13: Permission denied)
sudo ausearch -m avc -ts recent | grep 8888
# avc: denied { name_bind } for comm="nginx" src=8888 scontext=httpd_t tcontext=unreserved_port_t
sudo ausearch -m avc -ts recent --raw | audit2why   # 建议 setsebool -P nis_enabled 1 ←误导药方！
sudo semanage port -a -t http_port_t -p tcp 8888
sudo systemctl start nginx-lab && curl http://127.0.0.1:8888/    # "lab ok"
# 清场：stop + semanage port -d + rm 配置/unit + daemon-reload + grep 验收 rc=1
```

- ★`systemctl start` rc=0 ≠ 服务活着：Type=simple 启动竞态（bind 失败瞬间退出，systemd 没来得及通知）；验活 status/journalctl 或 start --wait。
- ★audit2why 是翻译官不是医生：nis_enabled 建议纯误导；端口问题正解=semanage port -a。
- 三层证据链：语法层（nginx -t 过）→ 网络层（端口空闲）→ SELinux 层（name_bind denied）。

## 四 活体 avc 定性：不是每扇门都该配钥匙

- rngd dac_override / agetty checkpoint_restore：系统组件噪音（服务功能正常=忽略）。
- 见 avc 就 audit2allow = 拿安全墙换安静；先看功能坏没坏。

## 五 文件上下文：cp vs mv + sesearch 结案

```bash
sudo cp /tmp/cp.html /usr/share/nginx/html/lab/    # 继承 httpd_sys_content_t
sudo mv /tmp/mv.html /usr/share/nginx/html/lab/    # 保留 user_tmp_t（mv 不换 label！）
sudo ls -Z /usr/share/nginx/html/lab/
curl http://127.0.0.1/lab/cp.html    # 200
curl http://127.0.0.1/lab/mv.html    # 200 ⚠️ 老 RHEL 是 403——openEuler 24.03 策略宽了
getsebool httpd_read_user_content    # off（布尔排除）
sudo dnf install -y setools-console && sesearch --allow -s httpd_t -t user_tmp_t -c file
# allow httpd_t user_tmp_t:file { append getattr ioctl lock map read write };  ← 无条件 allow！
# allow httpd_t user_home_type:file {...}; [ httpd_read_user_content ]:True    ← 对照：布尔条件式
sudo restorecon -v /usr/share/nginx/html/lab/mv.html
# Relabeled ... from user_tmp_t to httpd_sys_content_t
```

- ★label 跟 inode 走：cp 重新生成（继承新目录），mv 原样搬运（保留旧 label）；restorecon 按路径规则打回。
- ★版本差异：openEuler 24.03 无条件放行 httpd_t→user_tmp_t（教科书 403 不触发）；httpd_can_network_connect 默认 on（RHEL off）。
- 方法论：经典坑要在当前版本重验——本机回显才是事实。

## 六 坑位速查

| 坑 | 正解 |
|---|---|
| 777 还是 Permission denied | ausearch -m avc 看 tcontext |
| 服务换端口起不来 | semanage port -a -t <类型> -p tcp <端口> |
| start rc=0 但没起来 | Type=simple 竞态，status 验活 |
| audit2why 建议可疑 | 翻译官不是医生，按本质选 semanage/restorecon |
| mv 进 web 目录 403 | restorecon -v；老 RHEL 会 403，openEuler 24.03 不触发 |
| 查策略原文 | sesearch --allow（setools-console） |
| nginx web root | openEuler 是 /usr/share/nginx/html |

## 面试一句话

> "SELinux 排障三板斧：ausearch -m avc 抓记录、audit2why 翻译、按本质处理——端口问题 semanage port 注册（我实测 SSH 开 2222 不注册就是 bind Permission denied）、文件 label 污染 restorecon 打回。两个细节：systemctl start 的 rc=0 不代表服务活着（Type=simple 启动竞态，要 status 验活）；audit2why 的建议要过脑子（它给过我开 nis_enabled 的误导药方，真问题是端口类型没注册）。还有版本意识：mv 进 web 目录带 tmp label 在老 RHEL 必 403，但 openEuler 24.03 我用 sesearch 查过策略原文是无条件放行——经典坑也要在当前版本重验。"
