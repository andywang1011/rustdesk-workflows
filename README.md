# rustdesk-workflows

基于 GitHub Actions 编译的自定义 RustDesk Windows 客户端。

## 构建配置

本仓库不保存任何连接配置。所有值均以加密的 **GitHub Secrets** 存储，仅在构建时由工作流读取：

| Secret | 用途 |
| --- | --- |
| `ID_SERVER` | ID / 信令服务器 |
| `RELAY_SERVER` | 中继服务器 |
| `API_SERVER` | API 服务器（可选） |
| `PUBLIC_KEY` | 服务器公钥（可选） |
| `FIXED_PASSWORD` | 固定无人值守密码（可选） |
| `UNLOCK_PIN` | 设置页解锁 PIN（可选，明文注入，各机器自行加密） |

源码基于上游 `rustdesk/rustdesk`（master）构建，编译后的客户端使用上述固定密码连接指定服务器。

## 自定义默认设置

工作流中的 "Patch RustDesk custom config" 步骤会在构建时修改
`libs/hbb_common/src/config.rs`，使新安装的客户端默认启用以下设置：

| 设置项 | 选项键 | 默认值 |
| --- | --- | --- |
| 常规-启动时检查软件更新 | `enable-check-update` | `N`（不打勾） |
| 常规-启用UDP打洞 | `enable-udp-punch` | `Y`（打勾） |
| 安全-拒绝局域网发现 | `enable-lan-discovery` | `N`（打勾） |
| 安全-允许远程修改配置 | `allow-remote-config-modification` | `Y`（打勾） |

- 前两项注入 `LocalConfig::load()`（文件 `_local`），后两项注入
  `Config2::load()`（文件 `"2"`），与应用读取这些选项的位置一致
  （`LocalConfig::get_option`、`Config::get_option`）。
- 仅在对应键**不存在**时写入（`contains_key` 保护），不会覆盖用户已有设置。
- `enable-lan-discovery = "N"` 即"拒绝局域网发现"打勾状态：界面通过
  `_denyLANDiscovery = !option2bool("enable-lan-discovery", …)` 取反显示。
- `allow-remote-config-modification = "Y"` 表示"允许"（打勾）。

## 解锁 PIN（可选）

设置 Secret `UNLOCK_PIN` 后，"Patch RustDesk custom config" 步骤会在 `Config2::load()`
（`libs/hbb_common/src/config.rs`）中注入该明文 PIN：

- 仅当现有 `unlock_pin` **为空**时写入，不会覆盖用户后来修改的 PIN。
- Secret 通过 `option_env!("UNLOCK_PIN")` 在编译期读取，仓库与构建日志都不含明文值；
  但明文会编译进客户端二进制，该 PIN 仅用于锁定设置页 UI，属防误操作级别而非安全边界。
- 客户端首次启动会把明文用**本机 UUID 派生密钥**加密并回写 `RustDesk2.toml`，
  因此密文不跨机器分发，每台机器各自加密，避免密钥绑定导致的解密失败。
- 内置选项 `disable-unlock-pin = "Y"` 会使该 PIN 失效。
- 未设置 Secret 时注入块不生效，客户端保持无 PIN（不会导致构建失败）。

## 构建

工作流为**手动触发**（`workflow_dispatch`），推送到 `main` 不会自动构建。
需要新客户端时，在仓库 **Actions** 页手动运行 `Build RustDesk Windows`；
工作流包含两个作业（`generate-bridge` → `build-windows`），产物 `.exe`
在对应运行记录的工件（Artifacts）中下载。

## 参考文档

- [客户端修改参考.md](./客户端修改参考.md)：客户端全部源码定制项（服务器/账号、界面精简、首页 Logo、默认设置、安全密码）的整理说明。
