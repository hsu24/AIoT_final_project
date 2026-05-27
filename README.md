# 智慧城市交通安全：基於邊緣運算之智慧疲勞駕駛偵測系統
### Smart Fatigue Driving Detection System based on Edge AI

---

## 📌 專案背景
在智慧城市與 AIoT 設備普及的趨勢下，智慧交通與安全駕駛已成為城市治理的核心。然而，儘管車輛硬體與主動安全防護不斷升級，根據交通統計，**疲勞與分心駕駛仍佔重大交通事故的近 20%**。

本專案旨在突破傳統監控設備的技術瓶頸，實現：
1. **毫秒級即時防護**：捨棄雲端傳輸的延遲與隱私疑慮，完全在邊緣端（Edge）進行即時推論。
2. **極致輕量化與普及化**：透過高效能的輕量化神經網路，讓平價的電腦、車載邊緣運算主機（如 Raspberry Pi、Jetson Nano 或一般筆電）只需搭配普通 USB 鏡頭，即可享有高階車載 DMS（Driver Monitoring System）的安全監控。

---

## 🧠 核心技術與模型選擇：MobileNetV3-Small
本系統核心採用 **MobileNetV3-Small** 輕量化卷積神經網路架構，作為駕駛狀態的即時影像分類器。

### 🔄 技術路線轉折與決策優勢
原先專案計畫採用 *YOLOv8-Pose* 偵測面部關鍵點（計算 EAR 眼睛閉合比與 MAR 打哈欠張力）。但在深入分析 **NTHU-DDD 資料集** 後，發現該資料集為**資料夾分類標籤形式**，並無提供面部關鍵點之座標標記（XY Coordinate Labels）。

為了最大化利用 NTHU-DDD 豐富的真實駕駛疲勞數據，我們將架構優化為**基於 MobileNetV3-Small 的四分類影像識別模型**。此决策帶來了顯著的優勢：
- **邊緣效能極大化**：MobileNetV3 融合了深度可分離卷積（Depthwise Separable Convolutions）與隨機通道注意力機制（Squeeze-and-Excitation, SE），在 CPU 上的推論時間僅需數毫秒。
- **免去繁重的前處理**：不需額外運行臉部特徵點偵測器（如 Dlib 或 MediaPipe），降低多階段 Pipeline 的累計延遲與運算開銷，大幅減少系統資源佔用。
- **端到端直接學習**：模型直接從海量駕駛表情影像中提取高維特徵，對於頭部擺動、點頭、打哈欠等複雜疲勞特徵具有更強的魯棒性。

---

## 📊 數據集與模型訓練
### 📂 NTHU-DDD (Driver Drowsiness Detection) 數據集
系統使用 NTHU-DDD 真實駕駛疲勞數據集（包含不同光照、紅外線環境、駕駛配戴眼鏡、多種角度等真實場景），共計 **66,521 張** 影像，劃分為 4 個類別：
- `notdrowsy` (清醒狀態)：30,491 張
- `sleepyCombination` (疲勞眨眼與睏倦)：17,756 張
- `slowBlinkWithNodding` (慢速眨眼伴隨點頭)：9,412 張
- `yawning` (打哈欠)：8,862 張

### 🚀 Google Colab GPU 加速訓練
為克服本地 CPU 運算資源限制並發揮完整數據集的優勢，我們採用 Google Colab 的 **T4 GPU** 進行加速訓練：
- **訓練配置**：採用 Adam 優化器、CrossEntropyLoss 損失函數。
- **資料增強**：加入隨機水平翻轉（RandomHorizontalFlip）、亮度與對比度抖動（ColorJitter）以防止模型過擬合。
- **劃分比例**：80% 訓練集、20% 驗證集。
- **高效率訓練**：在 Batch Size = 128 的 GPU 配置下，僅需 15 ~ 25 分鐘即可完成 10 個 Epoch 的完整訓練，並自動儲存驗證集準確率最高的最佳權重檔 `mobilenetv3_fatigue_best.pth`（檔案大小僅約 **6.2 MB**，極度適合邊緣端部署）。

---

## 🛠️ 即時偵測系統與雙重門檻防誤報機制
在實際車載應用中，「誤警報」與「漏報」同樣致命。為了給予駕駛最舒適且準確的警示體驗，我們在 `camera_demo.py` 中設計了**雙重門檻防誤報控制系統**：

### 🎛️ 雙重調校旋鈕（雙重門檻）
1. **[旋鈕 1] 連續幀數限制 (`CONSECUTIVE_FRAMES_THRESHOLD = 45`)**
   - 在常見 30 FPS 鏡頭下，45 幀代表**必須連續偵測到 1.5 秒的疲勞狀態**才會觸發警告。這能完美排除正常的快速眨眼或駕駛轉頭等短暫行為，符合安全客觀規格。
2. **[旋鈕 2] 信心分數門檻 (`CONFIDENCE_THRESHOLD = 0.32` 可自訂調高)**
   - 只有當模型的預測信心（Confidence）高於該門檻時，該幀才會計入疲勞。這防止了模型在邊界模糊、光影突變時產生的不確定性預測干擾警報器。

### 🔄 漸進式計數恢復機制 (Gradual Counter Decrement)
當模型未偵測到疲勞或信心不足時，計數器**不會瞬間歸零**，而是以每次 **`-2`** 的速度漸進遞減。這有效避免了因駕駛瞬間眨眼、逆光陰影導致模型預測中斷時，警報 UI 產生劇烈閃爍（Flickering）的問題，大幅提升視覺穩定度。

### 🎨 高質感邊緣運算視覺 UI
即時影像畫面包含豐富且 premium 的視覺反饋：
- **即時預測狀態顯示**：畫面左上方以綠色（清醒）或橘色（疲勞）字體顯示目前模型預測類別及精確的百分比信心度。
- **動態警告進度條**：以動態橘/紅進度條展示目前連續疲勞幀數的累積進度。當計數器逼近門檻時，進度條會實時填滿，提供駕駛直觀的視覺心理預警。
- **全畫面警示橫幅**：一旦觸發預警門檻，畫面會疊加半透明的深紅色警告遮罩，並顯示強烈的 `"WARNING: FATIGUE DETECTED!"` 字樣，並可結合 Windows `winsound.Beep` 進行聽覺示警。

---

## 🚀 快速開始 / 安裝與執行

### 1. 環境需求
請確保您的邊緣裝置已安裝 Python（推薦 3.8 ~ 3.10）及以下依賴庫。
> [!NOTE]
> 若在本地 CPU 執行，請勿盲目升級 PyTorch 以防 DLL 初始化錯誤（WinError 1114）。建議安裝與硬體適配的穩定 CPU 版本。

```bash
pip install opencv-python pillow numpy torchvision torch
```

### 2. 檔案目錄結構
請確認您的專案資料夾結構如下：
```text
final project/
├── Multi class/                   # NTHU-DDD 數據集資料夾
│   └── train/
│       ├── notdrowsy/
│       ├── sleepyCombination/
│       ├── slowBlinkWithNodding/
│       └── yawning/
├── camera_demo.py                 # 本地即時鏡頭偵測 Demo 腳本
├── mobilenetv3_fatigue_best.pth  # 訓練完成的最佳權重檔（約 6.2 MB）
└── README.md                      # 專案說明文件
```

### 3. 啟動系統
將鏡頭連接至電腦，並於終端機執行：
```bash
python camera_demo.py
```
- 啟動後系統會開啟即時預覽視窗。
- 在畫面前嘗試閉眼、連續眨眼或打哈欠，可測試雙重門檻警報的靈敏度。
- 按下鍵盤 **`q`** 鍵即可安全退出系統並釋放相機資源。

---

## 👥 團隊成員
**國立中興大學 智慧物聯網應用與實作課程 - 第五組**
- **朱軒麟** - 電資四 (學號：4111064206)
- **徐宛禾** - 電資四 (學號：4111064237)

---

## 🌟 專案願景
我們致力於落實智慧城市「零傷亡 (Vision Zero)」的交通安全全球趨勢。藉由 MobileNetV3 極輕量、低延遲的邊緣運算特點，我們將原本專屬於高階車款的 DMS 科技，成功落地於親民的 AIoT 硬體平台上。**讓每一次毫秒級的成功預警，都能守護一個家庭的完整與安全。**

---

## 📝 參考文獻
1. 江秉穎等 (2022)。《現行國際疲勞駕駛監測科技資料蒐集彙整》。交通部運輸研究所。
2. Maleki Varnosfaderani, S., et al. (2025). A Comprehensive Review of Unobtrusive Biosensing in Intelligent Vehicles. *Bioengineering (Basel)*.
