# SFA（去掉 NO_RETRY 的 fork）

> 本分支 `no-retry-1.14.4` 基于上游 [SagerNet/sing-box-for-android](https://github.com/SagerNet/sing-box-for-android) 的提交
> [`f87c2eb`](https://github.com/SagerNet/sing-box-for-android/commit/f87c2ebdb7e34754be751bb5e58926db2fed8f8c)（Bump version 1.14.4），
> 也就是 sing-box 正式版 [v1.14.4](https://github.com/SagerNet/sing-box/tree/v1.14.4/clients) 中 `clients/android` 子模块固定的版本。
> 这是非官方修改版，与 SagerNet 无关。

## 为什么 fork

SFA 的 `local` DNS 服务器（`LocalResolver`）通过 Android 的 `DnsResolver.rawQuery` / `query` 把查询交给系统解析器，并且一直带着 `DnsResolver.FLAG_NO_RETRY`。

在 AOSP 的 DnsResolver 里（`res_send.cpp`），带 `ANDROID_RESOLV_NO_RETRY` 且可用上游多于 1 个时，**不是按顺序问第一个服务器，而是按查询 ID 取模随机选一个**，并且只发一次。

HyperOS / MIUI 会在 `system_server` 里（`miui-connectivity-service.jar` 的 `ConnectivityServiceImpl.addDnsIfRequired`）给非 VPN 网络追加 DNS：只要网络下发的 DNS 不超过 2 个，就追加 `114.114.114.114`（有 IPv6 DNS 时还追加 `240c::6666`）。而且 `libnetd_resolv.so` 的 `isTrustMIUIAddDns` 会保证这两个地址不会因为统计差被跳过。

结果是：在校园网这类依赖内网 DNS 的 Wi‑Fi 上，上游列表变成 `[内网 DNS, 114.114.114.114]`，SFA 约一半的查询会落到 114 上，内网域名被解析成公网地址，并写进该网络的系统 DNS 缓存。

实测（Wi‑Fi 上游为 `[172.18.0.11, 114.114.114.114]`，同一内网域名，查询 20 次）：

| 标志 | 来自内网 DNS | 来自 114 |
|---|---|---|
| 不带 NO_RETRY | 20 | 0 |
| 带 NO_RETRY | 9 | 11 |

去掉 `FLAG_NO_RETRY` 后，系统解析器恢复默认行为：按列表顺序询问，只有前面的服务器失败或超时才轮到后面的 114。

## 改动

只有一处：`app/src/main/java/io/nekohasekai/sfa/bg/LocalResolver.kt` 中三次 `DnsResolver` 调用的 flags，从 `DnsResolver.FLAG_NO_RETRY` 改为 `0`：

- `exchange()` 中的 `rawQuery`
- `lookup()` 中指定类型的 `query`
- `lookup()` 中不指定类型的 `query`

副作用：上游服务器不通时，系统解析器会重试并依次尝试下一个，单次查询可能变慢（受系统的超时和重试次数控制）。

包名仍是 `io.nekohasekai.sfa`。自行签名的 APK 与官方版签名不同，需要先卸载官方版（数据会丢失，请先导出配置）。

## 编译

以下流程整理自 sing-box v1.14.4 的 `Makefile` 和 `.github/workflows/build.yml`（`build_android_library` / `build_android` 两个 job）。SFA 仓库本身不包含 CI，也不包含 libbox。

### 1. 环境

- Go `1.26.10`（`version.properties` 中的 `GO_VERSION`）
- JDK 17（CI 使用 Temurin 17）
- Android SDK（compileSdk 37）、NDK r28（`28.0.13004108`），设置 `ANDROID_NDK_HOME`

### 2. 编译 libbox.aar（在 sing-box 主仓库中）

```sh
git clone -b v1.14.4 https://github.com/SagerNet/sing-box
cd sing-box
make lib_install                       # 安装 sagernet/gomobile、gobind v0.1.13
export PATH="$PATH:$(go env GOPATH)/bin"
make lib_android                       # 即 go run ./cmd/internal/build_libbox -target android
# 生成 libbox.aar 与 libbox-legacy.aar
```

CI 是按架构（`-platform android/arm64` 等）分别编译，再用 `go run ./cmd/internal/merge_aar` 合并。本地直接 `make lib_android` 会一次编出全部架构。

### 3. 编译 APK（在本仓库中）

```sh
mkdir -p app/libs
cp /path/to/sing-box/libbox.aar /path/to/sing-box/libbox-legacy.aar app/libs/
./gradlew :app:assembleOtherRelease        # Android 7+，输出 app/build/outputs/apk/other/release/
./gradlew :app:assembleOtherLegacyRelease  # Android 5/6 版本
```

release 构建需要签名：把 keystore 放到 `app/release.keystore`，并在 `local.properties` 中写入
`KEYSTORE_PASS`、`ALIAS_NAME`、`ALIAS_PASS`（或把整个 properties 文件 base64 后放进环境变量 `LOCAL_PROPERTIES`，CI 就是这么做的）。
不想配置签名可以用 `./gradlew :app:assembleOtherDebug`，生成调试签名的 APK。

上游 CI 还会运行 `go run ./cmd/internal/update_android_version --ci` 根据 git 标签改写 `version.properties`；本分支已经是 `1.14.4`（versionCode 746），无需执行。

---

# SFA

Experimental Android client for sing-box, the universal proxy platform.

## Documentation

https://sing-box.sagernet.org/installation/clients/sfa/

## License

```
Copyright (C) 2022 by nekohasekai <contact-sagernet@sekai.icu>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <http://www.gnu.org/licenses/>.

In addition, no derivative work may use the name or imply association
with this application without prior consent.
```

Under the license, that forks of the app are not allowed to be listed on F-Droid or other app stores
under the original name.
