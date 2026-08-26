# Woow Tailscale — Home Assistant Add-on Repository

[![Add repository to Home Assistant](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2FWOOWTECH%2FWoow_ha_vpn_tailscale_package)

以 Home Assistant add-on 形式打包官方 [Tailscale](https://tailscale.com/) client；同一支 add-on 也能接你自架的 [Headscale](https://headscale.net/) control plane（透過 `login_server` 選項切換）。HA 主機加入 tailnet 後可當 subnet router / exit node，也能用 Taildrop / Taildrive 分享檔案，或用 Serve / Funnel 分享 HA 本體到外網。

配對用途：搭配 [`Woow_ha_vpn_headscale_package`](https://github.com/WOOWTECH/Woow_ha_vpn_headscale_package) 就能整組自架 VPN（HA 同時是 control plane 與 client）。

## 加入商店（HAOS / HA Supervised）

1. 按上方 badge，或到 **Settings → Add-ons → Add-on Store → ⋮ → Repositories** 貼上：

   ```
   https://github.com/WOOWTECH/Woow_ha_vpn_tailscale_package
   ```

2. 商店會出現 **Woow Tailscale**，點 **INSTALL**。第一版不提供預建 image，由 Supervisor 在本地 build（首次約 3–5 分鐘）。
3. 安裝後先看 **Configuration** 分頁設定 `login_server`（自架 Headscale 用），再 **START**。完整使用說明見 [`tailscale/DOCS.md`](./tailscale/DOCS.md)。

## 內含 add-on

| Slug | 名稱 | 說明 |
|------|------|------|
| [`woow-tailscale`](./tailscale) | Woow Tailscale | 官方 Tailscale client（Tailscale / Headscale 兩用），支援 subnet router、exit node、Taildrop / Taildrive、Serve / Funnel |

支援架構：`amd64`、`aarch64`。

## 架構重點

- **Ingress 為預設管理介面**：從 HA sidebar「Woow Tailscale」點開即進 Tailscale Web UI，無需自建 host port。
- **`login_server` 熱切換**：填空 = 官方 Tailscale；填 `http://<HA_IP>:28080` 或自架 URL = 走 Headscale。切換時自動 `tailscale logout` + 重連，`init-login-server-migration` s6 service 保證乾淨。
- **NET_ADMIN + /dev/net/tun**：走 kernel networking 才有完整 subnet router / exit node 能力；`userspace_networking` 可關閉。
- **Taildrive 精細掛載**：所有 HA folder（`addons`、`addon_configs`、`backup`、`config`、`media`、`share`、`ssl`）皆可個別 opt-in 分享到 tailnet。
- **備份即還原**：所有 tailnet state（machine key、prefs）由 `tailscaled` 自維護於 add-on data，HA backup 還原後直接復活。

## 已知限制（第一版）

- `share_homeassistant: funnel` 需要 tailnet 開通 Funnel（Tailscale 官方需在 admin console 打開；Headscale 尚不支援 Funnel）。
- MagicDNS ingress proxy 對非標準 DNS suffix 未做特殊處理，走預設 tailnet 設定。
- 第一版與 [`Woow_ha_vpn_headscale_package`](https://github.com/WOOWTECH/Woow_ha_vpn_headscale_package) 的 control plane 相依 → 對外曝露 control plane 需另設反代或 ngrok（詳見 headscale add-on 的 DOCS）。

## 對照文件與出處

- Add-on 使用說明：[`tailscale/DOCS.md`](./tailscale/DOCS.md)（zh-TW）
- Upstream Tailscale：[`tailscale/tailscale`](https://github.com/tailscale/tailscale)（BSD-3-Clause）
- 打包基礎：[`hassio-addons/addon-tailscale`](https://github.com/hassio-addons/addon-tailscale)（Apache-2.0）— s6-overlay service 樹、AppArmor profile、magicdns proxy 與 login-server reconcile 腳本
- 姊妹倉：[`WOOWTECH/Woow_ha_vpn_headscale_package`](https://github.com/WOOWTECH/Woow_ha_vpn_headscale_package)（自架 Headscale + Headplane）
- 源出商店：[`WOOWTECH/Woow_HA_App_Store`](https://github.com/WOOWTECH/Woow_HA_App_Store) 的 `woow-tailscale/`（本倉是同步鏡像，非取代）

## Licence

Add-on package: MIT — see `LICENSE`.
Tailscale client: BSD-3-Clause (tailscale/tailscale). Add-on packaging基底: Apache-2.0 (hassio-addons/addon-tailscale).
