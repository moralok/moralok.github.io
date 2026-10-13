---
title: Vaultwarden Backup 邮件通知排障：代理、认证与容器测试
date: 2026-10-12 05:29:26
tags: [vaultwarden, smtp]
---

本文记录一次 `Vaultwarden Backup` 邮件通知失败的排查过程。最初日志显示 `Unexpected EOF on SMTP connection`，测试发现 `SMTP` 流量经过代理时无法收到服务器问候，改为直连后连接恢复；随后又遇到 `535 5.7.3 Authentication unsuccessful`，进一步确认当前 Outlook 账号不再接受这套基本认证配置。最终，更换为 Serv00 提供 `SMTP` 服务，并成功收到备份邮件通知。排查中还发现，直接进入容器运行 `mail`，与通过镜像入口执行邮件测试并不等价。本文分别说明这些问题的现象、验证过程和处理结果。

<!-- more -->

## 项目背景

### Vaultwarden 介绍

`Vaultwarden` 是一个用 `Rust` 语言编写的轻量级、开源的第三方 `Bitwarden` 服务端实现。

**核心特点**

• 高度兼容：完全兼容 `Bitwarden` 的官方客户端（包括浏览器扩展、桌面端、`iOS` 和 `Android` 移动端）。
• 极其轻量：资源占用极低（通常只需几十兆内存），一台配置很小的主机或 `NAS` 就能流畅运行。
• 端到端加密：数据在本地设备加密后才会上传，服务端只存密文，保护隐私安全。
• 免费且功能丰富：支持组织密码共享、两步验证等官方需要付费的部分高级功能。

<div style="width:90%;margin:auto">{% asset_img "docker stats of vaultwarden.png" Vaultwarden 资源占用 %}</div>

### vaultwarden-backup 介绍

[ttionya/vaultwarden-backup](https://github.com/ttionya/vaultwarden-backup) 是一个专为 `Vaultwarden` 量身定制的开源自动备份工具。它基于 `Docker` 容器化部署，利用强大的底层传输工具 `Rclone`，能够安全、高效、自动化地将自建密码库的数据备份并同步到几乎所有主流的云存储或远程系统中。

### SMTP 邮件发送失败日志

`vaultwarden-backup` 可以设置在备份成功或失败时通过 `SMTP` 发送邮件通知。最开始注意到邮件发送失败时，我并没有马上处理，一方面是因为服务已经稳定运行多年，数据备份正常；另一方面是因为当初启用 Outlook 邮箱 `SMTP` 的过程有点麻烦，非必要实在不想访问微软的网站。

报错的日志如下：

```text
backup-1       | upload backup file to storage system [BitwardenBackup:/BitwardenBackup]
backup-1       | mail: Warning: variable superseded or obsoleted: smtp
backup-1       | mail: Warning: variable superseded or obsoleted: smtp-auth-user
backup-1       | mail: Warning: variable superseded or obsoleted: smtp-auth-password
backup-1       | mail: Obsoletion warning: please do not use *smtp*, instead assign a smtp:// URL to *mta*!
backup-1       | mail: Obsoletion warning: Use of old-style credentials, which will vanish in v15!
backup-1       | mail:   Please read the manual section "On URL syntax and credential lookup"
backup-1       | mail: Unexpected EOF on SMTP connection
backup-1       | /root/dead.letter 10/324
backup-1       | mail: ... message not sent
backup-1       | mail sending has failed
backup-1       | completion ping has been sent successfully
```

当时心里推测的可能原因有两个：

1. Outlook 邮箱的 `app password` 可能由于某种安全原因失效了
2. 网络流量经过 `mihomo` 代理分发导致了某种问题

事实上，**最终的排查结果是两方面的原因都有**：

1. Outlook 邮箱 `SMTP` 不再支持基本认证，要求使用 `OAuth2`（不排除不同类型，不同个体的 Outlook 账号仍支持基本认证的可能性）
2. 使用的代理服务器屏蔽 `25/465/587` 端口，防止垃圾邮件

## 排查过程

> 最近 `Homelab` 更新告一段落，着手处理这个问题，没想到又是一个大坑，埋了我大半天。Orz……

### 重置 app password

第一步，重置 `app password`。

Microsoft 账户 → 安全 → 管理登录方式：

1. 确认双重验证已开启（这是使用 `app password` 的前提条件）
2. 删除现有的 `app password`（应用密码） 并创建新的

### 邮件发送测试

第二步，使用新的 `app password` 测试是否可以发送邮件。

在 `vaultwarden-backup` 项目的 [README_zh.md](https://github.com/ttionya/vaultwarden-backup/blob/master/README_zh.md) 中，有一段关于邮件发送测试的说明。

> 你可以使用下面的命令测试邮件发送功能。我们会增加 `-v` 标志以显示详细信息，你无需在 `MAIL_SMTP_VARIABLES` 中重复设置。

```bash
docker run --rm -it -e MAIL_SMTP_VARIABLES='<your smtp variables>' ttionya/vaultwarden-backup:latest mail <mail send to>

# Or

docker run --rm -it -e MAIL_SMTP_VARIABLES='<your smtp variables>' -e MAIL_TO='<mail send to>' ttionya/vaultwarden-backup:latest mail
```

因为我的 `Vaultwarden` 服务正在正常运行，所以我按个人习惯进入 `backup` 容器进行测试。

```bash
docker exec -it vaultwarden-backup-1 bash
mail another-example.outlook.com
```

> **这里我其实踩到了一个深坑，只是无意中 AI 带着我飞过去了。提前剧透，作者的测试命令中的 `mail` 和我使用的命令中的 `mail` 指向的一个东西。**


刚开始我以为容器会根据配置设置好环境变量，但是测试报错，报错的核心信息 `sendmail: can't connect to remote host (127.0.0.1): Connection refused` 和 20240518 open 至今的 [issue \#153](https://github.com/ttionya/vaultwarden-backup/issues/153) 提到的报错信息相似。

```
1e4cb0d00964:/data# mail another-example.outlook.com
Subject: Test inside container
To: another-example.outlook.com
This is a test message from container.
^D
-------
(Preliminary) Envelope contains:
To: another-example.outlook.com
Subject: Test inside container
Send this message [yes/no, empty: recompose]? yes
sendmail: can't connect to remote host (127.0.0.1): Connection refused
/root/dead.letter 8/195
mail: ... message not sent
```

我的第一反应是怀疑 `SMTP` 相关设置没有被正确传递。[issue \#153](https://github.com/ttionya/vaultwarden-backup/issues/153) 的评论区有人有类似的想法：

> EDIT: Also want to add this. I think connection refused with 127.0.0.1 means that smtp or mta was not set correctly and is trying to connect to default smtp which is localhost.

测试发现环境变量 `MAIL_SMTP_VARIABLES` 确实为空，我只当是因为自己使用 `.env` 导致。

```
export MAIL_SMTP_VARIABLES=" \
-S smtp-use-starttls \
-S smtp=smtp://smtp.office365.com:587 \
-S smtp-auth=login \
-S smtp-auth-user=example@outlook.com \
-S smtp-auth-password=example-password \
-S from=example@outlook.com another-example@outlook.com"
```

但是在手动 `export` 后重新测试发现仍然报错。

```text
sendmail: can't connect to remote host (127.0.0.1): Connection refused
```

### SMTP 服务器可达性测试

其实到这里时我有过一瞬间的担心，先想起前段时间更新 `Homelab` 时卡壳多天的网络问题，不过又想起放下精神洁癖后让 AI 自己突突了 2 小时就解决了。于是我果断让 AI 进行第三步，测试容器和 `SMTP` 服务器是否可以连接。**果不其然，可恶，又有网络问题**。AI 排查过程简单处理后如下：

1. 确认可以网络可达，建立过连接

```text
Unexpected EOF 说明连接建立过，但 SMTP 对话中途断了。先看容器里的实际情况：

容器 nc 测试:   smtp.office365.com (198.18.2.112:587) open     ← TCP 握手成功
mihomo 日志:    [TCP] --> smtp.office365.com:587 using JMS[...]  ← 有日志！走了代理

宿主机 python:  :587  连接成功 8.01s  banner=<no banner: TimeoutError>
                :25   连接成功 8.01s  banner=<no banner: TimeoutError>
                ↑ TCP 连上了，但【SMTP 服务器一句问候都不回】
```

2. 确认是代理节点封了 `SMTP` 端口，还是 Office365 封了代理节点的出口 `IP`。

```text
Office365 :587   8秒无数据 ✗
Gmail     :587   8秒无数据 ✗
163       :587   8秒无数据 ✗
Gmail     :465   8秒无数据 ✗
Office365 :443   8秒无数据 ✗
QQ邮箱    :587   0.20s  220 newxmesmtplogicsvrsza73-0.qq.com XMail Esmtp  ✓

只有国内的 QQ 秒回——因为它走 DIRECT 不走代理。所有走代理的国外 SMTP 全部卡死。

同一个 IP、同一个端口，直连通、走代理死

smtp.office365.com  (真实 IP ******)
  绕过代理直连   0.24s  220 TP0P295CA0031.outlook.office365.com Microsoft ESMTP MAIL  ✓
  经 tun/代理    9.01s  <无任何数据>                                                     ✗

smtp.gmail.com      (真实 IP ******)
  绕过代理直连   0.63s  220 smtp.gmail.com ESMTP ...                                   ✓
  经 tun/代理    9.01s  <无任何数据>                                                     ✗
```

结论是**使用的代理服务器屏蔽了 `25/465/587` 端口**，修改 `mihomo` 规则走 Direct：

- 可以按端口
```
- DST-PORT,587,DIRECT
- DST-PORT,465,DIRECT
- DST-PORT,25,DIRECT
```
- 也可以按域名
```
- DOMAIN,smtp.office365.com,DIRECT
```

在修复网络问题后，AI 立刻发现了新的报错 `535 5.7.3 Authentication unsuccessful`。

### SMTP 可用性测试

显然，这是微软的响应。第四步，测试 Outlook 邮箱的 `SMTP` 功能是否可用。**在某种程度上这部分是最折磨的，因为 AI 工具介入不便，又因为微软的网站巨卡，还因为敏感操作要反复进行身份验证**。

确认并测试：

1. 邮箱是否正确
2. `app password` 是否从正确的账号上生成、复制并被正确粘贴
3. `SMTP` 服务器域名是否从 `smtp.office365.com` 变更为 `smtp-mail.outlook.com`
4. 双重验证是否开启

但报错信息始终不变，而且有点坑的是 `535 5.7.3 Authentication unsuccessful` 这种模糊的报错信息虽然阻止了攻击者依靠错误信息探测某个账号是否存在、是否启用了什么策略，但是也将账号所有者挡在信息之外。

> 测试过程中，注意到进入容器内的测试报错始终是 `sendmail: can't connect to remote host (127.0.0.1): Connection refused`，怀疑两条命令中的 `mail` 指向不同，先改用文档提供的测试方法，原因之后分析。

```text
mail: Resolving host smtp.office365.com:587 ... done
mail: Connecting to 40.99.10.114:587 ... connected.
mail:  Certificate depth 2
mail:   subject = /C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root G2
mail:   notBefore = Aug  1 12:00:00 2013 GMT
mail:   notAfter = Jan 15 12:00:00 2038 GMT
mail:   issuer = /C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root G2
mail:  Certificate depth 1
mail:   subject = /C=US/O=DigiCert Inc/CN=DigiCert Global G2 TLS RSA SHA256 2020 CA1
mail:   notBefore = Mar 30 00:00:00 2021 GMT
mail:   notAfter = Mar 29 23:59:59 2031 GMT
mail:   issuer = /C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root G2
mail:  Certificate depth 0
mail:   subject = /C=US/ST=Washington/L=Redmond/O=Microsoft Corporation/CN=outlook.com
mail:   notBefore = Jun 26 00:00:00 2026 GMT
mail:   notAfter = Jan 10 23:59:59 2027 GMT
mail:   issuer = /C=US/O=DigiCert Inc/CN=DigiCert Global G2 TLS RSA SHA256 2020 CA1
mail: Comparing subject_alt_name: need<smtp.office365.com> is<outlook.com>
mail: Comparing subject_alt_name: need<smtp.office365.com> is<*.hotmail.com>
mail: Comparing subject_alt_name: need<smtp.office365.com> is<*.internal.outlook.com>
mail: Comparing subject_alt_name: need<smtp.office365.com> is<*.live.com>
mail: Comparing subject_alt_name: need<smtp.office365.com> is<*.office.com>
mail: Comparing subject_alt_name: need<smtp.office365.com> is<*.office365.com>
mail: TLS certificate ok
mail: TLS BLAKE2s256 fingerprint: 57:13:29:BD:2E:18:88:2C:9F:34:2B:03:6D:C4:E7:C4:1D:A3:4A:55:7E:46:7A:7A:8A:D0:74:8C:8B:1D:E6:C4
mail: TLS connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
mail: SMTP server: 535 5.7.3 Authentication unsuccessful [SI2PR04CA0013.apcprd04.prod.outlook.com 2026-10-12T12:54:49.248Z 08DF283A92EB71C0]
/root/dead.letter 10/297
mail: ... message not sent
mail sending has failed
```

期间，AI 多次建议在 Thunderbird 客户端配一次 `SMTP` 验证是 `s-nail` 兼容性问题，密码问题，还是账号基本认证被禁用问题，或者直接更换国内发信服务商，但我还是想挣扎一下。
直到 AI 通过 `IMAP` 服务器拿到了更具体的响应 `Basic authentication is disabled`，判断微软确实已经禁用这个账号的基本认证，要求使用 `OAuth2`；进一步搜索到 [n8n Docs](https://docs.n8n.io/integrations/builtin/credentials/send-email/outlook)声称“微软已移除 Outlook `SMTP` 的基本认证和 `app passwords`”；最后根据【转发和 IMAP 页面】没有 SMTP 相关设置判断此路注定不通，我才终于死心，放弃使用 Outlook 作为发信服务商。

<div style="width:70%;margin:auto">{% asset_img "outlook forwarding and imap setting.png" Outlook 转发和 IMAP 设置 %}</div>

### 更换发信服务商

最终，我参考[SMTP on Serv00（仅需域名使用基于serv00的SMTP服务）](https://linux.do/t/topic/97186)选择使用 Serv00 作为发信服务商，测试成功发送邮件通知。 相信选择 QQ 或者 163 邮箱也同样可以。


## mail 不是 mail

命令 `docker run …… mail ……` 中的 `mail` 和进入容器后运行的 `mail` 不是一个东西，如果没有注意，后者的测试结果将会一直是 `sendmail: can't connect to remote host (127.0.0.1): Connection refused`。

### 惯性思维

如果你要连接容器中的 `Redis` 服务，你可能会使用一个用完就删的临时容器：

```
docker run -it --rm redis redis-cli -h <你的Redis服务IP或主机名> -p 6379
```

你还可能会想直接进入已经存在的 `Redis` 容器内部使用自带的 `redis-cli`：

```sh
docker exec -it <你的Redis容器名或ID> redis-cli

# 或者
docker exec -it <你的Redis容器名或ID> sh
# 进入后在容器内输入：redis-cli
```

所以，当我看到文档中提供的邮件发送测试方法：

```sh
docker run --rm -it -e MAIL_SMTP_VARIABLES='<your smtp variables>' ttionya/vaultwarden-backup:latest mail <mail send to>
```

我以为效果等同于：

```sh
docker exec -it vaultwarden-backup sh
mail <mail send to>
```

### 参数 vs 命令

在执行 `docker run` 时，`Dockerfile` 中配置的 `ENTRYPOINT` 脚本会作为容器启动时的第一个主进程（`PID 1`）优先运行。
- 如果 `Dockerfile` 中同时有 `ENTRYPOINT` 和 `CMD`，`CMD` 的内容会被当作参数传递给 `ENTRYPOINT` 脚本。
- 如果在 `docker run` 后面追加了命令参数（例如 `docker run <image> echo hello`），这些参数会追加到 `ENTRYPOINT` 后面，并覆盖掉 `Dockerfile` 中的 `CMD`。

查看项目源码，`scripts/entrypoint.sh` 中有以下一段代码：

```shell
. /app/includes.sh

# mail test
if [[ "$1" == "mail" ]]; then
    export_env_file
    init_env_display
    init_env_mail

    MAIL_SMTP_ENABLE="TRUE"
    MAIL_DEBUG="TRUE"

    if [[ -n "$2" ]]; then
        MAIL_TO="$2"
    fi

    send_mail "${DISPLAY_NAME} Backup Test" "Your SMTP configuration looks correct."

    exit 0
fi
```

而 `send_mail` 方法定义在 `scripts/includes.sh` 中：

```shell
function send_mail() {
    local MAIL_VERBOSE_FLAG=""
    local MAIL_TEMPLATE_FLAG=""
    local MAIL_TEMPLATE=""
    local MAIL_USED_MESSAGE_ID=""

    # verbose
    if [[ "${MAIL_DEBUG}" == "TRUE" ]]; then
        MAIL_VERBOSE_FLAG="-v"
    fi

    # template
    if [[ "${MAIL_USE_THREAD}" == "TRUE" ]]; then
        if [[ -n "${MAIL_PARENT_MESSAGE_ID}" ]]; then
            MAIL_USED_MESSAGE_ID="${MAIL_PARENT_MESSAGE_ID}"
            MAIL_TEMPLATE="References: ${MAIL_USED_MESSAGE_ID}\nIn-Reply-To: ${MAIL_USED_MESSAGE_ID}\n\n"
        else
            MAIL_USED_MESSAGE_ID="<$(date +%s%N).$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 16).$(hostname)@vaultwarden.backup>"
            MAIL_TEMPLATE="Message-ID: ${MAIL_USED_MESSAGE_ID}\n\n"
        fi
    fi
    if [[ -n "${MAIL_TEMPLATE}" ]]; then
        MAIL_TEMPLATE_FLAG="-t"
    fi

    echo -e "${MAIL_TEMPLATE}$2" | eval "mail ${MAIL_VERBOSE_FLAG} ${MAIL_TEMPLATE_FLAG} -s \"$1\" ${MAIL_SMTP_VARIABLES} \"${MAIL_TO}\""
    if [[ $? != 0 ]]; then
        color red "mail sending has failed"
    else
        color blue "mail has been sent successfully"

        if [[ "${MAIL_USE_THREAD}" == "TRUE" && -z "${MAIL_PARENT_MESSAGE_ID}" ]]; then
            mkdir -p "$(dirname "${MAIL_PARENT_MESSAGE_ID_FILE}")"
            echo -n "${MAIL_USED_MESSAGE_ID}" > "${MAIL_PARENT_MESSAGE_ID_FILE}"
        fi
    fi
}
```

因此，如果想要进入容器直接进行测试，必须正确使用 `mail` 命令。比如：

```sh
mail -S smtp-use-starttls \
-S smtp=smtp://smtp.office365.com:587 \
-S smtp-auth=login \
-S smtp-auth-user=example@outlook.com \
-S smtp-auth-password=example-password \
-S from=example@outlook.com another-example@outlook.com
```

### `.env` 和容器内环境变量

另外，之所以刚开始测试时容器内环境变量 `MAIL_SMTP_VARIABLES` 为空，是因为在 `Docker Compose` 中，项目目录下的 `.env` 文件和容器内的环境变量是两个不同的概念，它们之间不会自动划等号。`.env` 不会自动注入容器，必须通过以下两种方式“桥接”到容器内：

**通过 environment 显式引用**

在 `docker-compose.yml` 中使用 `${VAR_NAME}` 语法，把 `.env` 里的值显式传给容器的 `environment` 节点：

```yaml
services:
  web:
    image: nginx
    environment:
      - DATABASE_HOST=${DATABASE_HOST} # 从 .env 插值并传入容器
```

转换链路：`.env` 文件 ➔ `Compose` 解析并替换 `docker-compose.yml` 中的 `${DATABASE_HOST}` ➔ 启动容器时通过 `-e` 注入到容器运行时环境。

**通过 env_file 整体加载**

如果某个服务需要加载整个文件，可以在 `docker-compose.yml` 中通过 `env_file` 指定：

```yaml
services:
  web:
    image: nginx
    env_file:
      - .env # 明确告诉 Compose 把这个文件注入到该容器
```

转换链路：`Compose` 读取指定的 `.env` 文件，在创建容器时将其中的 `KEY=VALUE` 批量注入容器环境。

## 总结和反思

### 总结

**最终排查出的问题有两方面**：

1. Outlook 邮箱 `SMTP` 不再支持基本认证，要求使用 `OAuth2`（不排除不同类型，不同个体的 Outlook 账号仍支持基本认证的可能性）。文档参见[POP, IMAP, and SMTP settings for Outlook.com](https://support.microsoft.com/en-us/outlook/pop-imap-and-smtp-settings-for-outlook-com)。
2. 使用的代理服务器屏蔽 `25/465/587` 端口。可见，透明代理也不是一劳永逸。

排查有三个主要方向：

1. 确认发信服务商支持 `SMTP` 及其使用方式
2. 报错信息为 `sendmail: can't connect to remote host (127.0.0.1): Connection refused`，极有可能是 `SMTP` 相关设置没有正确传递给最终的二进制可执行程序 `mail`。
3. 报错信息为 `Unexpected EOF on SMTP connection`，检查网络是否走代理以及代理服务器是否屏蔽 `25/465/587` 端口。

### 反思

1. 为了网络畅通必须引入代理，但有时候会产生一些奇怪的问题，你往往很难定位到底是哪个环节出了问题。但在 AI 能够“触达和上手”之后，排查和测试变得容易非常多，它比半吊子的自己更知道如何从更底层获取信息并发现关键点。
2. 马后炮地说，如果直接换 QQ 或 163 邮箱，好像一切自然迎刃而解，但避开了坑 vs 趟过了坑，各有好处。
3. 善游者溺，自以为已经熟悉的地方也可能有自己不了解的坑。
    - 几年前 Outlook 邮箱 `SMTP` 要求必须开启双重验证、使用 `app password` 而不能直接使用主密码。几年后 Outlook 禁用了基本认证，要求使用 OAuth2。年年坑不同。不过难道微软完全放弃那些不支持 `OAuth2` 的场景了？
    - `Docker` 常常用，但常用的知识其实很少，很多时间一长大部分就模糊了，而且有些关键点，如果不是踩过坑，可能根本留意不到。
