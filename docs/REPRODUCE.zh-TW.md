# 我們實際使用的重現與安裝流程

這是 2026-10-25 的操作紀錄。命令範例需要依自己的磁碟、GRUB、PCI 位址與核心調整；沒有把這些範例當作通用安裝器在其他機器測過。所有 GPU 操作都在目標 Linux 主機執行。

## 1. 確認環境、備份與可操作的復原入口

本次組合是 Ubuntu 26.04.1、核心 7.0.0-38-generic、NVIDIA 595.99.02、dracut、GRUB、Secure Boot 關閉、RTX 2060 12GB 10de:1f03，功耗上限維持 184W。

唯讀確認：

```sh
uname -r
lspci -Dnn
nvidia-smi
mokutil --sb-state
findmnt -rn -o TARGET,SOURCE,FSTYPE
```

檢查機器實際使用的 initramfs 產生器，以及 NVIDIA 是否被更早的 hook 強制載入。pre-udev 的先後順序只在本次映像得到驗證，不能推論所有 dracut 設定都相同。

在只供本次作業使用的本機目錄備份並計算雜湊：/etc/default/grub、/etc/grub.d/40_custom、/boot/grub/grub.cfg、/boot/grub/grubenv，以及目前核心與 initramfs。原有 custom.cfg、GRUB drop-in 若存在也要保存。先確認原始 Ubuntu 項目可以啟動，並準備能操作 GRUB 的本機畫面或遠端主控台。

不要公開這些完整備份：它們可能包含主機 UUID、檔案系統配置和其他私人資料。

## 2. 固定上游來源，只建置所需產物

以下從本紀錄 repository 的 clone 根目錄開始，在同一個 Bash 工作階段使用變數。指定尚不存在的專用工作目錄；不要把 ASSETS 指向其他資料。上游 make assets 會重建 ASSETS.src。

```sh
RECORD_ROOT="$PWD"
WORK="$RECORD_ROOT/trial-work-20261025"
KVER=7.0.0-38-generic
TARGET_BDF=YOUR_FULL_PCI_ADDRESS
mkdir "$WORK"
git clone https://github.com/nlzy/turing-unlock.git "$WORK/source"
git -C "$WORK/source" checkout --detach 1d379b724454d6fb7e8a7a8dd0ea359febc05ca3
git -C "$WORK/source" status --short
make -C "$WORK/source" assets INPUTS="$WORK/inputs" ASSETS="$WORK/assets"
make -C "$WORK/source/kernel/turing_unlock" -j2 KDIR="/lib/modules/$KVER/build"
```

TARGET_BDF 必須改成 lspci -Dnn 確認的完整位址，格式為 domain:bus:device.function；不要填 all。先確認實際 PCI ID 與本次條件相符。

本次使用既有工具鏈、核心標頭、dracut 與 CUDA，沒有安裝或升級套件。上游韌體產生器會下載兩份固定版本輸入並檢查 SHA-256，產出六個韌體：

- booter_load_tu102.bin
- gsp_boot_desc_tu102.bin
- gsp_boot_tu102.bin
- gsp_tu10x.bin
- sec2_bl_tu102.bin
- v67-tu102-610.43.03.bin

名稱含 tu102 不代表模組完全沒有 TU106 支援；該 commit 已列入 10de:1f03。上游作者首頁的主要實測板卡仍是 2080 Ti。不要因為外觀相似就自行增加未知 PCI ID。

## 3. 建立獨立 initramfs，保留原始映像

從 upstream 建好的模組與資產建立 staging：

```sh
STAGE="$WORK/staging"
install -d "$STAGE/usr/lib/turing-unlock-trial"
install -d "$STAGE/usr/lib/firmware/turing-unlock/610.43.03"
install -d "$STAGE/usr/lib/dracut/hooks/pre-udev"
install -m 600 "$WORK/source/kernel/turing_unlock/turing_unlock.ko" "$STAGE/usr/lib/turing-unlock-trial/"
install -m 644 "$WORK/assets/"*.bin "$STAGE/usr/lib/firmware/turing-unlock/610.43.03/"
sed "s/@TARGET_BDF@/$TARGET_BDF/g" "$RECORD_ROOT/examples/01-turing-unlock-trial.sh.in" > "$STAGE/usr/lib/dracut/hooks/pre-udev/01-turing-unlock-trial.sh"
chmod 755 "$STAGE/usr/lib/dracut/hooks/pre-udev/01-turing-unlock-trial.sh"
sh -n "$STAGE/usr/lib/dracut/hooks/pre-udev/01-turing-unlock-trial.sh"
```

先人工檢查替換結果、完整 BDF、模組與六個 .bin 雜湊。確定沒有殘留 @TARGET_BDF@，再建立新的映像檔名：

```sh
IMAGE="/boot/initrd.img-$KVER-turing-trial-20261025"
test ! -e "$IMAGE"
sudo dracut --hostonly --kver "$KVER" --include "$STAGE" / --install /usr/sbin/insmod "$IMAGE"
sudo lsinitrd "$IMAGE"
```

每一步都必須成功才繼續；沒有使用 --force。test 失敗時應停止，不要接著執行 dracut。確認映像內模組、hook、六個韌體與 staging 逐一雜湊相同；例如用 lsinitrd -f 取出指定檔案比較。只看清單出現名稱不夠。

範本是實際 hook 的 PCI 位址去識別化版本：
- 只有 kernel command line 包含 turing_unlock_trial=20261025 才執行。
- /run 下的 marker 限制同一次開機只嘗試一次。
- 直接 insmod 映像內的 .ko，傳入單一指定 targets。
- 把 insmod 狀態寫到 /run 與 kernel log。

**insmod 回傳 0 不代表解鎖成功。** 上游 init 函式可能在個別顯卡失敗後仍回傳成功，必須另外查每張卡的日誌與 selector 回讀。

## 4. 準備單次 GRUB 項目

備份後，從本機 /boot/grub/grub.cfg 複製「目前正常 Ubuntu」的完整 menuentry 到 custom.cfg；若 custom.cfg 已有內容，保留它。保留該機器的磁碟搜尋、root=、分割區、加密設定和必要參數，只調整：

| 欄位 | 本次試驗值 |
| --- | --- |
| menuentry ID | turing-unlock-trial-20261025 |
| kernel | 原始 vmlinuz-7.0.0-38-generic |
| initrd | 步驟 3 的獨立映像 |
| 附加參數 | turing_unlock_trial=20261025 turing_unlock.targets=自己的完整BDF |

不要把別人的磁碟 UUID 或範例 menuentry 當作自己的開機設定。確認生成的 grub.cfg 確實會載入 custom.cfg，並保持原始 Ubuntu 為預設。執行：

```sh
sudo grub-script-check /boot/grub/custom.cfg
sudo grub-script-check /boot/grub/grub.cfg
```

本次另準備了有條件的五分鐘 Linux rollback service/timer，但成功確認後、計時器觸發前已取消。它從未被驗證觸發，也無法救援未進入 systemd 的開機卡死；見[復原限制](RECOVERY.zh-TW.md)。本 repository 沒有提供會自動啟用 timer 的安裝腳本。

確認正在選的 ID 正確、有可操作的復原入口後，才執行本次使用的單次開機方式：

```sh
sudo grub-reboot turing-unlock-trial-20261025
sudo systemctl reboot --no-block
```

grub-reboot 的單次選擇不會替你再次重新啟動已卡死的機器。不同 GRUB/檔案系統對 next_entry 的支援也必須事先確認。

## 5. 分開驗證解鎖、驅動與數值

開機後查：

```sh
journalctl -k -b --no-pager | grep turing_unlock
nvidia-smi
systemctl is-active ssh gdm
```

本次預期 selector 是 SS0=0x88888888、SS1=0x8、readout=0，並有目標 PCI 顯卡的 turing_unlock: unlocked。這些是開機時的回讀，並非 GPU 後續重置或睡眠恢復後的即時證明。

[原始測試碼範本](../tools/gemm_check.cu.in)只將本機 BDF 換成 @TARGET_BDF@。本次完整試驗跑 2048/4096/8192；正式安裝只用 2048。以下在同一個 WORK 內產出小型 smoke 版本：

```sh
sed "s/@TARGET_BDF@/$TARGET_BDF/g; s/{2048,4096,8192}/{2048}/" "$RECORD_ROOT/tools/gemm_check.cu.in" > "$WORK/smoke.cu"
nvcc -ccbin g++-13 -O3 -arch=sm_75 "$WORK/smoke.cu" -lcublas -o "$WORK/smoke"
timeout 30 "$WORK/smoke"
```

編譯器版本與 nvcc 支援須依本機核對。每種模式各檢查 64 個 CPU 參考樣本、全輸出有限性與 rolling hash。這是限定輸入的 smoke，不是完整正確性證明；詳見[結果](RESULTS.md)。

原始完整 trial 的 wrapper 每兩秒觀察溫度／功耗，80°C 或 120 秒時終止負載；這不是硬體保護保證。本次發生過顯存 OOM，因此不要直接把完整尺寸負載當成無成本測試。要做性能對照，必須先建立相同的其他程式、顯存占用、功耗、時脈與輸入條件。

## 6. 實際回到原始 Ubuntu

本次在試驗完成後封存試驗 menuentry 與 timer/service，回到未包含解鎖 hook 的原始 Ubuntu，再實際重新開機。確認 command line 無試驗參數、解鎖模組不在該次開機載入，且 NVIDIA/SSH/GDM 正常。

對照備份：原始核心、initramfs、GRUB 設定及 grubenv 在試驗清理後全部雜湊相符。這證明成功試驗後的正常復原，沒有測試 GPU hang、斷網或電源故障的救援。

## 7. 正式預設開機設定

只有在上一步確認後，本次才把成功試驗映像原封不動複製成：

```text
/boot/initrd.img-7.0.0-38-generic-turing-unlock-2060
```

兩份映像 SHA-256 相同，正式項目 ID 為 turing-unlock-2060，仍使用原始 kernel。本次新增 /etc/default/grub.d/99-turing-unlock-2060.cfg：

```sh
GRUB_DEFAULT="turing-unlock-2060"
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=5
```

保留原始 Ubuntu 項目；update-grub 與 grub-script-check 成功後，**沒有設定 next_entry 覆寫**，實際重新開機以證明預設項目正確，再跑步驟 5 的 smoke。

正式映像保留 turing_unlock_trial=20261025 是為了沿用已驗證的 hook；它不是 rollback timer 仍在執行的證據。臨時部署 guard 已取消並封存，不能日後重新啟用含相同條件的試驗 timer，否則可能造成非預期重開機。

目前方案固定在 7.0.0-38-generic；核心／驅動更新、Secure Boot 或睡眠策略變更都需要重新檢查相容性。沒有自動更新解鎖映像的常駐服務。
