# Incident: クラスタダウン & Grafana PVC 権限エラー

- **発生日時**: 2026-03-31 07:11 UTC頃
- **検知**: Grafana に接続不能
- **影響範囲**: Kubernetes API サーバー含むクラスタ全体 + Grafana
- **復旧完了**: 2026-03-31 07:28 UTC頃

## 症状

1. `kubectl` コマンドが `dial tcp 192.168.1.100:6443: connect: host is down` で失敗
2. Grafana Pod が `Init:Error` で起動不能

## 経緯

Grafana に繋がらなくなっていたため、RPi5（コントロールプレーン）を手動で再起動した。再起動後もすぐには復旧せず、調査を開始。

## 原因

### 1. API サーバー一時停止（再起動後の etcd 不安定）

RPi5 手動再起動後、etcd が安定するまで API サーバーが機能しなかった。apiserver のコンテナ自体は起動していたが、etcd への接続失敗によりリクエストを処理できない状態だった。

- etcd が起動直後に `etcdserver: too many requests` を大量出力（raft 状態の回復中）
- etcd の leader election 完了後（term 5）、サービスは SERVING に復旧
- kube-apiserver は `talosctl services` に登録されなかったが、コンテナ自体は `CONTAINER_RUNNING`
- kube-apiserver が etcd（127.0.0.1:2379）への gRPC 接続に繰り返し失敗（`authentication handshake failed: context canceled`）
- kube-controller-manager / kube-scheduler は CrashLoopBackOff
- etcd 安定後、数分で API サーバーが自然回復し、全ノード Ready に

### 2. Grafana PVC 権限エラー

クラスタ復旧後も Grafana Pod は `Init:Error` のまま。

- `init-chown-data` コンテナ（root, CHOWN capability 付き）が PVC 上のディレクトリに `chown` できず `Permission denied`
- 対象: `/var/lib/grafana/png`, `/var/lib/grafana/csv`, `/var/lib/grafana/pdf`
- StorageClass: `local-path`
- 再起動時にホスト側のファイルシステム権限が壊れた可能性

## 復旧手順

### クラスタ復旧（RPi5 手動再起動 → 自然回復待ち）

```bash
# 1. ノードの疎通確認
ping 192.168.1.100  # OK
ping 192.168.1.101  # OK
ping 192.168.1.102  # OK

# 2. Talos サービス確認
talosctl -n 192.168.1.100 services
# → etcd, kubelet は Running だが kube-apiserver が一覧にない

# 3. etcd 状態確認
talosctl -n 192.168.1.100 etcd alarm list   # アラームなし
talosctl -n 192.168.1.100 etcd status       # DB 33MB, 正常

# 4. コンテナ確認 → kube-apiserver は実は CONTAINER_RUNNING
talosctl -n 192.168.1.100 containers -k | grep api

# 5. 数分待つと API サーバーが自然回復
kubectl get nodes  # 全ノード Ready
```

### Grafana PVC 復旧

```bash
# 1. Deployment をスケールダウン（PVC の finalizer 解放のため）
kubectl scale deployment -n monitoring monitoring-grafana --replicas=0

# 2. 壊れた PVC を削除（Pod 停止後に Terminating が完了する）
kubectl delete pvc -n monitoring monitoring-grafana

# 3. PVC を再作成
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: monitoring-grafana
  namespace: monitoring
  labels:
    app.kubernetes.io/instance: monitoring
    app.kubernetes.io/name: grafana
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: local-path
EOF

# 4. Deployment をスケールアップ
kubectl scale deployment -n monitoring monitoring-grafana --replicas=1

# 5. Pod が 3/3 Running になるまで待機
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -w
```

### HelmRelease Stalled 解消

```bash
# upgrade 失敗の履歴で Stalled になるため suspend/resume でリセット
flux suspend helmrelease monitoring -n monitoring
flux resume helmrelease monitoring -n monitoring

# 確認
kubectl get helmrelease -n monitoring monitoring
# → Ready: UpgradeSucceeded
```

## 教訓

1. **RPi5 再起動後は数分待つ** — etcd が安定するまで apiserver は機能しない。再起動自体はクラスタ復旧に有効だが、すぐに kubectl が通らなくても焦らない
2. **local-path PVC はノード再起動時に権限が壊れることがある** — Grafana データの永続化には注意が必要。バックアップやダッシュボードの GitOps 管理（ConfigMap/Sidecar）を検討する
3. **etcd 起動直後の "too many requests" は一時的** — 数分待てば自然回復する。慌てて etcd を操作しない
4. **HelmRelease が Stalled になったら suspend/resume** — `flux reconcile` だけでは Stalled はリセットされない
5. **PVC 削除時は先に Deployment をスケールダウン** — Pod が PVC を掴んでいると `kubernetes.io/pvc-protection` finalizer で Terminating のまま止まる
