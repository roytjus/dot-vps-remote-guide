# 让 dot 连接已有的 Linux VPS

一次通过 Remote Desktop Commander 跑通远程命令的实践记录。

记录日期：2026 年 10 月 6 日。下文的 dot 指我的 AI 助手。

## 这次解决了什么

我们把 dot 接到了我已有的 Linux VPS 上。完成连接后，dot 从自己这一端确认设备在线，并在 VPS 上执行了主机名、用户身份和项目目录存在性检查，拿到了实际返回结果。

这次验证的是远程命令通道。自动部署、视频上传、长期稳定运行等后续工作，还需要分别验证。

## 一开始卡在了哪里

最初尝试从 dot 当时使用的云端环境直接 SSH 到 VPS，报了网络不可达错误，连接在 SSH 身份认证之前就失败了。进一步检查时，该环境没有可用的直连 IPv4 路由，但经代理的 HTTPS 请求可以成功。

这两条网络路径要分开判断：HTTPS 能访问，不足以证明 SSH 端口也可达。遇到这类错误，应先检查网络路径，再考虑密钥和登录权限。

这里只记录当时那个环境的结果，不能据此推断所有 OpenAI 云端环境都无法使用 SSH。

## 最后跑通的连接方式

我们用了 Desktop Commander 的 Remote MCP。它支持让网页端 AI 客户端调用另一台机器上的工具，命令仍在目标机器执行。官方入口和启动说明见 [Desktop Commander 项目说明](https://github.com/wonderwhy-er/DesktopCommanderMCP#install-in-other-clients)。

连接过程是：

1. 由已经有权管理 VPS、也能连接它的一方，在 VPS 上启动 Remote Device。
2. 我完成设备配对。
3. 我在 AI 客户端连接 Remote Desktop Commander，并完成另一端的授权。
4. dot 通过这条连接发出命令，在 VPS 上执行并返回结果。

这里最有用的协作方式，是让已经能进入 VPS 的一方完成首次启动，再让 dot 接手验证。这个角色可以是服务器所有者、已授权的管理员，或已有访问能力且获得明确授权的助手，不依赖某个特定产品。

## 复现前先确认权限

这套方案会赋予 AI 远程执行命令的能力。请只连接自己有权管理的机器，并看清授权范围。

一般建议使用独立的非 root 用户，只给它完成项目所需的系统权限。这次验证使用 root，是我明确授权的选择；照搬到其他服务器会把影响范围扩大到整台机器，不适合作为默认配置。

如果需要严格限制 AI 能接触的文件和系统资源，应使用经过正确配置的容器或虚拟机等操作系统级隔离。Desktop Commander 的目录白名单和命令黑名单只能减少误操作，不能当作安全沙箱。详见 [官方安全模型](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/main/SECURITY.md)。

## 连接步骤

### 1 在 VPS 上准备运行环境

通过你已有的管理方式登录 VPS，切换到计划运行 Remote Device 的用户，检查：

```bash
node --version
npm --version
```

本次成功运行时，Node.js 为 24，Desktop Commander 为 0.2.52。服务器最初看到的是 Node.js 18，因此记录环境时，要以实际启动 Remote Device 的运行时为准。

项目的 [package.json](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/main/package.json) 声明 Node.js 最低版本为 18，但最低版本声明不等于仍受维护。按本文日期，Node.js 18 已结束支持，Node.js 24 为 LTS。新部署建议选择仍受支持、且与所用包版本兼容的版本，参见 [Node.js 发布状态](https://nodejs.org/en/about/previous-releases)。

### 2 启动 Remote Device 并配对

官方推荐的启动命令是：

```bash
npx @wonderwhy-er/desktop-commander@latest remote
```

首次运行按终端提示打开验证页面，核对页面与终端的验证码，再由账号所有者登录并授权设备。无图形界面的 VPS 可以在自己的浏览器打开终端给出的验证地址。具体流程见 [官方 Remote Device 文档](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/main/src/remote-device/README.md#quick-start)。

不要把验证码、完整授权链接或登录凭据发到公开仓库、帖子和截图中。

`@latest` 会随发布变化。记录自己实际使用的版本，后续排查才有可比性；本文的 0.2.52 是本次实测版本，不代表以后始终最新。

### 3 在 AI 客户端完成连接

设备配对后，还要在 AI 客户端连接 Remote Desktop Commander。我们这次在 ChatGPT 中完成了该连接的 OAuth 授权，两端使用同一个 Desktop Commander 账号。

设备显示 Online，只能说明设备端已经接入。还要确认 AI 客户端已经连接到对应账号，并且能看到这台设备。不同客户端的入口可能变化，请从 [官方 Remote MCP 入口](https://mcp.desktopcommander.app/) 查看当前连接方式。

### 4 从 dot 这一端验收

让 dot 先列出在线设备，确认选中的就是目标 VPS，再执行只读检查：

```bash
hostname
id
```

然后让它检查已授权项目目录是否存在。不要一上来就读取配置文件、打印环境变量或运行部署脚本。

我们这次确认了：

- dot 能看到目标设备在线
- 远程命令返回了目标机器的主机信息和用户身份
- 已授权项目目录的存在性检查通过

公开记录保留这些检查项即可，实际 IP、主机名、账号、设备标识和项目路径应隐藏。

## 几个容易误判的地方

**设备配对成功后，AI 仍然不能调用。** 分别检查设备进程、在线状态、AI 端连接，以及两端账号是否一致。我们实际遇到的关键点，就是设备登录和客户端连接需要分别完成。

**网页可以打开，SSH 却失败。** 按协议和路径分别检查。代理支持的 HTTPS 请求和直连 SSH 不能互相代替验收。

**设备在线就算完成。** 还应从真正使用它的 AI 客户端执行一次无副作用命令，确认结果来自目标机器。

**关掉进程就等于撤销授权。** 停止 Remote Device 会中断连接，但已保存的授权仍可能存在。正式停用时，需要分别处理设备授权与 AI 客户端连接，不能只关掉终端。官方区分了 [停止、退出登录、撤销设备和断开连接](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/main/src/remote-device/README.md#stop-logout-revoke-or-disconnect)。

## 还没有完成的部分

本次运行是临时启动，没有配置开机自启。进程退出或 VPS 重启会让连接中断，不能当作已经完成了长期托管。

如果后续确实需要常驻，再单独决定运行用户、进程管理、自动重启、更新和日志策略，并实际验证重启后的恢复情况。

命令及结果会经过 Remote MCP 服务。请避免输出密钥和其他不需要交给 AI 处理的数据；若准备公开日志或截图，先做脱敏。数据路径及记录方式见 [官方 Remote Device 文档](https://github.com/wonderwhy-er/DesktopCommanderMCP/blob/main/src/remote-device/README.md#security-and-history)。

## 这次值得留下的经验

先确认哪一段连接失败，再选择可用的接入方式。让已有访问能力的一方完成初始化，由账号所有者完成授权，最后从使用端做实际验收。这几步分清楚，排查就容易很多。

这是一条在我们当前环境中已经跑通的路径。换机器、账号、客户端或软件版本之后，仍然要重新验证权限和实际调用结果。
