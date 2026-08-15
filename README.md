# k3s-apps

k3s の上にセルフホストのアプリを載せるときの参考マニフェスト集。

1 アプリ = 1 ディレクトリ = 1 namespace = 1 ArgoCD Application で完結している。**どう配られるかは各ディレクトリの `argocd.yaml` で決まる**ので、クラスタ側にも [5ym/bootstrap](https://github.com/5ym/bootstrap) 側にも設定を持たせない。

本家が Helm chart を出しているアプリはその chart から出す。chart が無いアプリだけ素の YAML で書いてある。

| ディレクトリ | 出し方 | 構成 | 公開方法 |
| --- | --- | --- | --- |
| [discourse](discourse) | 素の YAML | Discourse + PostgreSQL + Redis + Sidekiq | `ds.doany.io` |
| [jitsi](jitsi) | 素の YAML | Jitsi Meet（web / prosody / jicofo / jvb） | `jt.doany.io` + JVB は LB `10000/udp` |
| [misskey](misskey) | 素の YAML | Misskey + PostgreSQL + Redis | `mk.doany.io` |
| [mstdn](mstdn) | Helm [`mastodon`](https://github.com/mastodon/helm-charts) + 素の DB | Mastodon（web / streaming / sidekiq）+ PostgreSQL + Redis | `mn.doany.io` |
| [nextcloud](nextcloud) | Helm [`nextcloud`](https://github.com/nextcloud/helm) | Nextcloud（SQLite） | `nd.doany.io` |
| [openproject](openproject) | Helm [`openproject`](https://github.com/opf/helm-charts) | OpenProject + PostgreSQL + memcached | `ot.doany.io` |
| [speed](speed) | 素の YAML | LibreSpeed | `sd.doany.io` |
| [tinyproxy](tinyproxy) | 素の YAML | HTTP プロキシ | LB `8888/tcp`（Ingress なし） |
| [watch](watch) | Helm [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts) | Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics | `ps.doany.io` / `gn.doany.io` / `ar.doany.io` |

ホスト名は **アプリ名の頭文字 + 発音上の最後の子音 + `.doany.io`** の 2 文字。綴りの末尾ではなく音で取るので、discourse は `de` ではなく `ds`、jitsi は `ji` ではなく `jt`、misskey は `my` ではなく `mk` になる。1 つのディレクトリから複数出すものはディレクトリ名ではなく中身の名前で付ける（[watch](watch) の `ps` = prometheus、`gn` = grafana、`ar` = alertmanager）。

Helm にしていないのは、**本家（配信元）が chart を出していない**もの。Discourse・Misskey・Jitsi・LibreSpeed・tinyproxy は公式 chart が無く、あるのは Bitnami や jitsi-contrib のような第三者のものだけなので、素の YAML のままにしてある。

## クラスタ側の前提

[5ym/bootstrap](https://github.com/5ym/bootstrap) で構築した k3s クラスタを前提にしている。別のクラスタで使う場合は、以下に対応する箇所を書き換える必要がある。

| 前提 | マニフェスト側の記述 |
| --- | --- |
| **ArgoCD** の ApplicationSet が各ディレクトリの `argocd.yaml` を読んで Application を組み立てる | `*/argocd.yaml` |
| **Ingress** は Traefik CRD（`traefik.io/v1alpha1 IngressRoute`）。`networking.k8s.io/v1 Ingress` は使わない | `*/ingress.yaml` |
| **TLS** は certResolver `mydnschallenge`（Cloudflare DNS-01）。entryPoint は `websecure` のみで、`web` → `websecure` のリダイレクトは Traefik 側で設定済み | `tls.certResolver` |
| **StorageClass** は `local-path-retain`（`reclaimPolicy: Retain`）。単一ノード想定で `ReadWriteOnce` | `*/pvc.yaml` の `storageClassName` |
| **認証** は `auth` namespace の `forward-auth` / `basic-auth` Middleware を挟む（`allowCrossNamespace: true` 済み） | `routes[].middlewares` |
| **Secret** は SealedSecrets コントローラ（`kube-system`）で暗号化する | `*/secret.yaml` |

## ファイルの分け方

どのアプリも同じ構成にしてある。

| ファイル | 中身 |
| --- | --- |
| `argocd.yaml` | この置き場の ArgoCD 配置。bootstrap の ApplicationSet が読む |
| `application.yaml` | Helm chart を指す `Application`。chart から出すアプリだけ |
| `deployment.yaml` | アプリ本体と DB / Redis などを `---` 区切りで 1 ファイルに |
| `service.yaml` | 基本は ClusterIP。Traefik を通さないものだけ `LoadBalancer` |
| `pvc.yaml` | `local-path-retain`。誤削除防止に `argocd.argoproj.io/sync-options: Prune=false,Delete=false` を付けている |
| `secret.yaml` | プレースホルダ（`CHANGE_ME`）。SealedSecret 化してからコミットする |
| `ingress.yaml` | `IngressRoute`。外に出さないアプリには置かない |
| `configmap.yaml` | 設定ファイルを渡すものだけ（jitsi） |

Namespace のマニフェストは持たない。ApplicationSet の `CreateNamespace` が `argocd.yaml` の `namespace` を見て作る。PVC には `Prune=false` を付けてあるので、Application を消しても中身ごと消えることは無い。

### `argocd.yaml`

```yaml
name: <アプリ名>          # Application の名前
namespace: <アプリ名>     # 配り先の namespace（CreateNamespace が作る）
sourcePath: <アプリ名>    # マニフェストの置き場（リポジトリルートからの相対）
recurse: false
autoSync: true           # false にすると UI からの手動 sync になる
prune: true
selfHeal: true
```

### `application.yaml`

ApplicationSet はディレクトリを「素のマニフェストの置き場」として読むだけなので、chart から出したいアプリはここに `Application` を 1 つ置いて chart を指させる（App-of-Apps）。**この家の値は `helm.valuesObject` にインラインで持つ。** 別ファイルにしないのは、ApplicationSet が `<name>/*.yaml` を全部マニフェストとして読むため（値ファイルは kind が無いので置けない）。

chart に入れられないもの（`IngressRoute`、`local-path-retain` の PVC、SealedSecret）は同じ階層に素のまま残して、chart 側からは `existingClaim` / `existingSecret` で引く。

## デプロイ

ArgoCD が `argocd.yaml` を見て勝手に配る。手で何かする必要は無い。

手元で中身を確かめたいときは、`argocd.yaml` を除いて apply する（`argocd.yaml` は kind を持たない設定ファイルなので、そのまま `kubectl apply -f <name>/` に渡すと落ちる）。

```sh
# chart を使っていないアプリ（argocd.yaml だけ外して apply する）
kubectl apply $(ls <name>/*.yaml | grep -v argocd.yaml | sed 's/^/-f /')

# chart を使っているアプリは helm template で展開してから見る
helm template <name> <chart> --repo <repoURL> --version <targetRevision> \
  -f <(yq '.spec.source.helm.valuesObject' <name>/application.yaml)
```

## Secret の扱い

`*/secret.yaml` は **すべてプレースホルダ**（`CHANGE_ME`）で、そのままでは動かない。public リポジトリなので平文のままコミットしないこと。

```sh
# 1. 実値を入れたファイルをリポジトリ外に用意する
cp misskey/secret.yaml /tmp/misskey-secret.yaml
vim /tmp/misskey-secret.yaml

# 2. SealedSecret 化してリポジトリに戻す
./bootstrap/kubeseal.sh /tmp/misskey-secret.yaml misskey/secret.yaml
```

生成方法が決まっているものは各 `secret.yaml` の先頭コメントに書いてある（`openssl rand -hex 64`、`rake secret`、`rails db:encryption:init` など）。

## 書くときにハマったところ

- **`local-path` の PVC は root:root で切られる** … イメージが非 root で動くものは `securityContext.fsGroup` を合わせないと書き込めない（Prometheus は 65534、Grafana は 472）。chart から出す場合は chart 側が既定で合わせてくれる。
- **kube-prometheus-stack は `ServerSideApply=true` が要る** … CRD が client-side apply の `last-applied-configuration` annotation の上限（262144 バイト）を超えるので、外すと sync が落ちる。
- **k3s には controller-manager / scheduler / proxy の scrape 先が無い** … 1 プロセスに畳まれていて、メトリクスの bind-address が既定で `127.0.0.1`。有効のままだと down のままアラートが鳴り続けるので落としてある。etcd も既定のデータストアが sqlite なので居ない。
- **Operator の PVC は `existingClaim` で渡せない** … Prometheus と Alertmanager は StatefulSet の `volumeClaimTemplate` からしか PVC を作れないので、`pvc.yaml` ではなく `application.yaml` 側に書く。StatefulSet が作った PVC は ArgoCD の管理外なので prune はされない。
- **PVC を持つ Deployment は `strategy: Recreate`** … `ReadWriteOnce` なので、既定の RollingUpdate だと新旧 Pod が同じ PVC を掴もうとして新 Pod が起動できない。
- **chart の PVC は `existingClaim` で外から渡す** … 多くの chart は共有ボリュームに `ReadWriteMany` を要求する（Mastodon の chart は `ReadWriteMany` 以外だと `fail` する）。`existingClaim` を渡すと chart は PVC を作らないので、この要求ごと回避できて `local-path-retain` の `ReadWriteOnce` がそのまま使える。
- **chart の Service 名は `fullnameOverride` で固定する** … 既定だとリリース名が前に付いて `IngressRoute` から引けない。port も chart 既定（多くは 80 や 8080）でコンテナの port とは違う。
- **`postgres:18` は既定の `PGDATA` が変わっている** … `/var/lib/postgresql/data/pgdata` を明示している。既存データを移す場合は PVC 直下ではなく `pgdata/` の下に置くこと。
- **`IngressRoute` は match の長さで優先度が決まらない** … Mastodon の `/api/v1/streaming` のようにパスで振り分けるときは `priority` を明示する。
- **Traefik を通さない通信は ServiceLB で出す** … HTTP 以外（tinyproxy の TCP、JVB の UDP）は `type: LoadBalancer` + `externalTrafficPolicy: Local`。k3s の ServiceLB がノードのポートを直接開ける。
- **JVB が広告する IP は Downward API で入れる** … `externalTrafficPolicy: Local` なのでメディアは Pod が載っているノードに直接着く。そのノードの IP を `status.hostIP` から `JVB_ADVERTISE_IPS` に入れているので、ノードの IP を直書きしなくて済む。NAT の外から使う場合だけ `configmap.yaml` にグローバル IP を書く。
- **ArgoCD では Helm の `pre-install` フックを当てにしない** … ArgoCD は `pre-install` も `pre-upgrade` も PreSync として毎回回す。Mastodon の `db:prepare`（`pre-install` のみ）は落としてあるので、初回だけ手で流すこと。
- **起動順を待つ仕組みは無い** … DB 待ちが要るものは `readinessProbe` を置いて、失敗中の再起動に任せる。
- **設定ファイルが秘密を含むなら ConfigMap ではなく Secret でマウントする** … Misskey の `default.yml` は DB パスワード入りなので Secret 側に置いている。
- **アプリ本体のディレクトリをボリュームで覆わない** … LibreSpeed の `/var/www/html` のように、イメージ内にアプリが入っている場所にマウントすると空になる。永続化するのはデータの場所だけ（`/database`）。

## 適用前に直すところ

- **すべての `secret.yaml`** … `CHANGE_ME` のままでは動かない。上の「Secret の扱い」の通り実値を入れて SealedSecret 化する。
- **[openproject](openproject) を素の構成から移す場合** … chart は PostgreSQL を別 Pod（サブチャート）で立てるので、PVC の名前もレイアウトも変わる。旧 PVC から `pg_dump` を取って新しい方に流し込むこと。
- **[jitsi](jitsi) を NAT の外から使う場合** … `JVB_ADVERTISE_IPS` はノードの IP（`status.hostIP`）が入る。グローバル IP を広告する必要があるなら `deployment.yaml` の `env` を消して `configmap.yaml` に直書きする。
- **[watch](watch) の Traefik の scrape** … k3s 同梱 Traefik の `HelmChartConfig` で `metrics.prometheus` を有効にしていない場合は、`application.yaml` の `additionalScrapeConfigs` を消すこと（有効でないと down のままになる）。
- **[watch](watch) の Alertmanager の通知先** … 既定の receiver は `null` でどこにも飛ばない。飛ばしたい先が決まったら `application.yaml` に `alertmanager.config` を足すこと。

## Renovate

[5ym/renovate-config](https://github.com/5ym/renovate-config) を継承し、`**/*.yaml` の `image:` と、`application*.yaml` の Helm chart の `targetRevision` を追跡する。
