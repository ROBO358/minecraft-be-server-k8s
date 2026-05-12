# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

このリポジトリは **homelab-gitops (`ROBO358/homelab-gitops`) の `apps/` レイヤに接続する Minecraft Bedrock Edition サーバーのアプリリポジトリ**。

- GitHub リポジトリ: `ROBO358/minecraft-be-server-k8s`
- コンテナイメージ: `itzg/minecraft-bedrock-server` (Docker Hub)
  - Java Edition 用の `itzg/minecraft-server` とは**別イメージ**。Bedrock には必ずこちらを使うこと。
- デプロイ先 namespace: `minecraft-be`（namespace 自体は homelab-gitops 側で管理）

## リポジトリ構成

```
manifests/                          # Kubernetes マニフェスト（Flux が直接 apply する）
  kustomization.yaml
  deployment.yaml
  service.yaml
  pvc.yaml
  allowlist.json                    # ホワイトリスト（ConfigMap として apply される）
  ciliumnetworkpolicy.yaml
  prometheusrule.yaml
  externalsecret-playit.yaml        # playit.gg SECRET_KEY（1Password → ESO）
  deployment-playit.yaml            # playit.gg agent
  ciliumnetworkpolicy-playit.yaml   # playit-agent の egress ポリシー
Taskfile.yml                        # ワールドデータ操作タスク
```

このリポジトリにはカスタムコンテナイメージがなく、CI/CD はない。Flux が `manifests/` を直接 apply する。

## manifests/ の各リソース

| ファイル | 内容 |
|---|---|
| `deployment.yaml` | `itzg/minecraft-bedrock-server:stable` を 1 レプリカで起動。strategy は **Recreate**（RWO PVC 制約 + 同時 2 台起動禁止のため必須） |
| `service.yaml` | `type: LoadBalancer`、UDP 19132（IPv4）と UDP 19133（IPv6）を公開。Cilium LB IPAM で `192.168.1.103` を固定割り当て |
| `pvc.yaml` | Longhorn 10Gi PVC でワールドデータ（`/data`）を永続化 |
| `allowlist.json` | プレイヤーホワイトリスト。`configMapGenerator` で ConfigMap 化され `/data/allowlist.json` にマウント。**変更すると Pod が再起動される**（Kustomize のハッシュ更新による） |
| `ciliumnetworkpolicy.yaml` | minecraft-be の ingress/egress ポリシー |
| `prometheusrule.yaml` | MinecraftBEDown / MinecraftBEPodRestarting アラート（kube-state-metrics ベース） |
| `externalsecret-playit.yaml` | 1Password `minecraft-be/playit-secret-key` から `Secret/playit-secret` を生成 |
| `deployment-playit.yaml` | `ghcr.io/playit-cloud/playit-agent:0.17` を 1 レプリカで起動 |
| `ciliumnetworkpolicy-playit.yaml` | playit-agent の egress: DNS / playit.gg cloud (TCP 443, UDP 5500-5600) / minecraft-be pod (UDP 19132) |

## itzg/minecraft-bedrock-server の重要な仕様

- **サーバーバイナリは起動時に Mojang からダウンロード**される。コンテナイメージにはバイナリが含まれていない。インターネット egress が必須。
- `EULA=TRUE` の設定が必須。設定しないとサーバーが起動しない。
- `VERSION=LATEST`（デフォルト）でコンテナ再起動時に最新版へ自動アップグレードされる。固定する場合は `VERSION=1.21.x` のように指定。
- ポートは **UDP**。TCP ではない。Service / CiliumNetworkPolicy 両方で `protocol: UDP` が必要。
- `UID=1000` / `GID=1000` + `fsGroup: 1000` で非 root 実行。PVC の所有権はコンテナ起動時に自動調整される。
- サーバープロパティは環境変数（`SERVER_NAME`, `GAMEMODE`, `DIFFICULTY` 等）で設定できる。`server.properties` を直接編集してもよいが再起動で上書きされるため環境変数推奨。

## サーバー設定の変更方法

`deployment.yaml` の `env` セクションに環境変数を追加・変更する。主要な変数:

| 変数 | 説明 |
|---|---|
| `SERVER_NAME` | サーバー名（クライアントのサーバーリストに表示） |
| `GAMEMODE` | `survival` / `creative` / `adventure` |
| `DIFFICULTY` | `peaceful` / `easy` / `normal` / `hard` |
| `MAX_PLAYERS` | 最大同時接続数 |
| `ONLINE_MODE` | `true` で Xbox 認証必須、`false` でオフラインモード |
| `ALLOW_LIST` | `true` でホワイトリスト制限。`/data/allowlist.json` で管理 |
| `VIEW_DISTANCE` | チャンク描画距離 |
| `LEVEL_SEED` | ワールドシード値 |

## LB IP の変更

`service.yaml` の `io.cilium/lb-ipam-ips: 192.168.1.103` を IPAM プールの空き IP に変更すること。

## playit.gg による外部公開

playit.gg の agent を別 Deployment として動かし、友人がインターネット経由で接続できるようにしている。

### 構成

```
友人の PC → playit.gg cloud → playit-agent Pod → minecraft-be Service (UDP 19132) → minecraft-be Pod
```

- agent は playit.gg cloud にアウトバウンド接続するだけ。インバウンドポート開放不要。
- Minecraft server への転送先は playit.gg ダッシュボード上で ClusterIP（`10.103.116.210:19132`）または DNS 名（`minecraft-be.minecraft-be.svc.cluster.local:19132`）として設定する。

### 初回セットアップ手順

1. [playit.gg](https://playit.gg) でアカウント作成
2. ダッシュボードで「New Agent」→ Docker を選択し `SECRET_KEY` を生成
3. 1Password に `minecraft-be/playit-secret-key` というアイテムを作成し、`SECRET_KEY` の値を保存
4. Flux が apply した後、agent Pod が起動して playit.gg ダッシュボードにエージェントが表示される
5. ダッシュボードでトンネルを作成:
   - Protocol: **UDP**
   - Local address: `minecraft-be:19132`
6. 払い出された公開アドレス（例: `abc123.mc.playit.gg:19132`）を友人に共有

### SECRET_KEY の管理

`externalsecret-playit.yaml` が 1Password から `Secret/playit-secret` を生成する。
1Password のアイテム名: `minecraft-be/playit-secret-key`（`key` フィールドに値を入れること）。

## homelab-gitops との責任分界

| 責務 | 配置先 |
|---|---|
| Namespace, PodSecurity | homelab-gitops `apps/minecraft-be/namespace.yaml` |
| RBAC（SA / ClusterRoleBinding） | homelab-gitops `apps/minecraft-be/rbac.yaml` |
| GitRepository / Flux Kustomization 定義 | homelab-gitops `apps/minecraft-be/source.yaml` |
| サーバー workload（Deployment / Service / PVC 等） | **このリポジトリ** `manifests/` |

## Taskfile タスク

| タスク | 内容 |
|---|---|
| `task upload-world` | ローカルのワールドデータを PVC にアップロード |
| `task download-world` | PVC からワールドデータをローカルにダウンロード |
| `task reset-world` | PVC 上のワールドデータを削除（確認プロンプトあり） |

いずれも scale→0 → world-manager Pod 起動 → 操作 → scale→1 の流れで動作する。
途中失敗時も `defer` により scale-up と Pod 削除が確実に実行される。

## CiliumNetworkPolicy の注意点

egress でインターネット（`world`）への HTTPS を許可しているのは、起動時にサーバーバイナリを `minecraft.azureedge.net` 等からダウンロードするため。この egress ルールを削除するとサーバーが起動しない。
