# HANDOFF — Woow_ha_vpn_tailscale_package（v1 initial extraction）

> 版本：v1（initial extraction，2026-08-26）
> 目的：把 `WOOWTECH/Woow_HA_App_Store/woow-tailscale/` 抽成獨立 add-on repository，對稱 `Woow_ha_vpn_headscale_package` 的結構。
> 語言：文件 zh-TW + English 技術詞；commit message 英文。

---

## 0. TL;DR

- 本倉 = `WOOWTECH/Woow_HA_App_Store` 於 SHA `5dbd054` 的 `woow-tailscale/` 目錄鏡像，重新編排為單一 add-on 商店結構。
- 子資料夾名從 `woow-tailscale/` 更名為 `tailscale/`（比照 headscale 版）；**add-on slug 保留 `woow-tailscale`**（`config.yaml` 內未變），已安裝的使用者升級不會斷。
- 對外可直接以 my.home-assistant.io badge 加入商店；第一版不提供預建 image，Supervisor 本地 build。
- **本倉與源倉需持續同步**：源倉（monorepo）仍會保留 `woow-tailscale/` 目錄；任何改動應同時 push 到兩邊，或後續改成單向鏡像（見 §3）。

---

## 1. 倉庫現狀

檔案樹：

```
Woow_ha_vpn_tailscale_package/
├── repository.yaml           # store metadata
├── README.md                 # 商店首頁 + my.home-assistant.io badge
├── LICENSE                   # MIT + hassio-addons/addon-tailscale Apache-2.0 attribution
├── HANDOFF.md                # 本文件
└── tailscale/                # 原 monorepo 的 woow-tailscale/ 內容
    ├── config.yaml           # slug=woow-tailscale, version=0.1.0, url→本倉
    ├── build.yaml
    ├── Dockerfile
    ├── apparmor.txt
    ├── DOCS.md
    ├── CHANGELOG.md
    ├── README.md
    ├── .README.j2
    ├── icon.png / logo.png
    ├── translations/en.yaml
    ├── tests/{test-reconcile-login-server.sh, test-s6-layout.sh}
    └── rootfs/               # s6-overlay 服務樹、nginx、bin scripts
```

抽出時對 monorepo 版本的**唯一** in-place 修改：
- `tailscale/config.yaml` 的 `url:` 從 `https://github.com/WOOWTECH/Woow_HA_App_Store` 改為 `https://github.com/WOOWTECH/Woow_ha_vpn_tailscale_package`（讓 HA 商店顯示正確倉庫 link）。

---

## 2. As-built 規格摘要（詳看 `tailscale/config.yaml` 與 `tailscale/DOCS.md`）

| 項目 | 值 |
|------|-----|
| slug | `woow-tailscale`（沿用，勿改） |
| version | `0.1.0` |
| arch | `aarch64`, `amd64`；`init: false`；`startup: services` |
| ingress | `true`，`ingress_port: 0`，`ingress_stream: true`；GUI 走 HA Ingress |
| network | `host_network: true`；`privileged: NET_ADMIN, NET_RAW, SYS_ADMIN`；`devices: /dev/net/tun` |
| ports | `41641/udp` (WireGuard) |
| options（節錄） | `login_server`（url，切 Tailscale/Headscale）、`accept_dns`、`accept_routes`、`advertise_exit_node`、`advertise_connector`、`advertise_routes`、`share_homeassistant`（disabled/serve/funnel）、`taildrive`（各 HA folder 個別 opt-in）、`taildrop` |
| map | `addons` / `all_addon_configs` / `backup` / `homeassistant_config→/config` / `media` / `share` / `ssl` 全部 rw（Taildrive 分享用） |

---

## 3. 兩倉同步策略（**兩邊都是可寫的鏡像**）

依 WoowTech split-repo 慣例（見 `Woow_docker-compose split repos` 相關倉），源 monorepo `Woow_HA_App_Store` 保留、獨立 repo 為對外入口；兩邊需持續內容一致。

### 選項 A：手動雙推（第一版採用）
在 monorepo 或本倉編輯後，把差異也 apply 到另一邊（同一 commit message，加 `Mirrors: <對面倉 sha>` 尾註）。
- 優點：兩倉都能收 PR。
- 缺點：容易忘記其中一邊。

### 選項 B：GitHub Actions 單向鏡像（未來可加）
在 monorepo 加 workflow：`push` 到 `main` 且路徑含 `woow-tailscale/**` 時，自動 `git subtree split --prefix=woow-tailscale` push 到本倉 `main`。
- 需要 PAT with `contents:write` 存放在 monorepo secret。
- 反向（本倉 → monorepo）較麻煩，通常只做單向。

決策待你確認：**要繼續選項 A（雙推）**還是**開始建選項 B 的 Action**。目前先按 A 走。

---

## 4. 首次驗收 checklist（在辦公室 HA 或任一台 HAOS 上）

1. **加入商店**：從 my.home-assistant.io badge 或手動貼 URL。
2. **INSTALL**：本地 build 應能過（若在其他 add-on 已成功 build 相同 base image，此步 <2 分鐘）。
3. **啟動**：`state: started`；Log 出現 `tailscaled` 已起、`Backend in state Running`。
4. **Ingress**：從 HA sidebar 點「Woow Tailscale」→ 顯示 Tailscale Web UI（尚未登入時給 login link）。
5. **登入官方 Tailscale**：`login_server` 留空 → 點 Log 上的 URL 通過 OAuth → `tailscale status` 拿到 100.x.y.z。
6. **切 Headscale**：改 `login_server: http://<HA_IP>:28080` → 重啟 add-on → 應看到 `login-server-migration` s6 service 執行 `logout` + 換 URL 重連（Headscale 側 pre-auth key 需先建）。
7. **Subnet routes**：`advertise_routes: [local_subnets]` + 到 Headplane 或 Tailscale admin console 核准路由 → 從另一台 tailnet 裝置 ping HA LAN 上的印表機或分享器。
8. **備份 / 還原**：HA backup 含此 add-on data → 還原後 machine key 不變、tailnet 上仍在。

---

## 5. 不要動 / 已知坑

- **`slug: woow-tailscale`**：改了 = 已裝的機器會被視為不同 add-on，資料不會自動遷移。
- **`arch` 順序**：`aarch64, amd64` 是刻意的，先 arm 讓 M-series Mac 的 buildx 快取先熱。
- **`privileged: SYS_ADMIN` + `/dev/net/tun`**：拿掉 = kernel networking 死，只剩 `userspace_networking` 模式（效能差 5-10 倍、無法當 subnet router）。
- **`login_server` 網域限制**：Tailscale 官方接受 `https://` only；Headscale 允許 `http://` 但 client 舊版可能吐 warning。改動時建議兩邊都測。
- **`hassio_api: true` + `all_addon_configs` map**：需要讀其他 add-on config 才能做 Taildrive 分享；不要收窄為 `homeassistant_config` only，會斷 Taildrive addons 分享。
- **改 add-on 原始碼要讓 HA 生效**：走「移除→重加 repository→install」重新 clone；只 `rebuild` 會用 stale store 快取。

---

## 6. 出處與環境事實

- 上游 add-on：`hassio-addons/addon-tailscale`（Apache-2.0）— s6-overlay 服務組裝、nginx template、magicdns proxy、reconcile-login-server 腳本
- 上游 Tailscale：`tailscale/tailscale`（BSD-3-Clause）
- 源出處：`WOOWTECH/Woow_HA_App_Store` 於 SHA `5dbd054` 的 `woow-tailscale/`
- 姊妹倉：`WOOWTECH/Woow_ha_vpn_headscale_package`（Headscale + Headplane control plane）
