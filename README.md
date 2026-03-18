# Woow_anythingllm_docker_compose_all
---

## K3s/Kubernetes Deployment

This project also supports deployment on **K3s/Kubernetes** clusters. The K3s manifests are maintained on a separate branch.

### Quick Start (K3s)

```bash
# Clone the k3s branch
git clone -b k3s https://github.com/WOOWTECH/Woow_anythingllm_docker_compose_all.git Woow_anythingllm_docker_compose_all-k3s
cd Woow_anythingllm_docker_compose_all-k3s

# Edit secrets before deploying
nano secret.yaml

# Deploy to your k3s cluster
kubectl apply -k .

# Verify pods are running
kubectl -n anythingllm get pods
```

### Deployment Methods Comparison

| Feature | Podman/Docker Compose | K3s/Kubernetes |
|---------|----------------------|----------------|
| Branch | `main` | `k3s` |
| Orchestrator | Podman / Docker | K3s / Kubernetes |
| Config format | `.env` + `docker-compose.yml` | ConfigMap + Secret + YAML manifests |
| Scaling | Manual | `kubectl scale` |
| Health checks | Docker healthcheck | liveness/readiness/startup probes |
| Service discovery | Docker DNS | Kubernetes DNS (`svc.cluster.local`) |
| Storage | Docker volumes | PersistentVolumeClaims |
| Rolling updates | `docker compose pull && up -d` | `kubectl rollout restart` |

> For full K3s deployment documentation, switch to the [`k3s` branch](https://github.com/WOOWTECH/Woow_anythingllm_docker_compose_all/tree/k3s).
