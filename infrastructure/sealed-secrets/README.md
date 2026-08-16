# Bitnami Sealed Secrets 基礎設施服務

[Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) 用於解決在 GitOps 工作流程中「安全儲存敏感資訊 (Secrets)」的需求。它採用「非對稱加密 (Asymmetric Encryption)」架構，允許開發者使用公鑰在本地將 Kubernetes `Secret` 加密為 `SealedSecret` 自訂資源 (CRD)，並安全地提交至公開或私有的 Git 儲存庫。

只有運行於 Kubernetes 叢集內的 **Sealed Secrets Controller** (持有私鑰) 才能將 `SealedSecret` 解密還原為標準的 Kubernetes `Secret`。

---

## ⚙️ Fleet 部署設定

本服務透過 Rancher Fleet 進行自動化 GitOps 部署：

* **Fleet Bundle 目錄**：`infrastructure/sealed-secrets/`
* **目標命名空間 (Namespace)**：`sealed-secrets`
* **Helm Chart 來源**：`https://bitnami.github.io/sealed-secrets`
* **Chart 版本**：`2.19.1` (對應 Sealed Secrets Controller `v0.38.4`)

---

## 🛠️ CLI 工具與使用工作流

要使用 Sealed Secrets，開發者需在本地終端機安裝 `kubeseal` 命令列工具。

### 1. 安裝 `kubeseal` CLI

* **Windows (Chocolatey / Scoop)**:
  ```powershell
  choco install kubeseal
  # 或使用 Scoop
  scoop install kubeseal
  ```
* **macOS (Homebrew)**:
  ```bash
  brew install kubeseal
  ```
* **Linux (Direct Binary Download)**:
  ```bash
  KUBESEAL_VERSION="v0.28.0"
  wget https://github.com/bitnami-labs/sealed-secrets/releases/download/${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION:1}-linux-amd64.tar.gz
  tar -xvzf kubeseal-${KUBESEAL_VERSION:1}-linux-amd64.tar.gz kubeseal
  sudo install -m 755 kubeseal /usr/local/bin/kubeseal
  ```

---

### 2. 下載/取得叢集公鑰 (Public Key Certificate)

若可連線至 RKE2 叢集，可以直接透由 `kubeseal` 向控制器取得公鑰：

```bash
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  > pub-sealed-secrets.pem
```

---

### 3. 加密本地 Secret 為 `SealedSecret`

以建立一個名為 `db-credentials` 的密碼為例：

#### 步驟 A：產生明文 Secret 並透過管道加密

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecretPassword123 \
  --dry-run=client -o yaml | \
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --format yaml > db-credentials-sealed.yaml
```

#### 步驟 B：使用離線公鑰檔進行加密

若未直接連線至叢集，可指定預先下載的公鑰進行加密：

```bash
kubeseal --cert pub-sealed-secrets.pem --format yaml < my-secret.yaml > my-sealedsecret.yaml
```

產生出來的 `db-credentials-sealed.yaml` (`kind: SealedSecret`) 可以安全地提交至 Git 儲存庫中。

---

## 🔐 主金鑰 (Master Key) 備份與復原

> [!IMPORTANT]
> Controller 啟動時會自動生成用於解密的主金鑰 (Master Key Secret)。若叢集重建或重裝，必須還原該主金鑰，否則先前加密的所有 `SealedSecret` 將無法被解密！

### 1. 備份主金鑰

```bash
kubectl get secret -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > sealed-secrets-master-key-backup.yaml
```

### 2. 復原主金鑰

在新的叢集或重新部署控制器前，預先套用主金鑰：

```bash
kubectl create namespace sealed-secrets
kubectl apply -f sealed-secrets-master-key-backup.yaml
# 重新重啟 Controller 以載入金鑰
kubectl rollout restart deployment sealed-secrets -n sealed-secrets
```
