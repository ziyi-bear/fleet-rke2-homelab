# PlantUML Server

此 Bundle 用於在 Kubernetes 叢集中透過 **Rancher Fleet** 自動部署 **PlantUML Server** 服務。

---

## 📌 服務說明

[PlantUML Server](https://github.com/plantuml/plantuml-server) 是一個網頁版與 API 繪圖服務，可用於線上將 PlantUML 文字描述檔渲染為圖片 (PNG, SVG, TXT)。

* **預設 Namespace**：`plantuml`
* **Helm Chart 來源**：`https://stevehipwell.github.io/helm-charts` (Chart: `plantuml`)
* **Ingress Controller**：`nginx`
* **TLS / Cert-Manager**：自動引用 Cluster templateValues `${ .ClusterValues.certManager.clusterIssuer }`
* **預設網址**：`https://plantuml.${ .ClusterValues.domain.base }`

---

## ⚙️ fleet.yaml 設定範例說明

```yaml
defaultNamespace: plantuml

labels:
  app: plantuml

helm:
  repo: https://stevehipwell.github.io/helm-charts
  chart: plantuml
  releaseName: plantuml
  values:
    image:
      tag: jetty
    env:
      - name: PLANTUML_LIMIT_SIZE
        value: "8192"  # 將圖片上限放寬至 8192px (若還不夠可改為 16384)
      - name: JAVA_TOOL_OPTIONS
        value: "-Djetty.httpConfig.requestHeaderSize=131072"
    ingress:
      enabled: true
      ingressClassName: "nginx"
      annotations:
        cert-manager.io/cluster-issuer: "${ .ClusterValues.certManager.clusterIssuer }"
        external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"
        external-dns.alpha.kubernetes.io/target: "${ .ClusterValues.ingress.targets.nginx }"
        nginx.ingress.kubernetes.io/large-client-header-buffers: "4 128k"
        nginx.ingress.kubernetes.io/client-header-buffer-size: "128k"
        nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
        nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
        nginx.ingress.kubernetes.io/proxy-busy-buffers-size: "256k"
        nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
        nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
        nginx.ingress.kubernetes.io/configuration-snippet: |
          http2_max_field_size 128k;
          http2_max_header_size 128k;
      hosts:
        - "plantuml.${ .ClusterValues.domain.base }"
      tls:
        - secretName: plantuml-tls
          hosts:
            - "plantuml.${ .ClusterValues.domain.base }"
```

---

## 🛠️ 常見問題與異常排除 (Troubleshooting)

### 1. `net::ERR_HTTP2_PROTOCOL_ERROR` / `net::ERR_CONNECTION_CLOSED` (長 UML 圖片)
* **原因**：PlantUML 預設會將繪圖碼壓縮編碼後放入 GET 請求的 URL Path (`/png/<encoded>`)。當 UML 圖形龐大時，URL 長度會超過 10KB~20KB。NGINX Ingress 在 HTTP/2 模式下預設 `http2_max_field_size` (HTTP/2 Header / Path 限制) 通常只有 4k~16k，導致 NGINX 直接主動中斷 HTTP/2 連線。
* **解法**：
  1. **Ingress 設定**：由於採用動態 DNS 直連（未過 CDN 限制），只要部署最新的 `fleet.yaml`，透過 `configuration-snippet` 將 NGINX Ingress 的 `http2_max_field_size` 與 `http2_max_header_size` 放大至 `128k`，即可正常處理 10KB~128KB 的超長 GET URL。
  2. **前端 / 外掛改用 POST 請求**：若前端套件支援，請調整為發送 HTTP POST 至 `/png`（Request Body 放 PlantUML 原始碼），可完全避免超長 URL 問題。

