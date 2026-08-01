# ClusterSecretStore per environment

Dev dan prod berbagi satu cluster EKS dan satu Vault. Yang memisahkan keduanya:

| | dev | prod |
|---|---|---|
| namespace k8s | `default` | `prod` |
| ClusterSecretStore | `vault-kv` | `vault-kv-prod` |
| role Vault | `eso` | `eso-prod` |
| path Vault | `secret/dev/*` | `secret/prod/*` |

Chart `app` memilih store lewat `vault.storeName`, dan nilainya dipaksa
workflow deploy (`--set`) dari input `env` — tidak bisa di-override dari file
values. Jadi tidak ada jalan untuk sebuah service di namespace `default`
membaca `secret/prod/*` selain dengan mengubah objek cluster-scoped ini.

## Terapkan

```sh
kubectl apply -f components/external-secrets/clustersecretstore-dev.yaml
kubectl apply -f components/external-secrets/clustersecretstore-prod.yaml
```

`clustersecretstore-dev.yaml` menambahkan `conditions` ke store yang sudah
jalan. Sebelum menerapkannya pastikan semua ExternalSecret yang memakai
`vault-kv` memang ada di namespace `default`:

```sh
kubectl get externalsecrets -A
```

## Sisi Vault (dijalankan sekali)

Role `eso-prod` belum ada — ini yang membuat `vault-kv-prod` bisa login.
Sekaligus persempit policy `eso` yang lama: selama ia masih boleh membaca
`secret/data/*`, pemisahan path dev/prod cuma konvensi penamaan.

```sh
kubectl -n vault exec -it vault-0 -- sh

vault policy write eso-dev - <<'EOF'
path "secret/data/dev/*"     { capabilities = ["read"] }
path "secret/metadata/dev/*" { capabilities = ["read", "list"] }
EOF

vault policy write eso-prod - <<'EOF'
path "secret/data/prod/*"     { capabilities = ["read"] }
path "secret/metadata/prod/*" { capabilities = ["read", "list"] }
EOF

# Role dev yang sudah ada dipersempit ke policy baru.
vault write auth/kubernetes/role/eso \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-dev ttl=1h

vault write auth/kubernetes/role/eso-prod \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-prod ttl=1h
```

Kedua role terikat ke ServiceAccount yang sama, jadi role bukan pemisah
identitas — ESO bisa mengambil token untuk dua-duanya. Yang menahan adalah
`conditions` di masing-masing store. Konsekuensinya: siapa pun yang bisa
membuat/mengubah ClusterSecretStore (objek cluster-scoped) bisa menembus
pemisahan ini. Batasi RBAC-nya ke cluster-admin.

## Isi secret prod

Setiap service butuh dua path, sama polanya dengan dev:

```sh
vault kv put -mount=secret prod/general \
  DB_HOST=... DB_USER=... DB_PASSWORD=... DB_PORT=3306

vault kv put -mount=secret prod/user-service-api \
  DB_NAME=... JWT_SECRET=...
```

`prod/general` dimuat lebih dulu dan jadi nilai dasar; field dengan nama sama
di `prod/<service>` menimpanya (lihat `dataFrom` di templates/externalsecret.yaml).
