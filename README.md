minimal-wifi-stack-study
========================
mac80211 or cfg80211, nl80211

Use QEMU Rasp3 is enough for development environment

### Ported
- [nxp-mcuxpresso/wpa_supplicant-rtos: WPA Supplicant & HOSTAPD](https://github.com/nxp-mcuxpresso/wpa_supplicant-rtos)
- https://github.com/rsta2/circle/tree/master/addon/wlan/sample/hello_ap
- https://github.com/rsta2/circle/tree/master/addon/wlan
- https://github.com/rsta2/hostap
- https://github.com/rsta2/circle/blob/7a81e1b480fe7d19fc1fc695b89bff0a0848f44b/addon/wlan/bcm4343.h

### Drivers
- [**StreamUnlimited/broadcom-bcmdhd-4359**](https://github.com/StreamUnlimited/broadcom-bcmdhd-4359)
- [openwrt/mt76: mac80211 driver for MediaTek MT76x0e, MT76x2e, MT7603, MT7615, MT7628 and MT7688](https://github.com/openwrt/mt76)


### AI
```
Wi-Fi 晶片必須是 SoftMAC 晶片
• 如果是 FullMAC 晶片（如 ESP32、CYW43439），晶片韌體已經內建了 MAC 層，則完全不需要移植 mac80211。


將 [rsta2/circle](https://github.com/rsta2/circle) 專案中的 Wi-Fi 堆疊（WLAN Stack）移植到 MCU 是一個非常聰明且具可行性的思路！ [1] 
相較於前面提到龐大且難以割捨的 Linux mac80211 / cfg80211，[rsta2/circle](https://github.com/rsta2/circle) 是一個專為樹莓派（Raspberry Pi）開發的裸機（Bare-metal）C++ 環境。它的 Wi-Fi 驅動與協定疊本質上就是為了「沒有 Linux 核心的環境」而設計的，已經幫你完成了大量的去核心化（De-kernelization）工作。 [1, 2] 
然而，這個方案也有其特定的優勢與潛在的「坑」，以下為您進行深度評估：
## 💡 為什麼這個方案比移植 Linux 好？（優勢）

   1. 架構極度簡化：Circle 的 Wi-Fi 模組最初是從 Plan 9 系統與輕量級驅動移植過來的，後來為了支援 WPA2/WPA3，它內嵌並移植了精簡版的 wpa_supplicant（最新版本已同步到 v2.11）。它沒有 Linux 的 Netlink、虛擬檔案系統等複雜包袱。
   2. C++ 物件導向設計：Circle 將設備驅動、網路介面全部封裝成 C++ 類別（Class）。如果你的 MCU 開發環境支援 C++（如現代的 STM32 CubeIDE、Zephyr 等），程式碼的模組化對接會非常直覺。
   3. 記憶體管理更親民：Circle 運作在樹莓派的 Bare-metal 環境，其動態記憶體分配（new/delete）邏輯與 MCU 上的 malloc/free（或 RTOS heap）非常接近。 [1, 3, 4] 

------------------------------
## ⚠️ 移植到 MCU 必須面對的挑戰
雖然 Circle 已經是 Bare-metal，但它是為**樹莓派（應用處理器級別，Cortex-A 系列）**設計的，直接搬到 MCU（微控制器，Cortex-M 系列） 仍有以下硬傷：
## 1. 硬體綁定過深（晶片驅動問題）

* 
* 問題：Circle 內建的 Wi-Fi 驅動是針對樹莓派板載的無線晶片（如 Cypress/Infineon CYW43439 或 Broadcom BCM43438 等 SoftMAC/FullMAC 晶片）。
* 影響：如果你 MCU 連接的不是樹莓派同款晶片，Circle 的「底層驅動代碼」對你沒有用。你只能借鑑它的 wpa_supplicant 移植層 與 802.11 協定管理邏輯，底層傳輸（SDIO/SPI）仍須自己重寫。 [2] 
* 

## 2. 執行緒與排程模型（Scheduler）的差異

* 
* 問題：Circle 預設使用協作式多工（Cooperative Multitasking），它的網路和 Wi-Fi 任務非常依賴頻繁呼叫 scheduler->yield() 來讓出 CPU。
* 影響：MCU 常用的 RTOS（如 FreeRTOS）是搶佔式排程（Preemptive）。在移植時，你需要將 Circle 的輪詢/協作邏輯，包裝成獨立的 RTOS Task，並妥善處理多執行緒下的資料競爭（Locking）。 [5] 
* 

## 3. 記憶體開銷（對樹莓派很小，對 MCU 很大）

* 
* Circle 在樹莓派上運算時，記憶體是以「MB」為單位在揮霍，其內嵌的 wpa_supplicant v2.11 在解析 WPA3 握手（SAE 協定）時會消耗不少 Stack 與 Heap。你的 MCU 最好有 512KB 以上的 RAM，若是只有 64KB/128KB 的 MCU 會非常吃力。 [4, 6] 
* 

------------------------------
## 🛠️ 具體的移植實踐策略
如果您決定動手，建議採取**「去骨留肉」**的抽離法：

   1. 抽取 wpa_supplicant 核心：
   參考 Circle 中 addon/wlan/ 內如何將 wpa_supplicant 裁剪、如何封裝 Crypto（連接到 mbed TLS）的實作。
   2. 替換硬體傳輸層：
   將 Circle 的 SDIO/USB 暫存器讀寫，替換為您 MCU 的 SPI/SDIO 驅動（例如 STM32 的 HAL 庫）。
   3. 對接 TCP/IP Stack：
   Circle 內部有自己的簡單網路層或對接特定核心，在 MCU 上你必須將它剝離，將收到的 Data 幀去頭後直接餵給 lwIP。 [4, 7] 

這個想法非常具有極客精神！為了進一步評估可行性，我想確認：

* 
* 您打算使用的 MCU 型號 是什麼？（這決定了 RAM 空間 是否足夠）
* 您的 Wi-Fi 模組晶片型號是什麼？（如果是樹莓派同款的 CYW 系列，Circle 的驅動就極具參考價值）
* 


[1] [https://stdrc.cc](https://stdrc.cc/post/2021/01/22/porting-circle/)
[2] [https://github.com](https://github.com/uhub/awesome-c/blob/master/README.md)
[3] [https://github.com](https://github.com/rsta2/circle/issues/93)
[4] [https://github.com](https://github.com/rsta2/circle/blob/master/CHANGELOG.md)
[5] [https://github.com](https://github.com/rsta2/circle/issues/93)
[6] [https://forums.raspberrypi.com](https://forums.raspberrypi.com/viewtopic.php?t=90130)
[7] [https://github.com](https://github.com/rsta2/circle/issues/125)

```
