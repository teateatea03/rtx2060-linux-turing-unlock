# RTX 2060 12GB：Linux Turing 解鎖實作紀錄

[English](README.md) · [實作方法](docs/REPRODUCE.zh-TW.md) · [驗證結果](docs/RESULTS.md) · [復原方法](docs/RECOVERY.zh-TW.md) · [上游與授權](docs/REFERENCES.md)

我們在一張 **RTX 2060 12GB（TU106、10de:1f03）** 重現了 [nlzy/turing-unlock](https://github.com/nlzy/turing-unlock)，並整理 Linux 開機整合、驗證與復原流程。**破解原理與核心程式來自上游作者；本專案是我們的重現紀錄。**

2026-10-25 已實際完成：單次解鎖試開機 → 回到原始 Ubuntu → 設定預設解鎖並再次開機驗證。沒有可靠的實際模型加速百分比，也沒有長期穩定性結論。

| 項目 | 本次實測 |
| --- | --- |
| 顯卡 | RTX 2060 12GB、TU106、sm_75、PCI 10de:1f03 |
| 系統／核心 | Ubuntu 26.04.1／7.0.0-38-generic |
| NVIDIA 驅動／功耗上限 | 595.99.02／184W |
| Secure Boot | 關閉 |
| 上游版本 | [1d379b724454d6fb7e8a7a8dd0ea359febc05ca3](https://github.com/nlzy/turing-unlock/commit/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3) |
| 解鎖回讀 | SS0=0x88888888、SS1=0x8、readout=0 |

## 我們採用的方法

將核心模組與六個韌體檔放進**獨立 dracut initramfs**，透過 pre-udev hook，在 NVIDIA 驅動綁定顯卡之前，只對指定 PCI 位址執行一次 insmod。使用獨立 GRUB 項目，保留原始 Ubuntu 核心與 initramfs。

先用 grub-reboot 選擇一次性的試驗項目，確認解鎖、驅動與數值測試，再回到原始 Ubuntu 驗證復原。最後把已成功的映像原封不動複製成正式檔名，將預設項目改為解鎖項目，並實際重新開機驗證。

這不是刷寫 vBIOS，也沒有安裝全域 DKMS 或修改 NVIDIA 驅動檔。每次開機重新套用；目前項目固定在已驗證核心。

## 實際證據與限制

- 試驗中五個已完成的矩陣測試，各有 64 個 CPU 參考樣本通過、整個輸出沒有 NaN/Inf，輸出雜湊與原始狀態相符。
- 正式開機的 2048³ FP16 輸入／FP32 累積，以及嚴格 FP32 測試通過。
- NVIDIA 驅動、SSH、圖形登入服務正常；這不等於已人工確認螢幕畫面。
- 試驗結束後，原始開機檔案的雜湊比對成功；正式安裝刻意改動的是預設選單與其生成設定。
- 原始速度對照受到模型程式自動載入約 10.5GiB 顯存影響，最後一個 8192³ FP32 案例記憶體不足。**不能拿這些數字宣稱解鎖提升幾成。**

[詳細結果與去識別化資料](docs/RESULTS.md)說明了測量方法、OOM 和測試範圍。尚未驗證冷斷電、睡眠恢復、長時間負載或實際模型吞吐量。

## 復原與風險

5 秒 GRUB 選單保留原始 Ubuntu，但這是**手動選擇**。當系統無法進入可操作狀態時，不能依賴 SSH 或 Linux 定時器復原。韌體停止失敗時，上游要求完全關機後再處理；更高運算負載也可能增加硬體損壞風險。

先閱讀[復原方法](docs/RECOVERY.zh-TW.md)，再依[實作紀錄](docs/REPRODUCE.zh-TW.md)調整自己的環境。本專案沒有通用一鍵安裝器，也不建議以修改 PCI ID 的方式略過不支援檢查。

只發布原創文件、原始碼範本及整理過的證據；不含 NVIDIA 韌體、核心模組、initramfs、ROM、內網位址或主機 UUID。[授權與致謝](docs/REFERENCES.md)。
