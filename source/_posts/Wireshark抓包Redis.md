---
title: 用 Wireshark 抓包 Redis
urlname: redis-wireshark
date: 2022-12-29 10:01:15
categories:
    - middlewire
    - redis
tags:
---


Redis 有自己的一套传输协议（RESP），因此想通过 Wireshark 抓包 Redis 得先安装插件，具体步骤如下：

1. 找到你的 Wireshark 安装目录下的 `init.lua` 所在的文件夹

例如我的是 MacOS，通过 DMG 安装的，是在 `/Applications/Wireshark.app/Contents/Resources/share/wireshark` 目录下

2. 把 [redis-wireshark.conf](https://github.com/jzwinck/redis-wireshark/blob/master/redis-wireshark.lua) 整个文件 copy 放在上述目录下

3. 编辑 `init.lua` 添加以下内容，加载第 2 步放进去的 lua 脚本

```lua
-- 其它内容...
if not running_superuser or run_user_scripts_when_superuser then
    dofile(DATA_DIR.."console.lua")
    dofile(DATA_DIR.."redis-wireshark.lua")     -- 这行是你要新加的，其它都是原有的
end
-- 其它内容...
```

4. 重启 Wireshark 或使用 UI 上的 `Analyse -> Reload Lua Plugins` 功能（CMD + SHIFT + L）重新加载插件

5. 启动抓包，用 `redis-cli` 连接你的 Redis，随便输入一些命令（我这里示例是 `get tac`），在 Wireshark filter 框框中输入 `redis` 或 `tcp.port == 6379` 过滤协议，看到以下数据说明成功了

![redis-wireshark-example](images/redis-wireshark/example.jpg)
