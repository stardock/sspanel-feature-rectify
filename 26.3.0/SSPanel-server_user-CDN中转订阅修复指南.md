# SSPanel-UIM 26.3.0：`custom_config.server_user`（CDN 中转地址）不生效的排查与修复

> 适用版本：SSPanel-UIM 26.3.0
> 涉及模块：`src/Services/Subscribe/*`
> 现象：节点 `custom_config` 里配置了 `server_user`，但订阅（vmess/clash/sing-box…）里客户端连接地址仍然是服务器地址

---

## 目录

- [1. 问题描述](#1-问题描述)
- [2. 根因分析](#2-根因分析)
- [3. 修复方案](#3-修复方案)
  - [3.1 新增统一取址方法](#31-新增统一取址方法)
  - [3.2 改动点清单](#32-改动点清单)
  - [3.3 刻意没改的地方](#33-刻意没改的地方)
- [4. 配置使用方式](#4-配置使用方式)
- [5. 验证步骤](#5-验证步骤)
- [6. 覆盖后仍不生效的排查顺序](#6-覆盖后仍不生效的排查顺序)
- [7. 附录：改动前后对比示例](#7-附录改动前后对比示例)

---

## 1. 问题描述

**环境信息**

| 项目 | 值 |
| --- | --- |
| 服务端 | SSPanel-UIM 26.3.0 |
| 需求 | 客户端连接走 CDN/中转地址，节点真实 IP 不暴露 |
| 已做配置 | 节点 `custom_config` 增加 `"server_user": "cdn.example.com"` |
| 预期行为 | 订阅中 vmess 的 `add`、clash 的 `server`、sing-box 的 `server` 等应为 CDN 域名 |
| 实际行为 | 上述字段仍是节点表 `nodes.server` 里的服务器地址 |

**结论先行**：这不是配置写法问题，而是 **26.3.0 中 `server_user` 这个 key 根本没有被任何代码读取**，属于功能缺失。

---

## 2. 根因分析

### 2.1 全仓库不存在 `server_user` 的读取逻辑

对 `src/` 与 `app/` 全量搜索 `server_user`：

```bash
grep -rn "server_user" src/ app/
# 输出：0 条
```

即该字段只是被当作 JSON 原样存进 `nodes.custom_config`，服务端从未消费它。

### 2.2 订阅模块把连接地址写死为 `$node->server`

逐文件定位（行号为原始 26.3.0 代码）：

```php
V2Ray.php:52            'add'     => $node_raw->server,   // ← 用户看到的 add
V2RayJson.php:30,54,83,137  'address' => $node_raw->server,
Clash.php:38,67,86,105,189,224      'server'  => $node_raw->server,
SingBox.php:34,58,76,106,164,213    'server'  => $node_raw->server,
SIP002.php:38           $host = (string) $node_raw->server;
SIP008.php:33           'server' => $node_raw->server,
Trojan.php:39           'trojan://' . $user->uuid . '@' . $node_raw->server ...
SS.php:29               ... '@' . $node_raw->server ...
Hysteria2.php:74         'server' => $node->server,
Vless.php:147           $host = (string) $node->server;
```

所有格式的"连接地址"来源单一且硬编码，没有任何回退/覆盖机制，因此改 `custom_config` 无效。

### 2.3 为什么不能直接改 `nodes.server`

`nodes.server` 同时承担下列职责，直接改会导致节点不可用、流量统计异常：

- `src/Controllers/WebAPI/NodeController.php:38` —— 下发给节点端的通信地址
- `src/Models/Node.php:124-132` —— `ipv4` / `ipv6` 判定与 DNS 解析

因此正确做法是：**只在"生成客户端订阅"这一层覆盖地址**。

---

## 3. 修复方案

思路：在 `Base` 里新增一个统一取址方法，`custom_config.server_user` 优先、为空回退 `nodes.server`；再把上面所有硬编码引用改为调用该方法。这样所有订阅格式行为一致，无需逐格式打补丁，后续新增格式也只需调用同一方法。

### 3.1 新增统一取址方法

**文件：`src/Services/Subscribe/Base.php`**

```php
<?php

declare(strict_types=1);

namespace App\Services\Subscribe;

abstract class Base
{
    abstract public function getContent($user): string;

    /**
     * 客户端实际连接的地址（vmess 的 add、clash 的 server、sing-box 的 server 等）。
     * 优先取节点 custom_config 里的 server_user（CDN / 中转地址），没填则回退到节点表的 server。
     */
    public static function host(object $node, ?array $config = null): string
    {
        if ($config === null) {
            $config = json_decode((string) ($node->custom_config ?? ''), true);
        }

        $candidate = is_array($config) ? ($config['server_user'] ?? '') : '';
        if (is_string($candidate) && trim($candidate) !== '') {
            // 容错：允许误填 https://cdn.example.com/ 之类，只取 host[:port]
            $host = (string) preg_replace('#^[a-z][a-z0-9+.\-]*://#i', '', trim($candidate));
            $host = explode('/', $host)[0];
            if ($host !== '') {
                return $host;
            }
        }

        return (string) $node->server;
    }

    /** Isolate invalid VLESS nodes without logging credentials or private keys. */
    protected function vlessClient(object $node, mixed $config): ?array
    {
        if (! is_array($config)) {
            return null;
        }

        try {
            // server 只作为 sni / server_name 的兜底，连接地址在 Vless::uri() 里单独取
            return Vless::client($config, (string) $node->server);
        } catch (\InvalidArgumentException | \RuntimeException $error) {
            error_log('Skipped invalid VLESS subscription node id=' . (int) $node->id);

            return null;
        }
    }
}
```

设计要点：

1. **容错**：管理员误填 `https://cdn.example.com/` 也能正确提取 `cdn.example.com`。
2. **向后兼容**：`server_user` 为空 / `custom_config` 非法 JSON / 为 `null` 时，全部回退到 `$node->server`，行为与升级前完全一致。
3. **无需额外解码**：调用方已解码时传入 `$config` 复用；未解码（如 `SS.php`）时方法内部自行解码。

### 3.2 改动点清单

共 **12 个文件、24 处**。

| 文件 | 改动 |
| --- | --- |
| `Base.php` | 新增 `host()` 方法 |
| `V2Ray.php` | `'add' => self::host($node_raw, $node_custom_config)` |
| `Clash.php` | 6 处 `'server' => self::host(...)`（ss / ss2022 / tuic / vless / vmess / trojan） |
| `SingBox.php` | 6 处 `'server' => self::host(...)` |
| `V2RayJson.php` | 4 处 `'address' => self::host(...)` |
| `SIP002.php` | `$host = self::host($node_raw, $node_custom_config);` |
| `SS.php` | ss 链接 host 改为 `self::host($node_raw)` |
| `SIP008.php` | `'server' => self::host(...)` |
| `Trojan.php` | trojan 链接 host |
| `Hysteria2.php` | `config()` 中 `'server' => self::host($node, $custom)` |
| `Vless.php` | `uri()` 中 `$host = Base::host($node, $config);` |

核心改法示例（其余文件同理，仅替换取值表达式）：

```diff
-                    'add' => $node_raw->server,
+                    'add' => self::host($node_raw, $node_custom_config),
```

```diff
-        $host = (string) $node->server;
+        $host = Base::host($node, $config);
```

### 3.3 刻意没改的地方

| 位置 | 保持原样的原因 |
| --- | --- |
| `Vless::client()` 的 `server_name` 兜底 | SNI 应为真实域名，用 CDN 中转域名通常会 TLS 握手失败；需要自定义 SNI 请配 `custom_config.host` |
| `Hysteria2::config()` 的 `'sni'` 兜底 | 同上 |
| `V2RayJson.php:74,127` 的 `?? $node_raw->server` | 属于 TLS `server_name` 兜底，不是连接地址 |
| `src/Controllers/WebAPI/NodeController.php:38` | 节点端通信地址，必须为真实地址 |
| `src/Models/Node.php:124-132` | `ipv4`/`ipv6` 判定与 DNS 解析，基于真实地址 |

---

## 4. 配置使用方式

节点编辑页的 `custom_config` 是 JSON 编辑器（`resources/views/admin/node/edit.tpl:84`），直接加 **顶层**字段：

```json
{
  "server_user": "cdn.example.com",
  "host": "real.example.com",
  "network": "ws",
  "path": "/ws",
  "offset_port_user": 443
}
```

> 注意：`server_user` 必须是 `custom_config` 的**顶层 key**，不要嵌在 `hysteria2` 之类的子对象里。

各字段职责（CDN 中转典型组合）：

| 字段 | 含义 | 示例 |
| --- | --- | --- |
| `server_user` | 客户端连接地址（CDN 接入域名） | `cdn.example.com` |
| `host` | 回源真实域名，用于 **SNI + Host 头** | `real.example.com` |
| `offset_port_user` | 客户端连接端口（CDN 端口） | `443` |
| `network` | 传输方式，CDN 一般需 `ws` | `ws` |
| `path` | ws 路径 | `/ws` |
| `security` | 传输层安全 | `tls` |

---

## 5. 验证步骤

### 5.1 语法检查

```bash
php -l src/Services/Subscribe/Base.php
# 逐个检查改动过的 12 个文件
```

### 5.2 订阅内容检查

| 订阅格式 | 检查命令 | 期望 |
| --- | --- | --- |
| vmess / v2ray | `echo '<vmess串>' \| cut -d'/' -f3- \| base64 -d` | `"add"` 为 CDN 域名 |
| clash | `grep -E 'server:\|servername:'` | `server` 为 CDN，`servername` 仍为真实域名 |
| sing-box | `jq '.outbounds[].server'` | CDN 域名 |
| vless / trojan / hysteria2 | 查看链接 `@` 之后部分 | CDN 域名 |
| ss (SIP002 / SIP008) | 解码或看 `server` 字段 | CDN 域名 |

### 5.3 回归检查（确认没引入新问题）

- 未配置 `server_user` 的节点：地址仍为 `nodes.server`，订阅内容与升级前一致
- 节点在线状态、流量统计正常（说明 `nodes.server` 未被误改）
- `custom_config` 为 `{}` 或非法 JSON 的节点：不报错、正常回退

---

## 6. 覆盖后仍不生效的排查顺序

1. **`custom_config` 是否合法 JSON**：`json_decode` 失败会静默回退，导致"看起来改了但没用"
2. **字段是否在顶层**：检查是否误嵌进 `hysteria2` / `reality-opts` 等子对象
3. **节点类型是否对得上**：确认测试的订阅格式与节点 `sort`（0=SS，1=SS2022，11=VMess/VLESS，14=Trojan，15=Hysteria2）匹配
4. **缓存是否未清**：OPcache、`storage/` 下的缓存文件、`php-fpm` 是否重启
5. **是否覆盖到了正确路径**：`src/Services/Subscribe/`，注意目录大小写与部署目录是否一致

---

## 7. 附录：改动前后对比示例

### vmess 分享链接（`V2Ray.php`）

**改动前**

```php
$v2rayn_array = [
    'v'    => '2',
    'ps'   => $node_raw->name,
    'add'  => $node_raw->server,
    'port' => $v2_port,
    // ...
];
```

**改动后**

```php
$v2rayn_array = [
    'v'    => '2',
    'ps'   => $node_raw->name,
    'add'  => self::host($node_raw, $node_custom_config),
    'port' => $v2_port,
    // ...
];
```

### Clash（`Clash.php`）

**改动前**

```php
$node = [
    'name'   => $node_raw->name,
    'type'   => 'ss',
    'server' => $node_raw->server,
    'port'   => (int) $user->port,
    // ...
];
```

**改动后**

```php
$node = [
    'name'   => $node_raw->name,
    'type'   => 'ss',
    'server' => self::host($node_raw, $node_custom_config),
    'port'   => (int) $user->port,
    // ...
];
```

### 修复效果总结

| 维度 | 修复前 | 修复后 |
| --- | --- | --- |
| 连接地址来源 | 只能 `nodes.server` | `custom_config.server_user` 优先，回退 `nodes.server` |
| 支持范围 | —— | vmess / clash / sing-box / v2rayN-json / ss / trojan / hysteria2 / vless 全覆盖 |
| 节点端通信 | 正常 | 不变（继续用 `nodes.server`） |
| 未配置节点 | 正常 | 行为不变（自动回退） |
