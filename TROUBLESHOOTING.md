# 故障复盘手册（Troubleshooting Log）

> 每出一次问题就记一次。格式：现象 → 排查过程 → 根因 → 解决方案 → 预防。
> 这份手册是本项目最有价值的资产之一——README 的踩坑记录从这里面挑。

---

## 目录

| # | 日期 | 问题 | 根因 | 状态 |
|---|------|------|------|------|
| 1 | 2026-10-12 | Ansible 角色找不到（role not found） | roles_path 缺失 | ✅ 已解决 |
| 2 | 2026-10-12 | Docker apt 源 404（bookworm Release 不存在） | 源 URL 写死 ubuntu，机器实为 Debian | ✅ 已解决 |
| 3 | 2026-10-12 | Too many authentication failures（反复出现） | MaxAuthTries=3 + 客户端双钥匙轮询 | ✅ 已解决 |
| 4 | 2026-10-12 | NAT 出口地址漂移（连到别人的机器） | 商家更换转发地址 195:10012 → 77:10011 | ✅ 已解决 |
| 5 | 2026-10-12 | playbook 全绿但跑完密码就进不去 | 加固先于密钥生效 + 连接复用掩盖 | ✅ 已解决 |
| 6 | 2026-10-12 | 密钥推了但 root 始终免不了密 | 公钥只推 ops，ansible_user=root | ✅ 已解决 |
| 7 | 2026-10-12 | assert 报 Conditional is marked as unsafe | jinja 套 jinja 嵌套 | ✅ 已解决 |
| 8 | 2026-10-12 | 指定密钥仍要密码（免密终局） | sshd 的 PubkeyAuthentication 被外部配置关闭 | ✅ 已解决 |
| 9 | 2026-10-12 | known_hosts 旧主机指纹残留 | NAT 换入口后同 IP:port 的主机密钥变了 | ✅ 已解决 |
| 10 | 2026-10-12 | 重装后 REMOTE HOST IDENTIFICATION HAS CHANGED | 重装=新主机密钥，known_hosts 旧指纹冲突（#9 的真实触发） | ✅ 已解决 |
| 11 | 2026-10-12 | 镜像拉取 502 Bad Gateway | 公共加速器 daocloud 高峰过载 | ✅ 已解决（多加速器冗余） |
| 12 | 2026-10-12 | Load key "/dev/null": bad permissions | Ansible 连接参数异常（待验证 ControlMaster 残留 socket） | 🔍 待确认 |

---

## 1. Ansible 找不到 role

- **现象**：`ERROR! the role 'common' was not found in .../playbooks/roles:...`
- **排查**：报错列表里全是 `playbooks/roles` 等路径——Ansible 只搜 playbook 同级目录，而我们 roles/ 在项目根。
- **根因**：playbook 在 `playbooks/` 子目录，角色搜索路径不含项目根。
- **解决**：项目根新增 `ansible.cfg`，`roles_path = roles`。顺带设了默认 inventory 和 host_key_checking。
- **预防**：目录结构偏离 Ansible 默认约定时，永远用 ansible.cfg 显式声明。
- **提交**：`ea5ac66`

## 2. Docker apt 源 404

- **现象**：`The repository 'https://mirrors.aliyun.com/docker-ce/linux/ubuntu bookworm Release' does not have a Release file.`
- **排查**：报错 URL 里出现 `bookworm`（Debian 12 代号）+ `ubuntu`（写死的路径段）——不存在的组合。
- **根因**：docker 角色 defaults 里源 URL 的发行版段硬编码 `ubuntu`，服务器实际是 Debian 12。`{{ ansible_distribution_release }}` 本身工作正常。
- **解决**：URL 改为 `{{ ansible_distribution | lower }}` 动态适配（Ubuntu→ubuntu，Debian→debian）。顺带把 unattended-upgrades 模板的 Ubuntu ESM 条目包进 `{% if ansible_distribution == 'Ubuntu' %}`。
- **预防**：任何发行版相关路径禁止硬编码；写 roles 时先问"如果换 Debian/CentOS 这行还对吗"。
- **提交**：`8c03452`

## 3. Too many authentication failures

- **现象**：ssh/Ansible 连接被服务端秒踢，报认证失败次数超限（不同阶段反复出现）。
- **排查**：`-vv` 输出显示客户端携带 id_rsa + id_ed25519 两把钥匙依次 offer（`Will attempt key` 两行），服务端 MaxAuthTries=3（我们自己加固设的）。
- **根因**：客户端默认枚举所有密钥轮试 + 服务端认证次数上限过紧。两把钥匙 + 失败的密码尝试轻松耗尽 3 次配额。
- **解决**：MaxAuthTries 放宽至 6（security defaults）；连接侧建议 `IdentitiesOnly=yes` 只端指定钥匙。
- **预防**：控制机有多把密钥时，inventory 永远加 `IdentitiesOnly=yes`；收紧 MaxAuthTries 前先想客户端钥匙数量。
- **提交**：`1a80bd8`

## 4. NAT 出口地址漂移

- **现象**：Ansible 连 `103.236.70.195:10012`，报错里实际对端却是 `103.236.70.77:10011`，且认证全废。
- **排查**：报错对端四元组和 inventory 不一致——连到的根本不是自己的机器。
- **根因**：NAT VPS 商家更换了转发地址（外部 IP 和端口都变了），旧四元组指向别处。
- **解决**：改 inventory 的 `ansible_host`/`ansible_port` 为新地址。
- **预防**：NAT 机器连不上且报错行为诡异时，第一件事核对"报错里的对端地址 == inventory 地址"；把商家联系方式记在 inventory 注释里方便问。
- **备注**：这台机器是"外部 10011 → 内部 22"型 NAT（sshd 监听 22），所以 `ssh_port`（期望状态）保持 22 不动，只改 `ansible_port`（连接方式）。两个变量各管各的，这次实地验证了这个设计。

## 5. playbook 全绿但跑完密码进不去

- **现象**：`site.yml` 38 任务全绿成功，紧接着 `ansible homelab-server -m ping -k` 连不上，`Permission denied`。
- **排查**：B 机 VNC 里 `sshd -T`：`passwordauthentication no`、`maxauthtries 3`——加固早已生效。
- **根因**：security 角色 play 中途重启 sshd 应用 `PasswordAuthentication no`，但 Ansible 用 SSH 连接复用（ControlPersist）一直搭开局的密码连接，play 自然跑完；play 结束后新连接面对的是已加固的 sshd，密码通道已关。加固生效早于密钥通道建立 → 唯一通道被自己锁死。
- **解决**：VNC 控制台删 `/etc/ssh/sshd_config.d/00-homelab-hardening.conf` 恢复密码登录；策略改为双通道（密码+密钥常开，救援通道保留）。
- **预防**：安全加固类 playbook 必须保证"新通道建立成功后才关闭旧通道"；homelab 场景保留密码通道是合理权衡（fail2ban 兜底），生产再收紧。SSH 连接复用会让"跑完全绿"具有欺骗性——加固是否生效要看新连接。
- **提交**：`1a80bd8`

## 6. 密钥推了但 root 免不了密

- **现象**：`authorized_key` 任务每轮 `changed/ok`，但 `ssh root@...` 始终要密码。
- **排查**：common 只推 `deploy_user`（ops）；inventory 是 `ansible_user=root`。用 ad-hoc 命令手动把公钥推给 root，立即通了——对照实验定案。
- **根因**：推对了钥匙，配错了门——密钥在 ops 手里，登录用的是 root。
- **解决**：common 角色新增任务，同一把公钥推给 root 和 ops 两个用户（`common_push_key_to_root` 开关）。
- **预防**：设计"密钥推送"时先问"Ansible 用哪个用户连接？"——推送对象必须覆盖连接用户。中间还有个烟雾弹：手动推完 root 公钥后 `ping` 不带 `-k` 成功过一次，其实是 ControlPersist 连接复用搭了便车，没做新认证，让人误判已通。
- **提交**：`5c68b2b`

## 7. assert 报 Conditional is marked as unsafe

- **现象**：自己加的密钥核对 assert 任务直接 fail：`The conditional check ... Conditional is marked as unsafe, and cannot be evaluated.`（附 WARNING：条件里不应有 jinja 定界符）。
- **根因**：assert 的 `that` 条目里写了 `"'{{ lookup(...) }}' in common_remote_keys.stdout"`——外层已处于 jinja 求值上下文，再嵌 `{{ }}` 属于模板嵌套，Ansible 判 unsafe 拒评。
- **解决**：直接删掉该核对任务（`authorized_key` 模块本身幂等，不需要额外核对层）。
- **预防**：`when`/`assert` 条件里引用变量**不带** `{{ }}`，直接写变量名；需要 lookup 就先 `set_fact` 存下来再引用。
- **提交**：`fdfee65`

## 8. 免密终局：指定密钥仍要密码

- **现象**：`ssh -o IdentitiesOnly=yes -i ~/.ssh/id_rsa root@B机` 仍提示输密码（输对才能进）——钥匙、权限全对但密钥认证不走。
- **排查链**（本手册最经典的一案）：
  1. `/root/.ssh` 权限 700/600 完美，authorized_keys 内容与 A 机公钥一致 → 排除权限
  2. 指纹比对一致 → 排除钥匙不匹配
  3. `ssh -v -o BatchMode=yes` 关键行：`Authentications that can continue: password` ——**sshd 协议层只宣布了 password 一种认证方式，公钥通道根本没开**
- **根因**：B 机 sshd 某处配置将 `PubkeyAuthentication` 关闭（商家镜像或历史残留，非我们 playbook 所写——加固模板里原本没有这行，属于"未声明的配置漂移"）。
- **解决**：加固模板显式写入 `PubkeyAuthentication yes`。关键在位置：片段以 `00-` 前缀放在 sshd_config.d/ 最先加载、优先级最高，任何别处的关闭都会被它压住。playbook 一跑自动收敛。
- **预防**：对"期望状态"的每一项都要显式声明，不能依赖默认值——默认值可能被镜像/上游/残留配置改掉。这正是 IaC 的核心价值：**发现的每一处漂移都固化成代码，下次自动纠正**。
- **提交**：`2a8a5c9`

## 9. known_hosts 旧主机指纹残留

- **现象**：NAT 换入口地址后，ssh/Ansible 连接时警告主机密钥变更，或行为异常（本机 known_hosts 里同一个 `[103.236.70.77]:10011` 存了 3 条旧记录，line 6/7/8）。
- **排查**：`ssh-keygen -f "/root/.ssh/known_hosts" -R "[103.236.70.77]:10011"` 输出显示该条目出现 3 次——同一 host:port 之前反复写入过不同指纹（NAT 出口漂移/重装系统都会造成"地址没变、后面的机器换了"，每次连的都是不同的 sshd，指纹自然不同）。
- **根因**：NAT 环境下 `[IP]:port` 这个标识对应的**实际机器会变**（换转发目标、机器重装），known_hosts 里按标识缓存的旧指纹与新机器不匹配。本项目 inventory 配了 `StrictHostKeyChecking=no`（ ansible.cfg 同样配置），所以没有直接报 `REMOTE HOST IDENTIFICATION HAS CHANGED` 被拒连，但旧条目属于脏数据，且若哪天去掉 no 配置会立刻炸。
- **解决**：`ssh-keygen -f "/root/.ssh/known_hosts" -R "[103.236.70.77]:10011"` 清掉旧条目（自动备份为 known_hosts.old），下次连接写入新机器的真实指纹。之后 playbook 正常全绿。
- **预防**：
  - NAT 漂移/目标机重装后，顺手 `-R` 清旧指纹再连（或直接删 known_hosts 对应行）
  - `[非默认端口]` 的条目在 known_hosts 里**必须带方括号**：`[ip]:port`，`-R` 时也要带，不带匹配不到
  - 生产环境别长期用 `StrictHostKeyChecking=no`，它是把这类警告静音的掩盖型配置；homelab 图省事可用，但要知道代价是放弃了主机身份校验

---

## 10. 重装系统后 REMOTE HOST IDENTIFICATION HAS CHANGED

- **现象**：重装 B 机后跑 playbook，`@@@@@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @@@@@`，且"Password authentication is disabled to avoid man-in-the-middle attacks"——密码也不让用了。
- **排查**：这是 #9 的**真实触发**（上次是静默状态）：重装系统 = 机器生成了全新的 sshd 主机密钥，known_hosts 里 `[103.236.70.77]:10011` 缓存的还是旧机器指纹，ssh 按防中间人攻击处理，直接断连并禁用密码认证。
- **根因**：主机身份标识（IP:port）没变，后面的机器换了。
- **解决**：`ssh-keygen -f "/root/.ssh/known_hosts" -R "[103.236.70.77]:10011"`（输出显示旧条目在 line 6/7/8 共 3 条，全部清除）→ 重跑 `-k` 正常。
- **预防**：**重装/换转发目标后必跑一次 `-R`**，这是固定动作。进阶：把主机指纹管理做进流程（如 `known_hosts` 随 IaC 更新），或接受 `StrictHostKeyChecking=no` 的代价。

## 11. 镜像拉取 502 Bad Gateway

- **现象**：重装后全新部署，TeamSpeak 镜像拉到一半：`docker.m.daocloud.io ... blobs/sha256:4f4fb... 502 Bad Gateway`，`up -d` 失败。
- **排查**：502 出自**加速器服务端**（daocloud 代理 docker.io 的 blob 请求失败），不是网络不通——前面层都拉成功了，单个 blob 断流。公共免费加速器高峰期常态。
- **根因**：daemon.json 只配了单个公共加速器，无冗余。
- **解决**：重跑即可（pull 断点续传，502 是瞬时故障）；根治是 defaults 里 `docker_registry_mirrors` 配两个加速器按序回退（daocloud + 1ms.run），最稳是换成阿里云个人专属加速器地址（`https://<你的ID>.mirror.aliyuncs.com`，group_vars 覆盖）。
- **预防**：任何第三方公共依赖（镜像源、DNS、apt 源）都要有 plan B；国内拉 Docker Hub 属于"常态不稳"，加速器冗余是标配而非可选。
- **提交**：多加速器冗余配置

## 12. Load key "/dev/null": bad permissions（待确认）

- **现象**：带 `-k` 跑完一轮（changed=23，死于 #11 的 502）后，不带 `-k` 再跑：`WARNING: UNPROTECTED PRIVATE KEY FILE! Permissions 0666 for '/dev/null' are too open` → `Permission denied (publickey,password)`。
- **排查中**：ssh 客户端被要求加载 `/dev/null` 当私钥——`-i` 参数被空值/占位顶掉。嫌疑：Ansible 的 ControlMaster 复用 socket 残留（`/root/.ansible/cp/` 下指向已死会话的 socket），或 `-k` 模式下连接参数拼接异常。**待跑：`rm -rf /root/.ansible/cp` 清 socket 后重试不带 -k；`ls -la /root/.ssh/id_rsa` 确认钥匙存在。**
- **预防**（暂定）：连不上时先清 ControlMaster socket 目录再下结论，避免把连接复用的残留当成新问题。
- **状态**：🔍 待用户验证后补充结论

---

## 附：ssh 免密排查三步法（本案沉淀）

ssh 免密失败时，把三个环节分开验证，**顺序很重要**（后面的环节故障会完全挡住前面环节的验证）：

1. **门开不开**（先查这个！）：`ssh -v ...` 看 `Authentications that can continue` ——只有 password 说明 PubkeyAuthentication 没开，公钥全对也没用
2. **钥匙对不对**：两边 `ssh-keygen -lf` 比对公钥指纹
3. **门上有没有你的钥匙**：目标用户的 `~/.ssh/authorized_keys` 内容 + `.ssh` 700 / `authorized_keys` 600 权限 + 属主正确

其他高价值检查点：

- 报错里的**对端地址**是否等于 inventory 地址（NAT 漂移）
- 客户端**带了几把钥匙**（`ssh -v` 的 `Will attempt key` 行数 vs 服务端 MaxAuthTries）
- "刚才还通着"≠现在通：SSH 连接复用（ControlPersist）会掩盖服务端状态变化，验证必须开新连接
- `sshd -T` 查生效配置时，关键词要查全（本案例漏查 `pubkeyauthentication` 导致绕了大弯路）
- Debian 12 的 sshd 日志在 journald（`journalctl -u ssh`），不是 /var/log/auth.log
