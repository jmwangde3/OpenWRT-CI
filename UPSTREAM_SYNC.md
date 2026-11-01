# 上游更新后的固定改动

适用：`VIKINGYFY/OpenWRT-CI` 的上游 CI 更新。
`VIKINGYFY/immortalwrt` 的源码更新不需要合并进本仓库；QCA-ALL 构建时按分支 HEAD 拉取。

**不要使用 "Sync fork"**。用 `git fetch upstream main` + `git merge upstream/main`，冲突按本清单重写我们的部分。

## 一页清单

1. 工作流只保留 5 个：
   - `QCA-ALL.yml`
   - `Upstream-Watch.yml`
   - `WRT-CORE.yml`
   - `WRT-TEST.yml`
   - `Cache-Clean.yml`

   删除上游新加的 `AIROHA-ALL`、`MTK-ALL`、`QCB-ALL`、`OWRT-ALL`、`Auto-Clean` 以及任何其他 `*-ALL`。
   `Auto-Clean` 的 `releases_keep_latest: 0` / `workflows_keep_day: 0` 会删除全部 release/run，并触发其他 target，必须去掉。

2. `QCA-ALL.yml` 保持我们自己的两 job 版本：
   - 触发：`push` 到 main，paths 仅 `Config/**`、`Scripts/**`、`.github/scripts/**`；加 `workflow_dispatch`。
   - inputs：`PACKAGE`、`TEST`、`BRANCHES`（默认 `both`）。
   - 两个独立 job：`config-main`（`WRT_BRANCH: main`）、`config-owrt`（`WRT_BRANCH: owrt`）。
   - job 级 `if` 只能用 `github.event.inputs.BRANCHES`；不要在 job 级 `if` 用 `matrix` 或 `env`（GitHub 会报 422）。
   - `WRT_CONFIG: IPQ807X-WIFI-YES`，`WRT_SOURCE: VIKINGYFY/immortalwrt`，`WRT_REPO` 同上。
   - 默认值：`WRT_THEME: vantage`、`WRT_NAME: OWRT`、`WRT_SSID: OWRT`、`WRT_WORD: 12345678`、`WRT_IP: 192.168.10.1`、`WRT_PW: 无`。
   - 不要加 `workflow_run` / Auto-Clean 触发；上游 QCA-ALL 的 owrt-only 版本不能直接用。

3. `Upstream-Watch.yml` 必须保留：
   - 每天北京时间 21:00：`cron: '0 13 * * *'`；不要排 0:00。
   - `git ls-remote` 检查 `main` 和 `owrt`；首次只写基线不编译。
   - 状态存 `.upstream-state`，cache key `upstream-state-`。
   - 只有 SHA 变化才 `POST /actions/workflows/QCA-ALL.yml/dispatches`，`inputs.BRANCHES` 传 changed 分支列表。
   - 上游删除此文件时必须重写。

4. `Config/IPQ807X-WIFI-YES.txt` 只保留两台 stock：
```text
CONFIG_TARGET_qualcommax=y
CONFIG_TARGET_qualcommax_ipq807x=y
CONFIG_TARGET_DEVICE_qualcommax_ipq807x_DEVICE_redmi_ax6-stock=y
CONFIG_TARGET_DEVICE_qualcommax_ipq807x_DEVICE_xiaomi_ax3600-stock=y
```
   不要恢复非 stock、AX9000、Linksys、QNAP、CR1000A、Arista、Aliyun 等设备行。
   目标：release 里只出现 `xiaomi_ax3600-stock` 和 `redmi_ax6-stock` 的镜像。

5. `Scripts/Settings.sh` 必须保留：
   - 我们的 main-only NSS 块：
     - 修改 `package/network/config/firewall4/patches/002-default-enable-fullcone.patch`，把 `flow_offloading` / `flow_offloading_hw` 的默认 1 改成 0。
     - 生成 `files/etc/uci-defaults/99-nss-offload`：关闭 `flow_offloading`、`flow_offloading_hw`、`packet_steering`，仅 main。
     - 只在 `WRT_BRANCH == main` 时执行；owrt 不动卸载。
   - 上游的默认 SSID/密码逻辑要保留：写入 Router Tool 的 `option ssid` / `option key`。
   - 注意：上游现在用 `ssid '...'` 空格语法，只能匹配 Router Tool；`wifi-scripts/.../mac80211.uc` 用的是 `ssid='...'` 等号语法。若以后把 `WRT_SSID`/`WRT_WORD` 改成不是 `OWRT`/`12345678`，要把等号语法的 sed 补回来：
```bash
sed -i "s/ssid='.*'/ssid='$WRT_SSID'/g; s/key='.*'/key='$WRT_WORD'/g" "$WIFI_UC"
```
   - 不要因为上游删掉 NSS 块就跟着删。

6. `Scripts/Handles.sh`：跟上游即可。
   - 上游保留 vantage `#52`（进度条标签字体），保持它。
   - `#51` 的表单说明宽度已由上游主题删掉 `max-width`，不再需要；旧 sed 是空操作，不要加回。
   - 只有布局问题重新出现时才临时补，不列入必保项。

7. `Config/GENERAL.txt` 保留我们的包选择：
   - `luci-app-homeproxy=n`
   - `luci-app-passwall=y`
   - `luci-app-smartdns=y`
   - `fullconenat-sonic=y`
   - `samba4-server=n`
   - PassWall：`INCLUDE_Xray`、`INCLUDE_Haproxy`、`INCLUDE_Shadowsocks_Rust_Client/Server`、`INCLUDE_ShadowsocksR_Libev_Client/Server`、`INCLUDE_Simple_Obfs`、`INCLUDE_V2ray_Plugin`、`INCLUDE_V2ray_Geodata` 全部 `=n`。
   - 继续删 USB/ATA/NVMe/sound/storage/Samba/stuntman 相关包；上游新增的包不要整段照收。
   - 改 GENERAL 后要重新核对 `.config`/manifest，Kconfig 可能回选依赖。

8. `Scripts/Packages.sh`：
   - `vnt` feed 行保持删除（上游 967e4c77 已删）；不要把 `UPDATE_PACKAGE "vnt"` 加回来。
   - 其他上游包更新可以接受，但不要覆盖上面的 GENERAL 选择。

9. 构建/发布：
   - 只允许 QCA-ALL 出包；不要恢复 Auto-Clean 或其他 target。
   - push 触发默认编 main+owrt；Upstream-Watch 只编变化的分支。
   - 每次出包后检查 release 资产：只能有 `xiaomi_ax3600-stock`、`redmi_ax6-stock`（加 Config 文件），不能有其他设备。
   - 旧 release 手动清理；不再每天自动删。

## 合并后验证

```bash
cd OpenWRT-CI
bash -n Scripts/Settings.sh
bash -n Scripts/Packages.sh
python - <<'PY'
import glob, yaml
for f in glob.glob('.github/workflows/*.yml'):
    yaml.safe_load(open(f, encoding='utf-8'))
    print('OK', f)
PY
ls .github/workflows
grep -E 'DEVICE_qualcommax_ipq807x_DEVICE_' Config/IPQ807X-WIFI-YES.txt
grep -n 'workflow_run' .github/workflows/*.yml || true
grep -n '99-nss-offload\|theme-vantage' Scripts/Settings.sh Scripts/Handles.sh
```

预期：
- 工作流 5 个；没有 `workflow_run`；没有 Auto-Clean。
- `Config/IPQ807X-WIFI-YES.txt` 只有 2 条 DEVICE。
- `Settings.sh` 有 `99-nss-offload`；`Handles.sh` 有 `theme-vantage`。
- YAML/Shell 全部通过后，再 commit/push，并检查新的 QCA-ALL run 两个 job 和 release 资产。

## 上游常见破坏

| 上游动作 | 处理 |
|---|---|
| 删除 `Upstream-Watch` | 重写我们的 watcher |
| 新增 `Auto-Clean` + 多个 `*-ALL` | 全部删；只留 QCA-ALL |
| QCA-ALL 改成 Auto-Clean/workflow_run/owrt-only | 用我们的两 job 版本覆盖 |
| `Config/IPQ807X-WIFI-YES.txt` 恢复到全设备 | 收窄到两台 stock |
| Settings 删掉 main NSS 块 | 加回 |
| Handles 只剩 `#52` | 正常，跟上游；`#51` 已过时 |
| GENERAL 加回 USB/ATA/声音/Samba 等 | 按我们的包表裁剪 |
| Packages 加回 vnt | 再删 |
| 上游 source 更新 | 不合并仓库；QCA-ALL 构建时拉 branch HEAD |
