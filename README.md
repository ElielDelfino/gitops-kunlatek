# gitops-kunlatek

Repositório GitOps da aplicação **kunlatek-api** no EKS. O ArgoCD monitora este repositório e sincroniza automaticamente qualquer mudança no cluster.

## Padrão utilizado

**ArgoCD + Helm + Argo Rollouts (canary)**

```
GitHub (este repo)
    └── ArgoCD detecta mudança
            └── Helm Chart → recursos Kubernetes
                    └── Argo Rollouts executa canary deployment
```

## Estrutura

```
gitops-kunlatek/
├── argocd/
│   └── kunlatek-api-app.yaml     # Application resource do ArgoCD
└── apps/
    └── kunlatek-api/             # Helm Chart
        ├── Chart.yaml
        ├── values.yaml           # valores — atualizado automaticamente pelo CI
        └── templates/
            ├── deployment.yaml        # Argo Rollout com estratégia canary
            ├── service.yaml           # Services stable + canary
            ├── ingress.yaml           # ALB Ingress (internet-facing)
            ├── hpa.yaml               # HPA para API e Worker
            ├── worker.yaml            # Deployment do worker SQS
            ├── configmap-worker.yaml  # SQS_WORKER_URL (não é secret)
            ├── serviceaccount.yaml    # ServiceAccount com IRSA
            ├── cluster-secret-store.yaml  # ESO — autenticação no Secrets Manager
            ├── secret-provider.yaml       # ExternalSecret — sincroniza kunlatek/app
            ├── network-policies.yaml      # Zero-trust: deny-all + allowlist
            └── analysis-template.yaml    # Validação de saúde durante canary
```

## Templates

| Template | Tipo | Descrição |
|---|---|---|
| `deployment.yaml` | Argo Rollout | Deploy canary: 20% → 50% → 100% com pausas |
| `service.yaml` | Service (x2) | `kunlatek-api-stable` e `kunlatek-api-canary` para o ALB |
| `ingress.yaml` | Ingress | ALB internet-facing, target type `ip` |
| `hpa.yaml` | HPA (x2) | API: 2–6 réplicas (CPU 70%, mem 80%) / Worker: 1–4 réplicas (CPU 90%) |
| `worker.yaml` | Deployment | Processamento assíncrono via SQS (`node dist/worker`) |
| `configmap-worker.yaml` | ConfigMap | `SQS_WORKER_URL` injetado pelo CI — não é secret |
| `serviceaccount.yaml` | ServiceAccount | Anotada com IRSA role ARN para acesso AWS sem credenciais |
| `cluster-secret-store.yaml` | ClusterSecretStore | ESO autentica via JWT da SA `external-secrets/external-secrets` |
| `secret-provider.yaml` | ExternalSecret | Sincroniza `kunlatek/app` do AWS Secrets Manager a cada 1h |
| `network-policies.yaml` | NetworkPolicy | Deny-all ingress/egress + allowlist explícito por pod |
| `analysis-template.yaml` | AnalysisTemplate | Valida taxa de sucesso ≥ 99% durante o canary |

## Fluxo de secrets

```
AWS Secrets Manager (kunlatek/app)
    └── ClusterSecretStore (ESO — IRSA da SA external-secrets)
            └── ExternalSecret (sincronização a cada 1h)
                    └── K8s Secret "kunlatek-api-secret"
                            └── Pod (envFrom: secretRef)
```

O `SQS_WORKER_URL` **não** passa por esse fluxo — é uma configuração sem dado sensível, injetada via `ConfigMap` pelo pipeline de deploy da aplicação.

## Canary deployment

O Argo Rollout executa o deploy em etapas:

```
Início: 2 pods stable
  └── setWeight 20% → 1 pod canary + 2 stable (pausa 30s)
        └── AnalysisTemplate valida health do canary
              └── setWeight 50% → 1 pod canary + 1 stable (pausa 30s)
                    └── 100% → pods stable substituídos pelo canary
```

Se o `AnalysisTemplate` reprovar (taxa de sucesso < 99%), o Rollout faz rollback automático para os pods stable.

## Network policies

Padrão zero-trust: todo tráfego é negado por padrão e liberado explicitamente.

| Policy | Direção | Permite |
|---|---|---|
| default-deny-ingress | Ingress | nada (deny-all) |
| allow-alb-ingress | Ingress | porta 3000 vindo da VPC (10.0.0.0/16) |
| default-deny-egress | Egress | nada (deny-all) |
| allow-app-egress | Egress | MySQL 3306 → VPC, HTTPS 443 → qualquer, DNS 53 → kube-system |
| allow-worker-egress | Egress | HTTPS 443 → qualquer, DNS 53 → kube-system |

## Values atualizados pelo CI

O pipeline da aplicação (`deploy.yaml`) atualiza automaticamente estes campos após cada build:

| Campo | Atualizado por |
|---|---|
| `image.repository` | CI após push para ECR |
| `image.tag` | CI — formato `{semver}-{short-sha}` |
| `worker.sqsUrl` | CI — `aws sqs get-queue-url` |

## Como registrar a Application no ArgoCD

Após o `terraform apply` da infra e os addons prontos:

```bash
# 1. Conectar kubectl ao cluster
aws eks update-kubeconfig --name kunlatek-eks --region us-east-1

# 2. Aguardar ArgoCD, ESO e Argo Rollouts prontos
kubectl rollout status deployment/argocd-server -n argocd
kubectl rollout status deployment/external-secrets -n external-secrets
kubectl rollout status deployment/argo-rollouts -n argo-rollouts

# 3. Aplicar o manifest — repositório público, sem credenciais necessárias
kubectl apply -f argocd/kunlatek-api-app.yaml

# 4. Acompanhar o sync
kubectl get application kunlatek-api -n argocd -w
kubectl get pods -n kunlatek -w
```

## Acesso à UI do ArgoCD

```bash
# Port-forward (service é ClusterIP)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Senha inicial do admin
kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Acesse: `https://localhost:8080`
