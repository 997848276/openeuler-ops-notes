# day20 · Ansible 进阶（handlers + template + roles + vault）

> 本篇 = 仓库 day20。9/29 实操，对应手册 `Ansible命令解读与类比.md` §十~§十五。
> 续 day19：在 /root/ansible-lab 基础上把"自动化"从入门推到工程化四件套。

## 一、handlers + template（动态生成配置 + 改了才重载）

端口连环坑（开课先撞）：
- `ss -tlnp | grep 8080` → 被 **docker-proxy** 占（Docker 端口映射代理进程，双栈 4 处）；8081 也被占 → 挑端口先 `ss` 探测。
- ★ SELinux 端口白名单联动（DAY6）：nginx 只能 bind `http_port_t` 默认成员 80/81/443/488/**8008**/8009/8443/9000 → 选 8008 零 semanage 配置；随手挑 18080 会在 Enforcing 机器 bind 被拒。
- ★ `mkdir -p template`（少 s）→ `cat > templates/...` 报 No such file：重定向不建目录。

模板 + playbook 全案（Jinja2 变量 {{ inventory_hostname }}）：
```bash
mkdir -p templates
cat > templates/lab_index.html.j2 <<'EOF'
<h1>Managed by Ansible - {{ inventory_hostname }}</h1>
<p>host: {{ inventory_hostname }} / os: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
EOF
cat > templates/lab8008.conf.j2 <<'EOF'
server {
    listen 8008;
    server_name {{ inventory_hostname }};
    root /usr/share/nginx/html/lab;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
EOF
```
playbook 关键：`notify: reload nginx` 触发 `handlers:` 里的 `service: nginx state=reloaded`。

实测三跑曲线（RECAP）：
| 跑次 | 双机 | RUNNING HANDLER |
|---|---|---|
| 首跑 | ok=5 changed=4 | 出现（reload nginx changed） |
| 复跑 | ok=4 changed=0 | 整段消失 |
| 再跑 | ok=4 changed=0 | 无 |
验证：`ansible all -m shell -a 'curl -s 127.0.0.1:8008/'` → localhost 与 oe128 各吐带**自己主机名**的首页（同一模板两份输出 = 变量替换铁证）。
> handler = "改了才通知的工单"：比对出差异才派单，无差异连工单都不开。

## 二、roles：把 playbook 升级成标准件

```bash
mkdir -p roles/lab-nginx/{tasks,handlers,templates}
cp templates/*.j2 roles/lab-nginx/templates/
# tasks/main.yml 三任务照搬（role 内 template src 只写文件名，自动找本 role 的 templates/）
# handlers/main.yml: reload nginx
# site.yml 入口: hosts: all / become: yes / roles: [lab-nginx]
ansible-playbook site.yml
```
回显：任务名带前缀 **`TASK [lab-nginx : 确保 lab 目录存在]`**（role 生效视觉铁证）；RECAP 双机 ok=4 changed=0、无 handler = **重构等价性机器证明**（目标状态没变 → 零变更，等同重构后测试全绿）。

## 三、vault：敏感变量 AES256 加密

```bash
cat > secret.yml <<'EOF'      # 先写明文（不安全现状）
vault_db_password: SuperSecret@2026
vault_api_key: sk-lab-1234567890abcd
EOF
ansible-vault encrypt secret.yml      # Encryption successful（原地覆盖）
cat secret.yml                         # $ANSIBLE_VAULT;1.1;AES256 + 乱码（明文已死）
ansible-vault view secret.yml          # 输密码 → 吐明文
# playbook: vars_files: [secret.yml] + debug 只打指纹：length / [:8]
ansible-playbook use_secret.yml --ask-vault-pass
# msg: "DB密码长度=16位, API key前8位=sk-lab-1"  ok=1 changed=0
```
★ 坑（vault 不可逆课）：①忘密码=文件废（无后门，生产用 vault_password_file 600）；②encrypt 原地覆盖，加密前确认无明文散落副本；③debug 打敏感值只打指纹（日志会外流）；④忘带 --ask-vault-pass → Decryption failed（钥匙没插，非文件坏）。

## 坑清单（进阶篇）
9. `cat > 路径/文件` No such file → 重定向不建目录，先 ls 父目录
10. 宿主端口 8080/8081 被 docker-proxy 占 → 挑端口先 ss 探测
11. SELinux 机器 nginx 监听非常规端口 → 只能 bind http_port_t 白名单，否则 semanage port -a
12. vault 忘密码/忘带 --ask-vault-pass → 前者文件废，后者补参数即可

## 面试一句话（进阶版）
> "进阶我完整练过工程化四件套：template 用 Jinja2 给双机生成各自主机名虚拟主机——同一模板两份输出；handlers 做配置变更才触发 reload，三跑曲线 changed 4→0→0，零变更时 handler 整段不出现；roles 把一锅炖拆成 tasks/handlers/templates 目录契约，重构后 RECAP 零 changed 等价性机器验证；vault 用 AES256 原地加密敏感变量，--ask-vault-pass 解密，debug 只打指纹。挑端口还用上了 SELinux http_port_t 白名单——8008 零配置，18080 就得 semanage port 放行。"
