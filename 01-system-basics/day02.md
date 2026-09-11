# Day 02 · dnf / rpm 深度与排障实战
- dnf history undo：删掉 nginx 后用 ID 回滚，14 个包全部恢复
- systemctl enable 的实质：在 multi-user.target.wants/ 下建软链接
- nginx master/worker 架构：worker 数 = CPU 核数（本机 2 核 2 worker）
- 排障实战：nginx 已启动但 Windows 浏览器访问不通
  排查顺序：ss -tlnp 查监听 → curl 本机验证 → firewall-cmd 查防火墙
  根因：firewalld 默认仅放行 ssh，未放行 http；本机 curl 200 但外部不通
  解决：firewall-cmd --permanent --add-service=http && firewall-cmd --reload
