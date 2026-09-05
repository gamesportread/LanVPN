# 澜 VPN 2.3

中文 Android 多协议 VPN 客户端，支持 Android 7.0（API 24）及以上。

## 安装与使用

1. 安装 `LanVPN-2.3.0-debug.apk`。如系统阻止安装，请只为当前文件管理器临时允许“安装未知应用”。
2. 打开应用，使用相机扫码、二维码图片、配置文件、独立 Clash 入口或粘贴方式导入节点。
3. 检查导入预览。应用只在你点击“保存节点”后加密保存节点。
4. 选择一个节点，点击“连接 VPN”，同意 Android 的 VPN 授权。
5. 访问目标网络并查看收发流量。隧道启动只表示本机核心运行，不能代替服务器可用性测试。

如果连接后打不开网页，先在节点列表选中节点并点击“测试当前节点”。测试失败时会按超时、TLS、域名解析或服务器拒绝给出原因；测试成功后再连接 VPN。

本应用不附带服务器、免费节点或订阅。你仍需从自己的服务商或服务器管理员获得配置。

## 连接协议

- VLESS
- VMess AEAD（`alterId` 必须为 0）
- Trojan
- Shadowsocks AEAD / 2022 常见加密方式，不含插件
- WireGuard

Xray 节点支持 TCP、WebSocket、gRPC、HTTPUpgrade、XHTTP，支持 TLS 与 REALITY。具体扩展字段仍须与内置 Xray 版本兼容。

不存在能够可靠表示“所有 VPN 格式”的统一标准。当前版本会识别并明确拒绝 OpenVPN、SSR、Hysteria/2、TUIC、Shadowsocks 插件、私有加密二维码及无法无损转换的扩展参数，不会把它们伪装成导入成功。

## 导入与扫描

以下入口共用同一套解析与校验逻辑：

- 相机扫描二维码或其他包含文本的条码
- 从 JPG、PNG、WebP 图片识别一个或多个二维码
- 选择文本配置文件
- 粘贴文本
- 扫描或粘贴 HTTPS 订阅地址，再预览订阅返回的节点
- 扫描 `clash://install-config`、`clashmeta://install-config` 或 `clash-verge://install-config` 安装二维码

可解析的文本包括：

- `vless://`、`vmess://Base64(JSON)`、`trojan://`、`ss://`
- Base64 编码的多行节点列表
- Xray JSON 出站或完整配置中的可独立节点
- Clash YAML/JSON 的 `proxies` 节点
- sing-box JSON 的 `outbounds` 节点
- WireGuard 原始 `.conf`

从 Clash、sing-box 或 Xray 完整配置导入时，只提取可独立连接的节点。策略组、分流规则、DNS、入站、代理链和应用路由不会被复制，预览中会显示提示。单次最多导入 500 个节点，输入和订阅响应最大 2 MB。

订阅仅支持 HTTPS，最多跟随 3 次 HTTPS 跳转；这是一次性导入，当前版本不会后台自动更新订阅。大型配置通常无法直接放入单个二维码，应扫描服务商提供的 HTTPS 订阅二维码。

相机权限由主界面显式申请。若系统曾永久拒绝相机权限，应用会提供跳转到系统设置的入口。扫码器只识别二维码以提高弱光、倾斜和复杂背景下的识别速度；扫码窗口不再使用可能导致部分厂商相机预览黑屏的安全窗口标志。

## 安全与兼容说明

- 节点、密码和 WireGuard 私钥通过 Android Keystore 管理的 AES-GCM 密钥加密保存，并使用原子文件替换。
- 禁用应用备份和设备迁移，配置页及扫码页禁止系统截图。
- 订阅地址经过确认后才读取，禁止明文 HTTP、URL 内嵌用户名密码及跳转到 HTTP。
- Xray 模式使用全局 IPv4/IPv6 路由和 1280 MTU；Android 的 DNS 请求由 Xray DNS outbound 劫持，内置 DNS 通过节点访问 DoH（1.1.1.1 / 8.8.8.8）。VPN 应用自身不进入隧道，以免核心连接服务器时产生路由循环。
- WireGuard 保留原配置中的地址、DNS 和 `AllowedIPs` 路由。
- TLS 不接受已被当前 Xray 移除的无验证 `allowInsecure`。自签名证书节点需服务方提供 `pcs`（证书 SHA-256 指纹）或 `vcn` 校验名称；REALITY 节点需包含 `pbk` 与 `sni`。
- TLS 链接明确提供 `sni`/`servername` 时优先使用该值；若 VLESS/Clash WS 只有 `host`，应用会将它作为 SNI。手机系统时间必须准确。
- 没有开机自动连接、始终开启 VPN、按应用分流、断线后阻断全部网络或后台自动更新订阅。

## 构建

开发环境：JDK 21、Android SDK Platform 35、Build Tools 35.0.0、Gradle 8.11.1。

在 `local.properties` 中设置本机 Android SDK 路径：

```properties
sdk.dir=/your/path/to/Android/sdk
```

```sh
./gradlew :app:assembleDebug :app:testDebugUnitTest :app:lintDebug
```

`app/libs/libv2ray.aar` 固定为 AndroidLibXrayLite `v26.8.20`，SHA-256：

```text
670cf11d9d10a6bb6548ac4f593acfa4339155732f6f8de4d45923f30a74deed
```

其对应源码和内核源码归档位于 `third_party/source/`。WireGuard tunnel 由 Maven Central 解析，版本固定为 `1.0.20260102`。

`probe` 是仅用于自动化测试的独立 UID 应用，不会打包进澜 VPN APK。`app/src/androidTest` 的流量测试要求 Android 模拟器能够通过 `10.0.2.2` 访问 `test-fixtures` 中的本机 Xray 与 HTTP 服务。

交付 APK 使用 Android 开发测试签名，可直接侧载。正式商店或长期分发前，应创建并妥善保管发布签名，在目标品牌手机上完成兼容测试。

## 开源组件

- AndroidLibXrayLite v26.8.20 / Xray-core：多协议与 Android TUN 核心
- WireGuard Android tunnel 1.0.20260102：WireGuard 后端
- ZXing Android Embedded 4.3.0 / ZXing core 3.5.3：相机和图片二维码识别
- Gson 2.11.0：JSON
- SnakeYAML 2.3：安全 YAML 解析
- AndroidX 与 desugar_jdk_libs：Android 兼容组件

许可证和版本说明见 `THIRD-PARTY-NOTICES.md` 及 `app/src/main/assets/`。
