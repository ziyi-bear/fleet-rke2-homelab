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
    ingress:
      enabled: true
      ingressClassName: "nginx"
      annotations:
        cert-manager.io/cluster-issuer: "${ .ClusterValues.certManager.clusterIssuer }"
        external-dns.alpha.kubernetes.io/cloudflare-proxied: "false"
        external-dns.alpha.kubernetes.io/target: "${ .ClusterValues.ingress.targets.nginx }"
      hosts:
        - "plantuml.${ .ClusterValues.domain.base }"
      tls:
        - secretName: plantuml-tls
          hosts:
            - "plantuml.${ .ClusterValues.domain.base }"
```
