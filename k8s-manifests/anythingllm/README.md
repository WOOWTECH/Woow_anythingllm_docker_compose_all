# AnythingLLM K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

All-in-one AI application with RAG (Retrieval-Augmented Generation), document chat, and multi-LLM support. AnythingLLM allows you to upload documents, create workspaces, and chat with your data using local or cloud LLM providers. It supports embedding models, vector databases, and provides a polished web interface for knowledge management. This deployment is pre-configured to use Ollama as the LLM provider via cluster-internal networking.

> **GitHub Repo (Podman/Docker):** [Woow_anythingllm_docker_compose_all](https://github.com/WOOWTECH/Woow_anythingllm_docker_compose_all)

### Architecture

```
                        AnythingLLM K3s Architecture
 ============================================================================

   External Access                 K3s Cluster (namespace: anythingllm)
  +----------------+          +--------------------------------------------+
  |                |          |                                            |
  |  Browser       |          |   +------------------------------------+   |
  |  :30301  ------+---NodePort-->|  Service: anythingllm              |   |
  |                |          |   |  ClusterIP :3001                   |   |
  +----------------+          |   +----------------+-------------------+   |
                              |                    |                       |
                              |                    v                       |
                              |   +------------------------------------+   |
                              |   |  Pod: anythingllm                  |   |
                              |   |  Image: mintplexlabs/anythingllm   |   |
                              |   |  Port: 3001                        |   |
                              |   |  Volume: /app/server/storage       |   |
                              |   +----------------+-------------------+   |
                              |                    |                       |
                              +--------------------+-----------------------+
                                                   |
                                    Cluster-Internal DNS
                                                   |
                              +--------------------+-----------------------+
                              |  K3s Cluster (namespace: ollama)           |
                              |   +------------------------------------+   |
                              |   |  Service: ollama                   |   |
                              |   |  ollama.ollama.svc.cluster.local   |   |
                              |   |  Port: 11434                       |   |
                              |   +------------------------------------+   |
                              +--------------------------------------------+
```

### Features

- RAG (Retrieval-Augmented Generation) with document upload and chat
- Multi-LLM provider support (Ollama, OpenAI, Azure, Anthropic, etc.)
- Workspace-based knowledge management
- Embedding model and vector database integration
- Polished web interface with setup wizard
- Pre-configured Ollama connection via cluster-internal DNS

### Quick Start

```bash
# 1. Ensure Ollama is deployed first
kubectl -n ollama get pods

# 2. Deploy AnythingLLM
kubectl apply -k k8s-manifests/anythingllm/

# 3. Verify pods are running
kubectl -n anythingllm get pods

# 4. Watch startup logs
kubectl -n anythingllm logs deploy/anythingllm -f
```

### Configuration

#### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `STORAGE_DIR` | Path for AnythingLLM data storage | `/app/server/storage` | Yes |
| `LLM_PROVIDER` | LLM provider to use | `ollama` | Yes |
| `OLLAMA_BASE_PATH` | Ollama API endpoint (cluster-internal DNS) | `http://ollama.ollama.svc.cluster.local:11434` | Yes |

#### No Secrets Required

AnythingLLM does not use Kubernetes secrets in this deployment. API keys and LLM provider settings are configured through the web interface during initial setup.

### Accessing the Service

| Endpoint | URL | Protocol |
|----------|-----|----------|
| AnythingLLM Web UI | `http://<node-ip>:30301` | HTTP (NodePort) |
| Internal (cluster) | `http://anythingllm.anythingllm.svc.cluster.local:3001` | HTTP |

On first access, you will be guided through the setup wizard to configure your LLM provider, embedding model, and vector database.

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `anythingllm-data` | `/app/server/storage` | 10Gi | Uploaded documents, vector embeddings, workspaces, chat history, settings |

The PVC uses the `local-path` storage class (k3s default).

### Backup & Restore

#### Backup

```bash
# Backup all AnythingLLM data (documents, embeddings, settings)
kubectl -n anythingllm exec deploy/anythingllm -- tar czf /tmp/anythingllm-backup.tar.gz /app/server/storage
kubectl -n anythingllm cp anythingllm/<pod-name>:/tmp/anythingllm-backup.tar.gz ./anythingllm-backup.tar.gz
```

#### Restore

```bash
# Restore AnythingLLM data
kubectl -n anythingllm cp ./anythingllm-backup.tar.gz anythingllm/<pod-name>:/tmp/anythingllm-backup.tar.gz
kubectl -n anythingllm exec deploy/anythingllm -- tar xzf /tmp/anythingllm-backup.tar.gz -C /

# Restart to pick up restored data
kubectl -n anythingllm rollout restart deploy/anythingllm
```

### Useful Commands

```bash
# Check pod status
kubectl -n anythingllm get pods

# View real-time logs
kubectl -n anythingllm logs deploy/anythingllm -f

# Restart AnythingLLM
kubectl -n anythingllm rollout restart deploy/anythingllm

# Check storage usage
kubectl -n anythingllm exec deploy/anythingllm -- df -h /app/server/storage

# Test Ollama connectivity from AnythingLLM pod
kubectl -n anythingllm exec deploy/anythingllm -- \
  wget -qO- http://ollama.ollama.svc.cluster.local:11434/api/tags

# List available Ollama models
kubectl -n ollama exec sts/ollama -- ollama list
```

### Troubleshooting

#### "Cannot connect to Ollama" during setup

Verify Ollama is running and reachable:

```bash
# Check Ollama pods
kubectl -n ollama get pods

# Test connectivity from AnythingLLM pod
kubectl -n anythingllm exec deploy/anythingllm -- \
  wget -qO- http://ollama.ollama.svc.cluster.local:11434/api/tags

# Verify at least one model is pulled
kubectl -n ollama exec sts/ollama -- ollama list
```

#### No models available in the setup wizard

AnythingLLM lists models from the Ollama API. Ensure you have pulled at least one model:

```bash
kubectl -n ollama exec sts/ollama -- ollama pull llama3.1:8b
```

#### Document upload fails

Check available storage space:

```bash
kubectl -n anythingllm exec deploy/anythingllm -- df -h /app/server/storage
```

#### Pod OOMKilled

AnythingLLM loads embeddings into memory. For large document collections, increase memory:

```yaml
resources:
  limits:
    memory: 6Gi  # Increase from 4Gi
```

#### Change LLM provider

To switch from Ollama to another provider (e.g., OpenAI), update the settings through the AnythingLLM web UI under Settings > LLM Provider. You can also update the ConfigMap:

```yaml
LLM_PROVIDER: "openai"
```

```bash
kubectl apply -k k8s-manifests/anythingllm/
kubectl -n anythingllm rollout restart deploy/anythingllm
```

### File Structure

```
anythingllm/
├── anythingllm-deployment.yaml   # Deployment for AnythingLLM pod
├── anythingllm-service.yaml      # NodePort service (30301 -> 3001)
├── configmap.yaml                # Environment variables (LLM provider, Ollama URL)
├── kustomization.yaml            # Kustomize resource list
├── namespace.yaml                # Namespace: anythingllm
├── pvc.yaml                      # PersistentVolumeClaim for data storage
└── README.md                     # This file
```

---

## 中文

### 概述

AnythingLLM 是一款整合 RAG（檢索增強生成）、文件對話及多 LLM 支援的全方位 AI 應用程式。您可以上傳文件、建立工作區，並透過本地或雲端 LLM 提供者與資料進行對話。支援嵌入模型、向量資料庫，並提供精緻的 Web 介面進行知識管理。本部署預設透過叢集內部網路使用 Ollama 作為 LLM 提供者。

> **GitHub Repo (Podman/Docker):** [Woow_anythingllm_docker_compose_all](https://github.com/WOOWTECH/Woow_anythingllm_docker_compose_all)

### 架構圖

```
                        AnythingLLM K3s 架構
 ============================================================================

   外部存取                      K3s 叢集 (namespace: anythingllm)
  +----------------+          +--------------------------------------------+
  |                |          |                                            |
  |  瀏覽器        |          |   +------------------------------------+   |
  |  :30301  ------+---NodePort-->|  Service: anythingllm              |   |
  |                |          |   |  ClusterIP :3001                   |   |
  +----------------+          |   +----------------+-------------------+   |
                              |                    |                       |
                              |                    v                       |
                              |   +------------------------------------+   |
                              |   |  Pod: anythingllm                  |   |
                              |   |  映像: mintplexlabs/anythingllm    |   |
                              |   |  埠號: 3001                        |   |
                              |   |  磁碟區: /app/server/storage       |   |
                              |   +----------------+-------------------+   |
                              |                    |                       |
                              +--------------------+-----------------------+
                                                   |
                                         叢集內部 DNS
                                                   |
                              +--------------------+-----------------------+
                              |  K3s 叢集 (namespace: ollama)              |
                              |   +------------------------------------+   |
                              |   |  Service: ollama                   |   |
                              |   |  ollama.ollama.svc.cluster.local   |   |
                              |   |  埠號: 11434                       |   |
                              |   +------------------------------------+   |
                              +--------------------------------------------+
```

### 功能特色

- RAG（檢索增強生成）支援文件上傳與對話
- 多 LLM 提供者支援（Ollama、OpenAI、Azure、Anthropic 等）
- 基於工作區的知識管理
- 嵌入模型與向量資料庫整合
- 精緻的 Web 介面與設定精靈
- 預設透過叢集內部 DNS 連接 Ollama

### 快速開始

```bash
# 1. 確認 Ollama 已部署
kubectl -n ollama get pods

# 2. 部署 AnythingLLM
kubectl apply -k k8s-manifests/anythingllm/

# 3. 確認 Pod 運作中
kubectl -n anythingllm get pods

# 4. 查看啟動日誌
kubectl -n anythingllm logs deploy/anythingllm -f
```

### 設定

#### 環境變數

| 變數 | 說明 | 預設值 | 必填 |
|------|------|--------|------|
| `STORAGE_DIR` | AnythingLLM 資料儲存路徑 | `/app/server/storage` | 是 |
| `LLM_PROVIDER` | 使用的 LLM 提供者 | `ollama` | 是 |
| `OLLAMA_BASE_PATH` | Ollama API 端點（叢集內部 DNS） | `http://ollama.ollama.svc.cluster.local:11434` | 是 |

#### 無需 Secrets

本部署不使用 Kubernetes Secrets。API 金鑰與 LLM 提供者設定透過初次設定精靈在 Web 介面中設定。

### 存取服務

| 端點 | URL | 協定 |
|------|-----|------|
| AnythingLLM Web UI | `http://<node-ip>:30301` | HTTP (NodePort) |
| 叢集內部 | `http://anythingllm.anythingllm.svc.cluster.local:3001` | HTTP |

首次存取時，系統會引導您完成設定精靈，設定 LLM 提供者、嵌入模型及向量資料庫。

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|----------|----------|------|------|
| `anythingllm-data` | `/app/server/storage` | 10Gi | 上傳文件、向量嵌入、工作區、對話記錄、設定 |

PVC 使用 `local-path` 儲存類別（k3s 預設）。

### 備份與還原

#### 備份

```bash
# 備份所有 AnythingLLM 資料（文件、嵌入、設定）
kubectl -n anythingllm exec deploy/anythingllm -- tar czf /tmp/anythingllm-backup.tar.gz /app/server/storage
kubectl -n anythingllm cp anythingllm/<pod-name>:/tmp/anythingllm-backup.tar.gz ./anythingllm-backup.tar.gz
```

#### 還原

```bash
# 還原 AnythingLLM 資料
kubectl -n anythingllm cp ./anythingllm-backup.tar.gz anythingllm/<pod-name>:/tmp/anythingllm-backup.tar.gz
kubectl -n anythingllm exec deploy/anythingllm -- tar xzf /tmp/anythingllm-backup.tar.gz -C /

# 重啟以載入還原的資料
kubectl -n anythingllm rollout restart deploy/anythingllm
```

### 實用指令

```bash
# 查看 Pod 狀態
kubectl -n anythingllm get pods

# 查看即時日誌
kubectl -n anythingllm logs deploy/anythingllm -f

# 重啟 AnythingLLM
kubectl -n anythingllm rollout restart deploy/anythingllm

# 檢查儲存使用量
kubectl -n anythingllm exec deploy/anythingllm -- df -h /app/server/storage

# 從 AnythingLLM Pod 測試 Ollama 連線
kubectl -n anythingllm exec deploy/anythingllm -- \
  wget -qO- http://ollama.ollama.svc.cluster.local:11434/api/tags

# 列出可用的 Ollama 模型
kubectl -n ollama exec sts/ollama -- ollama list
```

### 疑難排解

#### 設定時出現「Cannot connect to Ollama」

確認 Ollama 正在運行且可連線：

```bash
# 檢查 Ollama Pod
kubectl -n ollama get pods

# 從 AnythingLLM Pod 測試連線
kubectl -n anythingllm exec deploy/anythingllm -- \
  wget -qO- http://ollama.ollama.svc.cluster.local:11434/api/tags

# 確認至少已拉取一個模型
kubectl -n ollama exec sts/ollama -- ollama list
```

#### 設定精靈中無可用模型

AnythingLLM 從 Ollama API 取得模型列表。請確認已拉取至少一個模型：

```bash
kubectl -n ollama exec sts/ollama -- ollama pull llama3.1:8b
```

#### 文件上傳失敗

檢查可用儲存空間：

```bash
kubectl -n anythingllm exec deploy/anythingllm -- df -h /app/server/storage
```

#### Pod OOMKilled

AnythingLLM 會將嵌入載入記憶體。對於大型文件集合，請增加記憶體限制：

```yaml
resources:
  limits:
    memory: 6Gi  # 從 4Gi 增加
```

#### 變更 LLM 提供者

若要從 Ollama 切換到其他提供者（如 OpenAI），可在 AnythingLLM Web UI 的 Settings > LLM Provider 中更新設定。也可以更新 ConfigMap：

```yaml
LLM_PROVIDER: "openai"
```

```bash
kubectl apply -k k8s-manifests/anythingllm/
kubectl -n anythingllm rollout restart deploy/anythingllm
```

### 檔案結構

```
anythingllm/
├── anythingllm-deployment.yaml   # AnythingLLM Pod 的 Deployment
├── anythingllm-service.yaml      # NodePort 服務 (30301 -> 3001)
├── configmap.yaml                # 環境變數（LLM 提供者、Ollama URL）
├── kustomization.yaml            # Kustomize 資源列表
├── namespace.yaml                # 命名空間: anythingllm
├── pvc.yaml                      # 持久卷宣告（資料儲存）
└── README.md                     # 本文件
```
