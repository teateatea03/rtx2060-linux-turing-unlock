# 復原方式與實際驗證邊界

## 本次證實可以做的事

在成功試驗後，我們回到原始 Ubuntu，確認正常核心/initramfs 啟動、沒有載入解鎖模組，NVIDIA/SSH/GDM 正常，原始開機檔案雜湊相符。之後正式安裝再做一次預設解鎖開機驗證。

這證明「成功試驗 → 正常開機復原」；不代表硬體卡死時必然可以遠端救回。

## 正式安裝後手動選擇原始 Ubuntu

開機時的 5 秒 GRUB 選單保留原始 Ubuntu 項目。選它可略過專用解鎖 initramfs。若 SSH 不通，需要能操作該選單的本機畫面或遠端主控台。

這個選單**不會偵測故障後自動切換**，也不會自動重新啟動卡死的機器。一般重開機／原始映像能否清除特定 GPU 故障狀態，仍取決於故障型態。

## 停用預設解鎖

本次已安裝機器保留以下私有管理工具（本 repository 沒有把它當通用工具發布）：

```sh
sudo python3 /var/lib/turing-unlock-2060/disable.py
```

它會檢查我們自己的 default drop-in 是否仍符合記錄的雜湊，封存它、重新生成 GRUB，並選擇原始 Ubuntu 作為一次性開機項目。它不會即時解除當前 GPU 狀態；之後需要重新開機。若工具回報 update-grub 或其他錯誤，不要把「已選 next_entry」誤認成永久停用成功，應核對生成設定再決定下一步。

在其他機器上，應根據自己的備份移除／停用「本次新增的預設選擇」，保留其他既有設定；update-grub 與 grub-script-check 都成功後，確認原始 Ubuntu 項目可選再重新開機。不要直接刪除原始 kernel/initramfs，不要照抄別台的 GRUB entry ID 或磁碟 UUID。

## 臨時計時器的能力

本次試驗和正式部署都曾準備五分鐘 Linux guard，條件是指定 kernel command line marker。所有 guard 都在成功確認後取消並封存，沒有保留永久啟用的 rollback timer。

- 試驗 timer 從未真的觸發，因此不能聲稱其端到端復原能力已測試。
- Linux guard 必須走到 systemd 能排程工作的狀態；韌體、核心或 initramfs 階段卡死不在其能力內。
- 指定一次性的 GRUB next_entry，只能影響下一次能夠進行的啟動，不能讓已卡死的系統自動重啟。
- 正式映像沿用試驗 marker；不要重新啟用舊 timer/service。

## 必須停止重試的情況

上游 README 明確說明：如果 GPU firmware 沒有停止，模組會保留 DMA buffers，應先完全關機，再處理下一次嘗試。不要連續 warm reboot 或反覆 insmod 當成一定有效的復原。

不要對正在顯示桌面或被模型使用的 GPU 熱卸載／重置。上游會拒絕已綁定驅動的顯卡，不能刪除檢查來強行執行。

上游警告解鎖後 Tensor 負載的功耗可能損壞硬體。我們維持 184W 上限，沒有因此證明硬體或供電絕對安全。

## 未驗證項目

冷斷電再啟動、GPU hang 救援、網路失聯救援、timer 實際觸發、長時間高負載、睡眠／休眠恢復，以及人工觀看實際桌面畫面。圖形登入服務 active 只是服務狀態。

[上游說明](https://github.com/nlzy/turing-unlock/blob/1d379b724454d6fb7e8a7a8dd0ea359febc05ca3/README.md) · [本次結果](RESULTS.md)
