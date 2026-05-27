# operator-upgrade-cronjob-sample

OpenShift で `installPlanApproval: Manual` に設定された Operator Subscription について、
**事前に許可リストに登録した Subscription の z (patch) リリースアップグレードだけ** を、
指定スケジュールで自動承認する CronJob のサンプルマニフェスト一式。

## 何をするか / しないか

- する: 許可リストに載った Subscription の `status.installPlanRef` をたどり、
  同一 channel 内で MAJOR/MINOR が変わらず PATCH のみ上がる InstallPlan を `spec.approved=true` に patch する
- する: それ以外 (許可リストに無い・y-stream や major bump・downgrade・初回インストール・既に承認済) は skip + ログ
- しない: 全 Operator の自動承認 (= 安全側のデフォルト)
- しない: アップグレード前提条件 (PreflightCheck) の検証、ロールバック

「許可リスト × スケジュール × z-release ガード」の三重チェックで承認する。

## 構成要素

| ファイル | 内容 |
|---|---|
| `manifests/00-namespace.yaml` | Namespace `operator-upgrade-approver` |
| `manifests/10-serviceaccount.yaml` | ServiceAccount `installplan-approver` |
| `manifests/20-clusterrole.yaml` | ClusterRole (subscriptions: get/list、installplans: get/list/patch) |
| `manifests/21-clusterrolebinding.yaml` | ClusterRoleBinding |
| `manifests/30-configmap-allowlist.yaml` | 許可リスト (運用者が編集する箇所) |
| `manifests/31-configmap-script.yaml` | 承認スクリプト `approve.sh` |
| `manifests/40-cronjob.yaml` | CronJob 本体 (`schedule` は TODO プレースホルダ) |

## 前提

- OpenShift 4.x クラスター
- 初期投入は `cluster-admin` 相当の権限が必要 (ClusterRole / ClusterRoleBinding の作成のため)
- 実行用イメージは `registry.access.redhat.com/ubi10/ubi:latest`
  (Pull に追加認証は不要、tar / gzip 同梱)
- Job Pod から OpenShift mirror (`https://mirror.openshift.com`) へ HTTPS で
  アクセスできること。UBI には `oc` が同梱されないため、起動時に
  `openshift-client-linux.tar.gz` を取得して `/tmp/bin/oc` に展開する。
  別 URL を指定したい場合は CronJob の env に `OC_CLIENT_URL` を設定する

## デプロイ手順

```sh
oc apply -f manifests/
```

### 1. 許可リストを編集

`manifests/30-configmap-allowlist.yaml` の `data.allowlist` に、自動承認対象とする
Subscription を `<namespace>/<subscription-name>` の書式で 1 行 1 件で記入し、再 apply する。
あるいは cluster 上で直接編集する:

```sh
oc edit configmap installplan-approver-allowlist -n operator-upgrade-approver
```

例:

```
openshift-logging/cluster-logging
openshift-operators/grafana-operator
```

### 2. スケジュールを設定

`manifests/40-cronjob.yaml` の `spec.schedule` はプレースホルダ (`"0 17 * * 5"`) になっている。
メンテナンス時間に合わせて編集して再 apply するか、cluster 上で直接編集する:

```sh
oc edit cronjob installplan-approver -n operator-upgrade-approver
```

OpenShift 4.14 以降 (Kubernetes 1.27 以降) なら `spec.timeZone: "Asia/Tokyo"` を有効化して
ローカルタイムでスケジュール指定できる。

## 動作確認 (手動キック)

スケジュールを待たずに 1 回だけ実行する:

```sh
oc create job --from=cronjob/installplan-approver manual-run-1 -n operator-upgrade-approver
oc logs -n operator-upgrade-approver job/manual-run-1
```

ログでは Subscription ごとに以下のいずれかが出る:

- `approving <ns>/<sub>: <from-csv> -> <to-csv> (...)` → 承認実行
- `approved <ip-ns>/<ip-name>` → patch 成功
- `skip <ns>/<sub>: ...` → skip 理由付きで未承認のまま
- `WARN  skip <ns>/<sub>: y-stream / major upgrade detected on channel '...': <from> -> <to>` → z-release ガードで弾かれた

最後に `done: approved=<n> skipped=<n> errors=<n>` のサマリ行が出る。
`errors > 0` の場合は Job が失敗 (exit 1) する。

## トラブルシュート

- **`subscription not found: <ns>/<sub>`**: 許可リストの記述ミス、または対象 Subscription が削除されている
- **`skip ...: no pending installPlanRef`**: 承認待ちの InstallPlan が現在は無い (正常)
- **`skip ...: installPlanApproval is 'Automatic'`**: 当該 Subscription が Manual ではない (本ツールの対象外)
- **`skip ...: y-stream / major upgrade detected`**: メジャー/マイナーが動くアップグレードはここで弾く。
  y-stream を上げたい場合は本ツール外で人手承認する
- **`forbidden` / RBAC エラー**: ClusterRoleBinding が適用されていない可能性。
  `oc auth can-i patch installplan -n <ns> --as=system:serviceaccount:operator-upgrade-approver:installplan-approver` で確認

## アンインストール

```sh
oc delete -f manifests/
```

