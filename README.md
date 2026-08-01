# k3s-apps

k3s の上にセルフホストのアプリを載せるときの参考マニフェスト集。

1 アプリ = 1 ディレクトリ = 1 namespace で完結していて、`kubectl apply -f <name>/` だけで立ち上がる。Helm も Kustomize も使っていない素の YAML なので、必要なところをコピーして書き換える使い方を想定している。

| ディレクトリ | 構成 | 公開方法 |
| --- | --- | --- |
| [discourse](discourse) | Discourse + PostgreSQL + Redis + Sidekiq | `i.daco.dev` |
| [jitsi](jitsi) | Jitsi Meet（web / prosody / jicofo / jvb） | ⚠️ 未設定 + JVB は LB `10000/udp` |
| [misskey](misskey) | Misskey + PostgreSQL + Redis | ⚠️ `m.doany.io`（衝突） |
| [mstdn](mstdn) | Mastodon（web / streaming / sidekiq）+ PostgreSQL + Redis | ⚠️ `m.doany.io`（衝突） |
| [nextcloud](nextcloud) | Nextcloud | ⚠️ 未設定 |
| [openproject](openproject) | OpenProject（all-in-one） | `o.doany.io` |
| [speed](speed) | LibreSpeed | `s.doany.io` |
| [tinyproxy](tinyproxy) | HTTP プロキシ | LB `8888/tcp`（Ingress なし） |
| [watch](watch) | Prometheus + Grafana + image-renderer | `w.doany.io` / `w.daco.dev` |

## クラスタ側の前提

[5ym/bootstrap](https://github.com/5ym/bootstrap) で構築した k3s クラスタを前提にしている。別のクラスタで使う場合は、以下に対応する箇所を書き換える必要がある。

| 前提 | マニフェスト側の記述 |
| --- | --- |
| **Ingress** は Traefik CRD（`traefik.io/v1alpha1 IngressRoute`）。`networking.k8s.io/v1 Ingress` は使わない | `*/ingress.yaml` |
| **TLS** は certResolver `mydnschallenge`（Cloudflare DNS-01）。entryPoint は `websecure` のみで、`web` → `websecure` のリダイレクトは Traefik 側で設定済み | `tls.certResolver` |
| **StorageClass** は `local-path-retain`（`reclaimPolicy: Retain`）。単一ノード想定で `ReadWriteOnce` | `*/pvc.yaml` の `storageClassName` |
| **認証** は `auth` namespace の `forward-auth` / `basic-auth` Middleware を挟む（`allowCrossNamespace: true` 済み） | `routes[].middlewares` |
| **Secret** は SealedSecrets コントローラ（`kube-system`）で暗号化する | `*/secret.yaml` |

## ファイルの分け方

どのアプリも同じ構成にしてある。

| ファイル | 中身 |
| --- | --- |
| `namespace.yaml` | アプリ名と同じ namespace |
| `deployment.yaml` | アプリ本体と DB / Redis などを `---` 区切りで 1 ファイルに |
| `service.yaml` | 基本は ClusterIP。Traefik を通さないものだけ `LoadBalancer` |
| `pvc.yaml` | `local-path-retain`。誤削除防止に `argocd.argoproj.io/sync-options: Prune=false,Delete=false` を付けている |
| `secret.yaml` | プレースホルダ（`CHANGE_ME`）。SealedSecret 化してからコミットする |
| `ingress.yaml` | `IngressRoute`。外に出さないアプリには置かない |
| `configmap.yaml` | 設定ファイルを渡すものだけ（jitsi / watch） |

## デプロイ

namespace ごとに独立しているので、必要なものだけ apply する。

```sh
kubectl apply -f <name>/
```

ArgoCD で管理する場合は `<name>` を source path にした Application を各自作成する（このリポジトリには Application マニフェストは含めていない）。

## Secret の扱い

`*/secret.yaml` は **すべてプレースホルダ**（`CHANGE_ME`）で、そのままでは動かない。public リポジトリなので平文のままコミットしないこと。

```sh
# 1. 実値を入れたファイルをリポジトリ外に用意する
cp misskey/secret.yaml /tmp/misskey-secret.yaml
vim /tmp/misskey-secret.yaml

# 2. SealedSecret 化してリポジトリに戻す
./bootstrap/kubeseal.sh /tmp/misskey-secret.yaml misskey/secret.yaml
```

生成方法が決まっているものは各 `secret.yaml` の先頭コメントに書いてある（`openssl rand -hex 64`、`rake secret`、`php artisan key:generate` など）。

## 書くときにハマったところ

- **`local-path` の PVC は root:root で切られる** … イメージが非 root で動くものは `securityContext.fsGroup` を合わせないと書き込めない（Prometheus は 65534、Grafana は 472）。
- **PVC を持つ Deployment は `strategy: Recreate`** … `ReadWriteOnce` なので、既定の RollingUpdate だと新旧 Pod が同じ PVC を掴もうとして新 Pod が起動できない。
- **`postgres:18` は既定の `PGDATA` が変わっている** … `/var/lib/postgresql/data/pgdata` を明示している。既存データを移す場合は PVC 直下ではなく `pgdata/` の下に置くこと。
- **`IngressRoute` は match の長さで優先度が決まらない** … Mastodon の `/api/v1/streaming` のようにパスで振り分けるときは `priority` を明示する。
- **Traefik を通さない通信は ServiceLB で出す** … HTTP 以外（tinyproxy の TCP、JVB の UDP）は `type: LoadBalancer` + `externalTrafficPolicy: Local`。k3s の ServiceLB がノードのポートを直接開ける。
- **起動順を待つ仕組みは無い** … DB 待ちが要るものは `readinessProbe` を置いて、失敗中の再起動に任せる。
- **設定ファイルが秘密を含むなら ConfigMap ではなく Secret でマウントする** … Misskey の `default.yml` は DB パスワード入りなので Secret 側に置いている。
- **アプリ本体のディレクトリをボリュームで覆わない** … LibreSpeed の `/var/www/html` のように、イメージ内にアプリが入っている場所にマウントすると空になる。永続化するのはデータの場所だけ（`/database`）。

## 適用前に直すところ

- **ホスト名の衝突** … `m.doany.io` を [misskey](misskey) と [mstdn](mstdn) が両方使っている。同時に適用しない。
- **プレースホルダのままのホスト名** … [jitsi](jitsi)（`example.test`）、[nextcloud](nextcloud)（`nextcloud.example.com`）。`.test` や `.example.com` のままでは DNS-01 で証明書を取得できない。
- **[jitsi](jitsi) の `JVB_ADVERTISE_IPS` が空** … ノードの到達可能な IP を入れないとメディアが繋がらない。
- **[watch](watch) の scrape 対象** … `configmap.yaml` は最小構成しか入っていない。

## Renovate

[5ym/renovate-config](https://github.com/5ym/renovate-config) を継承し、`**/*.yaml` の `image:` を追跡する。
