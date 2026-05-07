# minecraft-be-server-k8s

Kubernetes manifests for a Minecraft Bedrock Edition server using [`itzg/minecraft-bedrock-server`](https://github.com/itzg/docker-minecraft-bedrock-server).

Managed by Flux from [homelab-gitops](https://github.com/ROBO358/homelab-gitops).

## 接続情報

| 項目 | 値 |
|---|---|
| アドレス | `192.168.1.103` |
| ポート | `19132` (UDP) |

## 構成

| リソース | 内容 |
|---|---|
| Deployment | `itzg/minecraft-bedrock-server:stable`、strategy: Recreate |
| Service | LoadBalancer UDP 19132/19133 (Cilium LB IPAM) |
| PVC | Longhorn 10Gi (`/data`) |
| CiliumNetworkPolicy | UDP ingress + HTTPS egress（バイナリDL用） |
| PrometheusRule | ダウン検知・再起動頻度アラート |

## サーバー設定

`manifests/deployment.yaml` の `env` を編集して反映する。設定変更後は Flux の apply または `kubectl apply -k manifests/` で適用。

詳細は [CLAUDE.md](./CLAUDE.md) を参照。
