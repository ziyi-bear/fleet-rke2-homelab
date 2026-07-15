# Ansible AWX Operator 部署與說明文件

此目錄用於部署 **Ansible AWX Operator**，為 Kubernetes 叢集提供維護與管理 Ansible AWX 實例的能力。

## 目錄定位：為什麼放在 `apps/` 而非 `infrastructure/`？

本專案將服務劃分為兩大類：
1. **`infrastructure` (基礎設施)**：專門存放叢集底層的核心基礎服務（例如：動態儲存 [OpenEBS](file:///d:/gh/fleet-rke2-homelab/infrastructure/openebs/fleet.yaml)、證書管理 [cert-manager](file:///d:/gh/fleet-rke2-homelab/infrastructure/cert-manager/fleet.yaml)、以及資料庫運算 [CloudNativePG](file:///d:/gh/fleet-rke2-homelab/infrastructure/cnpg/fleet.yaml)）。這些服務是其他應用運作的基石。
2. **`apps` (終端應用與 SSO)**：存放終端使用者應用程式與開發/自動化工具（例如：Authentik、SonarQube、Coder、Open WebUI 等）。

雖然 **AWX Operator** 是一個 Operator（類似 CloudNativePG），但其最終管理的 **Ansible AWX** 是提供給維運人員使用的自動化管理平台與 Web 介面，性質上屬於終端維運應用，因此將其歸類於 `apps/` 目錄中，與其他應用程式一同管理。

---

## 運作機制與部署流程

### 1. 部署 Operator (CRD 註冊)
透過本目錄下的 `fleet.yaml`，Rancher Fleet 會將 AWX Operator 的 Helm Chart 部署至 `awx` 命名空間。
* **注意**：部署 Operator 本身**只會**註冊 Custom Resource Definitions (CRDs)（例如 `awxs.awx.ansible.com`）並運行 Operator 的控制器 Pod。此步驟**不會**自動建立 AWX 實體執行個體。

### 2. 部署 AWX 實例 (自訂資源)
要建立實際運作的 AWX 執行個體，您需要額外定義並套用一個 `kind: AWX` 的自訂資源檔。

#### `AWX` 實例配置範例 (`awx-instance.yaml`)
您可以撰寫如下的 YAML 設定檔，並套用至 `awx` 命名空間中：

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-homelab
  namespace: awx
spec:
  # 指定 AWX 的服務類型（可選用 ClusterIP，再搭配 Ingress 暴露服務）
  service_type: ClusterIP

  # 可以指定使用的 PostgreSQL 資料庫設定（預設會由 Operator 自動拉起一個隨附的 Postgres Pod）
  # postgres_configuration_secret: awx-postgres-secret
```

套用後，AWX Operator 的控制器會自動偵測到該資源，並為您拉起 AWX 的 Web 網頁服務、任務執行排程器 (Task Manager)、Redis 快取、以及 PostgreSQL 資料庫等整套運行環境。
