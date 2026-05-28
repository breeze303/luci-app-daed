# 2026-05-28 更改说明

## 1. LuCI 日志页面改动

文件：`luci-app-daed/luasrc/view/daed/daed_log.htm`

### 代码级变更点

- 把原来直接轮询日志的逻辑整理为 `fetch_log(force)`：
  - `force !== true` 且自动刷新关闭时，直接返回，不请求日志接口。
  - 请求仍然使用原来的 LuCI URL：`admin/services/daed/get_log`。
  - XHR 回调里再次判断自动刷新状态，避免关闭自动刷新后，已经发出的旧请求回来又写入 textarea。
  - 当 `force === true` 时跳过自动刷新状态限制，用于“刷新日志”按钮和重新开启自动刷新后的立即刷新。
- 保留 `scrolled` 变量，只在首次加载日志时自动滚动到底部，避免每次刷新强制拉动滚动条。
- 用 `window.setInterval(function() { fetch_log(false); }, 2000);` 替代直接把 `get_log` URL 交给 `XHR.poll`，这样关闭自动刷新时可以真正阻止继续发起日志请求。
- 页面加载后执行一次 `fetch_log(true)`，保持进入页面立即显示日志。
- 新增 `toggle_auto_refresh(btn)`：
  - 切换 `autoRefreshEnabled`。
  - 根据状态切换按钮文字：`Disable auto refresh` / `Enable auto refresh`。
  - 从关闭切回开启时立即执行 `fetch_log(true)`。
- 新增 `refresh_log(btn)`：
  - 只做一件事：调用 `fetch_log(true)`。
  - 不修改自动刷新开关状态。
- 在按钮区新增两个按钮：
  - `id="refresh_log_button"`，点击执行 `refresh_log(this)`。
  - `id="auto_refresh_button"`，点击执行 `toggle_auto_refresh(this)`。

### 新增自动刷新开关

- 新增页面局部变量：`autoRefreshEnabled = true`
- 默认进入页面后仍然自动刷新日志。
- 新增按钮：`关闭自动刷新` / `开启自动刷新`
- 点击关闭后，定时刷新不会继续请求日志接口。
- 点击开启后，会立即刷新一次日志，并继续按 2 秒间隔自动刷新。
- 自动刷新状态只在当前页面有效，不写入 UCI、cookie、localStorage 或后端配置。

### 新增手动刷新按钮

- 新增函数：`refresh_log(btn)`
- 新增按钮：`刷新日志`
- 点击后调用 `fetch_log(true)`，立即拉取一次日志。
- 即使自动刷新已关闭，手动刷新按钮仍可使用。

### 保留原有行为

- `清空日志` 按钮保留。
- 日志接口路径不变。
- 自动刷新间隔仍为 2 秒。
- 没有新增后端 controller、UCI 配置或 daemon 配置。

## 2. 中文翻译改动

源文件：`luci-app-daed/po/zh_Hans/daed.po`

### 代码级变更点

- 在原有 `Clear logs` 翻译附近新增日志按钮相关翻译，保持同一功能区域集中维护。
- 新增 `Refresh logs` 翻译，用于手动刷新按钮。
- 新增 `Disable auto refresh` / `Enable auto refresh` 翻译，用于自动刷新开关按钮的两个状态。
- 没有做无关翻译清理，也没有改其它文案。

新增翻译：

```text
Refresh logs -> 刷新日志
Disable auto refresh -> 关闭自动刷新
Enable auto refresh -> 开启自动刷新
```

上传到路由器前已用 `po2lmo` 编译为：

```text
/usr/lib/lua/luci/i18n/daed.zh-cn.lmo
```

## 3. 已上传到 OpenWrt 的 LuCI 文件

目标路由器：`192.168.3.15`

上传文件：

```text
luci-app-daed/luasrc/view/daed/daed_log.htm
-> /usr/lib/lua/luci/view/daed/daed_log.htm

/tmp/opencode/daed.zh-cn.lmo
-> /usr/lib/lua/luci/i18n/daed.zh-cn.lmo
```

上传后执行过：

```sh
rm -rf /tmp/luci-indexcache /tmp/luci-modulecache
/etc/init.d/uhttpd restart
```

路由器上已验证：

- `refresh_log_button` 存在
- `function refresh_log` 存在
- `Refresh logs` 存在
- `auto_refresh_button` 存在
- `toggle_auto_refresh` 存在

## 4. 已上传到 OpenWrt 的脚本文件

### 这些脚本本地相对原仓库的主要变更

#### `daed/files/daed.init`

- 启动前确保目录存在：
  - `/var/log/daed`
  - `/etc/daed`
- 设置 daemon 运行环境变量：
  - `DAE_LOCATION_ASSET=/usr/share/v2ray`
- 使用 procd 启动 `/usr/bin/daed run`，并传入：
  - `--config /etc/daed/`
  - `--listen <listen_addr>`
  - `--logfile /var/log/daed/daed.log`
  - `--logfile-maxbackups <log_maxbackups>`
  - `--logfile-maxsize <log_maxsize>`
- 保留日志文件，不在 stop 时删除 `/var/log/daed/daed.log`，方便排错。
- stop 时只尝试清理 daed 可能创建的 network namespace 挂载点：`/run/netns/daens`。

#### `luci-app-daed/root/etc/init.d/luci_daed`

- 不再运行时 `sed` 修改 `/etc/init.d/daed`，避免 LuCI 侧脚本篡改 daemon init 脚本。
- 根据 `daed.config.enabled` 调整 dnsmasq 的 `localuse`：
  - daed 未启用时：设置 `dhcp.@dnsmasq[0].localuse=1`
  - daed 启用时：设置 `dhcp.@dnsmasq[0].localuse=0`
- `dellocaluse()` 会尝试从 WAN 接口读取 DNS；读取不到时使用兜底 DNS：
  - `119.29.29.29`
  - `180.76.76.76`
  - `223.5.5.5`
- 写 `/etc/resolv.conf` 时先写临时文件，再覆盖目标文件，避免中途生成半截内容。
- cron 相关函数保留，但当前 start/stop 中仍是注释状态，没有启用订阅 cron 修改。

#### `luci-app-daed/root/etc/hotplug.d/iface/98-daed`

- 只处理 `ACTION=ifup`。
- 优先使用 hotplug 提供的 `$DEVICE`，没有时再从 `logread` 中尝试取最近 link-up 设备名。
- 使用 `ip link show dev "$DEVICE_NAME"` 判断设备是否存在。
- 只对 `link/ether` 以太网设备触发处理。
- 使用目录锁 `/tmp/lock/daed_hotplug_lock` 防止短时间内重复触发多个重启任务。
- 后台等待 60 秒后执行 `/etc/init.d/daed restart`。
- 锁会通过 `trap` 在后台任务结束时清理。

上传前已备份路由器原脚本到：

```text
/root/daed-script-backup-20260528-134510
```

上传文件：

```text
daed/files/daed.init
-> /etc/init.d/daed

luci-app-daed/root/etc/init.d/luci_daed
-> /etc/init.d/luci_daed

luci-app-daed/root/etc/hotplug.d/iface/98-daed
-> /etc/hotplug.d/iface/98-daed
```

上传后设置权限：

```sh
chmod 755 /etc/init.d/daed /etc/init.d/luci_daed /etc/hotplug.d/iface/98-daed
```

路由器上已验证：

```sh
sh -n /etc/init.d/daed
sh -n /etc/init.d/luci_daed
sh -n /etc/hotplug.d/iface/98-daed
```

结果：语法检查通过。

服务状态验证：

```text
/etc/init.d/daed enabled -> yes
/etc/init.d/daed status  -> running
```

注意：上传脚本后没有主动重启 `daed`，避免影响当前运行测试。

## 5. 本地静态验证

本地已执行并通过：

```sh
grep -q 'autoRefreshEnabled.*true' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q 'toggle_auto_refresh' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q 'auto_refresh_button' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q 'function refresh_log' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q 'refresh_log_button' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q 'Refresh logs' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q 'Refresh logs' luci-app-daed/po/zh_Hans/daed.po
grep -q '刷新日志' luci-app-daed/po/zh_Hans/daed.po
grep -q 'setInterval' luci-app-daed/luasrc/view/daed/daed_log.htm
grep -q '2000' luci-app-daed/luasrc/view/daed/daed_log.htm
! grep -q 'XHR.poll(2,.*get_log' luci-app-daed/luasrc/view/daed/daed_log.htm
```

脚本语法检查：

```sh
sh -n daed/files/daed.init
sh -n luci-app-daed/root/etc/init.d/luci_daed
sh -n luci-app-daed/root/etc/hotplug.d/iface/98-daed
```

结果：通过。

## 6. 测试建议

浏览器访问：

```text
http://192.168.3.15/cgi-bin/luci/admin/services/daed/log
```

建议测试：

1. 页面加载后日志默认自动刷新。
2. 点击 `关闭自动刷新` 后，日志不再自动自动更新。
3. 自动刷新关闭时，点击 `刷新日志`，日志应立即刷新一次。
4. 点击 `开启自动刷新` 后，日志应立即刷新一次，并恢复自动刷新。
5. `清空日志` 仍可正常使用。
