# k3s-apps

Docker Swarm + Traefik で運用していた各アプリの compose を k3s 向けマニフェストに書き換えて 1 つにまとめたもの。統合元の 14 リポジトリは本リポジトリに集約後に削除済み。

| ディレクトリ | 統合元リポジトリ | 概要 | ホスト名 |
| --- | --- | --- | --- |
| [apps/cron](apps/cron) | `cron-swarm-compose` | 定期実行（旧 crond + docker exec） | — |
| [apps/discourse](apps/discourse) | `discourse-traefik-swarm` | Discourse + PostgreSQL + Redis + Sidekiq | `i.daco.dev` |
| [apps/guacamole](apps/guacamole) | `guacamole-traefik-swarm` | Apache Guacamole + guacd + PostgreSQL | ⚠️ 未設定 |
| [apps/jitsi](apps/jitsi) | `jitsi-traefik-swarm` | Jitsi Meet（web/prosody/jicofo/jvb） | ⚠️ 未設定 |
| [apps/koel](apps/koel) | `koel-traefik-swarm` | Koel + MySQL | ⚠️ `k.doany.io`（衝突） |
| [apps/lego](apps/lego) | `lego-compose` | lego による証明書取得（DNS-01 / Cloudflare） | — |
| [apps/misskey](apps/misskey) | `misskey-traefik-swarm` | Misskey + PostgreSQL + Redis | ⚠️ `m.doany.io`（衝突） |
| [apps/mstdn](apps/mstdn) | `mstdn-traefik-swarm` | Mastodon（web/streaming/sidekiq） | ⚠️ `m.doany.io`（衝突） |
| [apps/nextcloud](apps/nextcloud) | `nextcloud-traefik-swarm` | Nextcloud | ⚠️ 未設定 |
| [apps/openproject](apps/openproject) | `openproject-traefik-swarm` | OpenProject（all-in-one） | `o.doany.io` |
| [apps/rocketchat](apps/rocketchat) | `Rocketchat-traefik-swarm` | Rocket.Chat + MongoDB | ⚠️ 未設定 |
| [apps/speed](apps/speed) | `speed-traefik-swarm` | LibreSpeed | `s.doany.io` |
| [apps/tinyproxy](apps/tinyproxy) | `tinyproxy-swarm` | HTTP プロキシ | — (LB :8888) |
| [apps/watch](apps/watch) | `watch-traefik-swarm` | Prometheus + Grafana + image-renderer | `w.doany.io` / `w.daco.dev` |

## 前提

[5ym/bootstrap](https://github.com/5ym/bootstrap) で構築した k3s クラスタを前提にしている。

- **Ingress**: Traefik CRD (`traefik.io/v1alpha1 IngressRoute`)。`networking.k8s.io/v1 Ingress` は使わない。
- **TLS**: certResolver `mydnschallenge`（Cloudflare DNS-01）、entryPoint は `websecure` のみ。`web` → `websecure` のリダイレクトは Traefik 側で設定済み。
- **StorageClass**: `local-path-retain`（`reclaimPolicy: Retain`）。単一ノード想定で `ReadWriteOnce`。
- **認証**: 非公開にしたいものは `auth` namespace の `forward-auth` / `basic-auth` Middleware を挟む（`allowCrossNamespace: true` 済み）。
- **Secret**: SealedSecrets コントローラが `kube-system` に居る。

## デプロイ

namespace ごとに独立しているので、必要なものだけ apply する。

```sh
kubectl apply -f apps/<name>/
```

ArgoCD で管理する場合は `apps/<name>` を source path にした Application を各自作成する（このリポジトリには Application マニフェストは含めていない）。

## Secret の扱い

`apps/*/secret.yaml` は **すべてプレースホルダ**（`CHANGE_ME`）で、そのままでは動かない。public リポジトリなので平文のままコミットしないこと。

```sh
# 1. 実値を入れたファイルをリポジトリ外に用意する
cp apps/misskey/secret.yaml /tmp/misskey-secret.yaml
vim /tmp/misskey-secret.yaml

# 2. SealedSecret 化してリポジトリに戻す
./bootstrap/kubeseal.sh /tmp/misskey-secret.yaml apps/misskey/secret.yaml
```

旧リポジトリには `openproject` の `SECRET_KEY_BASE`、`tinyproxy` の user/pass、`Rocketchat` の設定などが平文で public に置かれていた。**移行時にすべて新しい値へローテーションすること。**

## 要対応（適用前に必ず直す）

- **ホスト名の衝突**
  - `k.doany.io` … `apps/koel` と、既存の [5ym/k3s-konomitv](https://github.com/5ym/k3s-konomitv) が同じホスト名を使う。
  - `m.doany.io` … `apps/misskey` と `apps/mstdn` が同じホスト名を使う（旧構成でも misskey が mstdn を置き換えた形なので、両方同時に適用しない）。
- **プレースホルダのままのホスト名** … `guacamole` / `jitsi`（`example.test`）、`nextcloud`（旧 `yourdoamin`）、`rocketchat`（`r.localhost`）。`.localhost` や `.test` のままでは DNS-01 で証明書を取得できない。
- **`apps/cron`** … `web` Deployment の namespace / 名前が不明なため `web` で仮置きしている。`rbac.yaml` と `cronjob.yaml` の TODO を実環境に合わせること。
- **`apps/jitsi`** … `JVB_ADVERTISE_IPS` が空。ノードの到達可能な IP を入れないとメディアが繋がらない。
- **`apps/lego`** … `job.yaml` / `cronjob.yaml` のドメインとメールアドレスがサンプルのまま。

## Swarm からの主な変更点

| 旧 (Swarm) | 新 (k3s) |
| --- | --- |
| `deploy.labels` の `traefik.http.routers.*` | `IngressRoute` CRD |
| `traefik.http.middlewares.*` | `Middleware` CRD |
| `certresolver=myresolver` | `certResolver: mydnschallenge` |
| 外部ネットワーク `main_default` への相乗り | namespace 分離 + Service 名で解決 |
| ホストパスの bind mount (`./app/data`) | `PersistentVolumeClaim`（`local-path-retain`） |
| `env_file` / `.env` | `Secret` + `envFrom` |
| 環境変数に平文パスワード | `Secret` + `secretKeyRef`（→ SealedSecret） |
| `ports: mode: host` | `Service` `type: LoadBalancer`（k3s ServiceLB） |
| `depends_on: condition: service_healthy` | `readinessProbe` + 再起動待ち |
| `init.sh`（curl して vim して stack deploy） | `kubectl apply -f apps/<name>/` |
| crontab + `docker exec` | `CronJob` + `kubectl exec`（ServiceAccount/RBAC） |

compose のままでは動かなくなっていた箇所も、移行にあわせて直している。

- **rocketchat** … `mongod --smallfiles` を削除（MongoDB 4.2 で撤廃済みのオプション。Renovate が `mongo:8.3` に上げた時点で起動しなくなっていた）。`MONGO_URL` も `local` DB ではなく `rocketchat` DB を指すよう修正。replSet 初期化は `job.yaml`。
- **speed** … `./speed:/var/www/html` はイメージ内のアプリを空ディレクトリで覆う指定だった。telemetry の SQLite が置かれる `/database` のみ永続化に変更。
- **postgres 系** … `PGDATA` を `/var/lib/postgresql/data/pgdata` に明示。`postgres:18` でイメージ既定のデータディレクトリが変わっており、旧 compose のマウント先では永続化されていなかった。既存データを移行する場合は PVC 直下ではなく `pgdata/` の下に置くこと。
- **watch** … `local-path` が PVC を root 所有で作るため、`securityContext.fsGroup` を Prometheus (65534) / Grafana (472) に合わせた。Prometheus は無認証で公開されていたので `forward-auth` を挟んでいる（不要なら `middlewares` ごと削除）。
- **jitsi** … 各コンテナは起動時に env から `/config` を生成し直すので、`/config` は PVC ではなく `emptyDir`。永続が必要な `transcripts` と `prosody-plugins-custom` だけ PVC。
- **openproject** … service 名の typo (`openprojct`) を修正。
- **cron** … 旧 crontab の `MAILTO` + ssmtp によるメール通知は移していない。

## Renovate

[5ym/renovate-config](https://github.com/5ym/renovate-config) を継承し、`apps/**/*.yaml` の `image:` を追跡する。
