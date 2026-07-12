# SearXNG 部署與優化文件 (for Open WebUI)

此目錄用於部署 **SearXNG**，作為 Open WebUI 的後端網頁搜尋引擎。

## 為什麼不用預設的 `config.settings.data`？

如果完全不設定 `config.settings.data`，SearXNG 將會使用 Helm Chart 的預設值，這在與 Open WebUI 整合時會遇到以下問題：

1. **安全性風險 (`secret_key`)**：
   * Helm Chart 預設包含一個公開的 Hardcoded `secret_key`。任何人都可以利用此預設金鑰解密/偽造 SearXNG 的 Cookie 或 Session。我們為此生成了獨立的隨機金鑰以策安全。
2. **API 連線被阻擋 (`limiter`)**：
   * 預設情況下，若啟動了 IP Limiter（速率限制），SearXNG 會對頻繁請求的 IP 進行阻擋。因為 Open WebUI 的所有搜尋請求都是從其 Pod IP 發出，對 SearXNG 來說這些請求來自「同一個 IP」，因此非常容易觸發防爬蟲機制，導致搜尋失效（回傳 429 錯誤）。將 `limiter` 設為 `false` 可以解決此內部呼叫問題。
3. **搜尋格式不支援 (`formats`)**：
   * Open WebUI 是透過 JSON 格式與 SearXNG API 進行對接。若 formats 中沒有包含 `json`，API 請求將會被拒絕。

---

## 參數用途說明

在 `fleet.yaml` 的 `config.settings.data` 中，各項關鍵參數用途如下：

| 參數名稱 | 預設值 | 本專案設定 | 用途說明 |
| :--- | :--- | :--- | :--- |
| `use_default_settings` | `true` | `true` | 繼承 SearXNG 預設的基礎設定，僅覆寫下方指定的值。 |
| `server.secret_key` | 公開預設值 | 隨機十六進位值 | 用於加密瀏覽器 Cookie 和 Session 的金鑰。 |
| `server.limiter` | `true` | `false` | 控制是否啟用速率限制與防爬蟲機制。在內部叢集串接時必須關閉。 |
| `server.image_proxy` | `true` | `true` | 是否代理搜尋結果中的圖片。若開啟，圖片將透過 SearXNG Pod 下載後再傳給用戶，防範隱私洩露。 |
| `server.port` / `bind_address` | `8080` / `0.0.0.0` | `8080` / `0.0.0.0` | 服務監聽的埠口與綁定 IP。 |
| `search.formats` | `[html]` | `[html, json]` | 允許的搜尋回傳格式。Open WebUI 必須使用 `json`。 |

---

## 針對 Open WebUI 與 Ollama 需求的最佳化設定

為提升 AI 搜尋（RAG，檢索增強生成）的速度與品質，可以針對 SearXNG 進行以下優化調整：

### 1. 引擎超時優化 (Timeout)
AI 回答問題需要快速回應。預設情況下，SearXNG 的引擎超時時間是相對寬鬆的，若某個搜尋引擎（例如 Google）回應慢，就會拖慢整個 AI 的回答速度。
* **建議調整**：將 `outgoing.request_timeout` 設定為 `1.5` 秒，讓慢速搜尋引擎快速超時，確保 Open WebUI 能夠在最短時間內拿到已回應引擎的結果。

### 2. 引擎篩選 (Engine Selection)
有些搜尋引擎（如 Google）有強烈的防爬機制（CAPTCHA），容易回傳錯誤。而有些引擎（如 DuckDuckGo, Brave）則非常穩定。
* **建議調整**：在設定檔中啟用反應快且不易封鎖的引擎，並關閉不穩定或需要金鑰的引擎。

### 3. 關閉無用的搜尋分類
Open WebUI 主要需要**網頁文字**來進行閱讀和分析，並不需要圖片、影片、檔案等分類搜尋。
* **建議調整**：在 SearXNG 設定中可以限制只回傳 Web 類型的結果，以節省頻寬與運算資源。

### 4. 範例：進階優化 `fleet.yaml` 設定
如果您希望進行極致的效能優化，可以參考以下 values 配置：

```yaml
  values:
    config:
      settings:
        enabled: true
        data: |
          use_default_settings: true
          server:
            secret_key: "6b2a0fbcf7c28d541e21b06c83df9e4726f4f228b3c990263f10ef37cf8480d1"
            limiter: false
            image_proxy: false # RAG 不需要下載圖片，關閉可節省流量與運算
          outgoing:
            request_timeout: 1.5 # 限制外發搜尋超時為 1.5 秒，確保 AI 回答不延遲
          search:
            formats:
              - json # 只保留 JSON 格式，減少不必要的渲染
          # 篩選穩定且速度快的引擎
          engines:
            - name: google
              engine: google
              shortcut: go
              disabled: true # 關閉容易出現 CAPTCHA 的 Google
            - name: duckduckgo
              engine: duckduckgo
              shortcut: ddg
              disabled: false
            - name: wikipedia
              engine: wikipedia
              shortcut: wp
              disabled: false
```
