# ComfyUI (AMD ROCm / RX 6600 XT)

本 Fleet Bundle 用於在 **RKE2 Homelab 叢集**的 `d1581with6600xt` 節點上部署 **ComfyUI**，並專為 **影片生成與編輯 (Video Generation & Editing)** 進行硬體加速優化。

---

## 🚀 硬體與環境說明

* **顯示卡**: AMD Radeon RX 6600 XT (Navi 23 / `gfx1030`, 8 GB VRAM)
* **硬體加速設定**:
  * **ROCm 環境變數**: `HSA_OVERRIDE_GFX_VERSION=10.3.0` (強制啟動 PyTorch 對 Navi 23 顯示卡之 ROCm 支援)
  * **裝置透傳 (Device Passthrough)**: 掛載 `/dev/kfd` (AMD Kernel Fusion Driver) 與 `/dev/dri` (Direct Rendering Infrastructure - `renderD128`)
  * **節點綁定 (Node Selector)**: `kubernetes.io/hostname: d1581with6600xt`
* **VRAM 記憶體優化旗標**:
  * `--lowvram` / `--use-split-cross-attention` / `--fp16-vae` (適合 8 GB VRAM 生成高解析度與 AnimateDiff / Wan2.1 影片)

---

## 🎬 影片編輯與生成擴充套件 (Pre-installed Custom Nodes)

部署過程中會自動經由 initContainer 初始化安裝以下主要影片編輯與生成 Node：

1. **[ComfyUI-Manager](https://github.com/ltdrdata/ComfyUI-Manager)**: 提供 UI 介面一鍵下載模型與 Custom Nodes。
2. **[ComfyUI-VideoHelperSuite (VHS)](https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite)**: 支援影片載入 (Load Video)、剪輯切分 (Frame Select)、影音合成 (Combine Video/Audio)、MP4/GIF 匯出。
3. **[ComfyUI-AnimateDiff-Evolved](https://github.com/Kosinkadink/ComfyUI-AnimateDiff-Evolved)**: 支援 Text-to-Video、Video-to-Video 風格化轉換。
4. **[ComfyUI-Frame-Interpolation](https://github.com/Fannovel16/ComfyUI-Frame-Interpolation)**: 提供 RIFE / FILM 影片補幀與超慢動作運算。

---

## 📂 常用模型儲存位置與下載說明

所有模型與產出會儲存在持久化磁碟 (`comfyui-data` PVC) 中，掛載點為 `/root/ComfyUI`：

* **Checkpoint 主模型**: `/root/ComfyUI/models/checkpoints/`
* **VAE**: `/root/ComfyUI/models/vae/`
* **Motion Modules (AnimateDiff)**: `/root/ComfyUI/models/animatediff_models/`
* **ControlNet (影片風格/骨架控制)**: `/root/ComfyUI/models/controlnet/`
* **Outputs (生成的影片檔)**: `/root/ComfyUI/output/`

---

## 🌐 Ingress 存取資訊

* **網址**: `https://comfyui.${ .ClusterValues.domain.base }`
* **超時設定**: 已增強 `proxy-read-timeout: 3600s` 與 `proxy-body-size: 10G` 以支援大檔影片上傳與長影片渲染。

---

## ⚡ KEDA 自動縮放與 GPU 記憶體完全釋放 (Scale-to-Zero)

本服務已整合 **KEDA Autoscaler** (`scaledobject.yaml`)：

* **自動縮減至 0 (Scale down to 0)**: 當超過 15 分鐘（900 秒）未有 Ingress HTTP 請求流量時，KEDA 會將 ComfyUI Pod 副本數自動縮減至 **0**。
* **100% 釋放 GPU VRAM**: Pod 停止後，AMD Radeon RX 6600 XT 的 8 GB VRAM 與運算資源將被完全釋放，可完全讓給叢集內其他 AI 服務（如 Ollama 或 Open WebUI）使用。
* **自動喚醒 (Scale up to 1)**: 當有新的流量請求存取時，KEDA 會自動重新拉起 Pod (0 → 1)。

