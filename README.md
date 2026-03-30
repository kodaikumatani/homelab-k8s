# homelab-k8s

Talos Linux K8s cluster on Raspberry Pi 5 × 3.

## Nodes

| Node | Role | IP |
|------|------|----|
| cp-1 | Control Plane | 192.168.1.100 |
| worker-1 | Worker | 192.168.1.101 |
| worker-2 | Worker | 192.168.1.102 |

## Cluster Architecture

| Component | Value |
|-----------|-------|
| OS | Talos Linux v1.12.6 |
| Kubernetes | v1.35.2 |
| Container Runtime | containerd 2.1.6 |
| CNI | Flannel v0.27.4 |
| Proxy | kube-proxy (iptables) |
| DNS | CoreDNS |
| Storage | EPHEMERAL on NVMe (VolumeConfig), boot on SD card |

## Prerequisites

- Raspberry Pi 5 × 3
- SD card × 3
- [talosctl](https://www.talos.dev/latest/introduction/getting-started/#talosctl)
- kubectl

## Setup

### 1. secrets 生成 (初回のみ)

```bash
talosctl gen secrets -o secrets.yaml
```

### 2. config 生成

```bash
talosctl gen config homelab-k8s https://192.168.1.100:6443 \
  --with-secrets secrets.yaml \
  --install-disk /dev/nvme0n1 \
  -o . -f
```

生成されるファイル: `controlplane.yaml`, `worker.yaml`, `talosconfig`

### 3. talosconfig の設定

```bash
talosctl config merge talosconfig
# context 名を確認して切り替え
talosctl config context <context-name>
talosctl config endpoint 192.168.1.100
talosctl config node 192.168.1.100
```

### 4. config apply (各ノードがメンテナンスモードで起動している状態で)

```bash
# Control Plane
talosctl apply-config -n 192.168.1.100 -f controlplane.yaml \
  --config-patch @patches/volume-ephemeral-nvme.yaml --insecure

# Worker 1
talosctl apply-config -n 192.168.1.101 -f worker.yaml \
  --config-patch @patches/volume-ephemeral-nvme.yaml --insecure

# Worker 2
talosctl apply-config -n 192.168.1.102 -f worker.yaml \
  --config-patch @patches/volume-ephemeral-nvme.yaml --insecure
```

### 5. bootstrap (初回のみ、CP で実行)

```bash
talosctl bootstrap -n 192.168.1.100
```

### 6. kubeconfig 取得

```bash
talosctl -n 192.168.1.100 kubeconfig --force
```

### 7. Worker ノードにロールラベルを付与

```bash
kubectl label node talos-7pd-pv6 node-role.kubernetes.io/worker=
kubectl label node talos-7sd-211 node-role.kubernetes.io/worker=
```

### 8. 動作確認

```bash
# ノード確認
kubectl get nodes

# EPHEMERAL が NVMe にあることを確認
talosctl -n 192.168.1.100 get discoveredvolumes | grep EPHEMERAL

# etcd 安定確認
talosctl -n 192.168.1.100 service etcd
```

### ノードの再セットアップ

SD カード入れ替えなどで再セットアップする場合:

```bash
# STATE/EPHEMERAL をワイプ (BOOT は残るので SD カードブートは壊れない)
talosctl -n <node-ip> reset --system-labels-to-wipe STATE,EPHEMERAL \
  --reboot --graceful=false

# メンテナンスモードで起動後、config apply
talosctl apply-config -n <node-ip> -f <controlplane|worker>.yaml \
  --config-patch @patches/volume-ephemeral-nvme.yaml --insecure
```

---

## Troubleshooting: RPi5 + Talos で etcd が不安定になる問題

### 症状

- etcd のヘルスチェックが 1〜2 分おきに OK/Fail を繰り返す (flapping)
- kube-controller-manager / kube-scheduler が CrashLoopBackOff
- ManifestApplyController が etcd タイムアウトで RBAC マニフェストを適用できない
- kubelet が Authorization error でノード登録できない
- 最悪の場合、kube-apiserver のプロセスが D (disk sleep) でハング

### 根本原因

**RPi5 の SBC イメージ (dd) では `machine.install.disk` の設定が無視される。**

Talos は SD カードに `dd` で焼いた時点で「インストール済み」と判断し、SD カードをシステムディスクとして使い続ける。`install-disk: /dev/nvme0n1` を指定しても、EPHEMERAL パーティション（etcd データ、コンテナイメージ）は SD カード上に作成される。

SD カードの I/O 性能が低いため etcd の fsync がタイムアウトし、ヘルスチェックが flapping → RBAC が適用できない → API サーバーが認証を通さない、という連鎖的な障害が発生する。

```
# 実際の確認方法
$ talosctl -n <node> get systemdisk
# → mmcblk0 と表示されたら SD カードで動作中

$ talosctl -n <node> mounts | grep /var
# → /dev/mmcblk0pX なら SD カード上で etcd が動作
```

### 解決策: VolumeConfig で EPHEMERAL を NVMe に配置

`patches/volume-ephemeral-nvme.yaml` を作成:

```yaml
apiVersion: v1alpha1
kind: VolumeConfig
name: EPHEMERAL
provisioning:
  diskSelector:
    match: disk.transport == 'nvme' && !system_disk
  grow: true
  minSize: 2GiB
```

適用手順:

```bash
# 1. 既存の EPHEMERAL をワイプ (BOOT は残るので SD カードブートは壊れない)
talosctl -n <node> reset --system-labels-to-wipe STATE,EPHEMERAL --reboot --graceful=false

# 2. メンテナンスモードで起動後、VolumeConfig パッチ付きで apply
talosctl apply-config -n <node> -f controlplane.yaml \
  --config-patch @patches/volume-ephemeral-nvme.yaml --insecure

# 3. bootstrap (CP のみ、初回のみ)
talosctl bootstrap -n <cp-node>
```

適用後の確認:

```bash
# EPHEMERAL が NVMe にあることを確認
$ talosctl -n <node> get discoveredvolumes | grep EPHEMERAL
# → nvme0n1pX  EPHEMERAL

# /var が NVMe にマウントされていることを確認
$ talosctl -n <node> mounts | grep /var
# → /dev/nvme0n1pX  /var
```

### 結果

| 項目 | 変更前 (SD カード) | 変更後 (NVMe) |
|------|-------------------|--------------|
| EPHEMERAL の場所 | mmcblk0 (SD) | nvme0n1 (NVMe) |
| etcd flapping | 1〜2分おきに OK/Fail | なし |
| SD カードの速度要件 | 高性能必須 | 低速でも OK |
| EPHEMERAL 容量 | 62 GB | 254 GB |

### 注意事項

- `talosctl reset` (通常) は SD カードのブートイメージを破損する。**必ず `--system-labels-to-wipe STATE,EPHEMERAL`** を使うこと
- SBC イメージ (dd) では `machine.install.disk` は効果がない。NVMe を使うには VolumeConfig が必須
- VolumeConfig は EPHEMERAL が未作成の状態でのみ適用される。既存の EPHEMERAL がある場合は reset が必要
- `secrets.yaml` は一度生成したら再生成しない
