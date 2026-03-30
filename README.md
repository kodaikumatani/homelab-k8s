# homelab-k8s

Talos Linux K8s cluster on Raspberry Pi 5 × 3.

## Nodes

| Node | Role | IP |
|------|------|----|
| cp-1 | Control Plane | 192.168.1.100 |
| worker-1 | Worker | 192.168.1.101 |
| worker-2 | Worker | 192.168.1.102 |

## Prerequisites

- Raspberry Pi 5 × 3
- SD card × 3
- [talosctl](https://www.talos.dev/latest/introduction/getting-started/#talosctl)
- kubectl

## Plan

### Step 1: Talos Linux のインストール

- [x] Talos Linux の Raspberry Pi 5 用イメージをダウンロード
- [x] 各 SD カードにイメージを書き込み
- [x] 各 Pi を有線 LAN に接続して起動
- [x] 各 Pi の IP アドレスを確認・固定

### Step 2: クラスタの初期設定

- [x] talosctl をインストール
- [x] secrets を生成 (`talosctl gen secrets`)
- [x] control plane / worker のコンフィグを生成 (`talosctl gen config`)
- [x] 各ノードにコンフィグを適用 (`talosctl apply-config`)
- [x] クラスタをブートストラップ (`talosctl bootstrap`)
- [x] kubeconfig を取得

### Step 3: クラスタの動作確認

- [ ] `kubectl get nodes` で全ノードが Ready になることを確認
- [ ] テスト用 Pod をデプロイして動作確認
- [ ] Pod が worker ノードにスケジュールされることを確認

### Step 4: VPN (WireGuard) の構築

- [ ] WireGuard を K8s 上にデプロイ
- [ ] ルーターでポートフォワーディング設定
- [ ] 外部から VPN 接続できることを確認

### Step 5: 運用・発展

- [ ] monitoring (Prometheus + Grafana) の導入
- [ ] GitOps (Flux or ArgoCD) の導入
- [ ] worker ノード障害時の挙動を検証

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
