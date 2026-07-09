# classlab-k8s

Manifestes Kustomize de déploiement de l'application **Udagram** (frontend + 3 backends +
reverseproxy + Postgres). Moitié « k8s » du couple dual-repo : le code et la CI sont dans
`classlab-code` ; ici on décrit le déploiement. Synchronisé par ArgoCD (repo `classlab-infra`).

## Structure

```
base/                  # commun aux 3 envs
  frontend.yaml        # SPA (nginx)
  backend-user.yaml  backend-feed.yaml  backend-image-filter.yaml
  reverseproxy.yaml    # nginx, route /api/v0 -> backends
  cnpg-cluster.yaml    # Postgres (opérateur CloudNativePG)
  app-configmap.yaml   # config non secrète (URL/CORS, AWS_*)
  httproute.yaml       # expose via le Gateway classlab, sur *.classlab.lan
  servicemonitors.yaml # scrape Prometheus des backends
  kustomization.yaml   # ressources + images surveillées
overlays/
  dev/  staging/  prod/ # namespace, tags d'image, hostname, JWT (SealedSecret)
                        # prod ajoute hpa.yaml et passe Postgres en 3 instances
```

## Utilisation

Rien ne s'applique à la main. ArgoCD crée une Application par env et synchronise
`overlays/<env>` ; un `git push` = un déploiement. Les `newTag` sont réécrits par Argo CD
Image Updater à chaque build.

Valider un env avant commit :

```bash
kubectl kustomize overlays/prod
```

Patcher un env : entrée `patches:` dans `overlays/<env>/kustomization.yaml`, ciblée par
`kind` + `name`. Hostname et URL CORS sont déjà patchés par env ; prod patche aussi le
nombre d'instances Postgres.

Correspondance branche → env (via l'Image Updater) :

| Overlay | Namespace | Branche | Tag épinglé |
|---------|-----------|---------|-------------|
| `dev`     | `classlab-code-dev`     | `dev`          | `dev-<sha>`     |
| `staging` | `classlab-code-staging` | `staging`      | `staging-<sha>` |
| `prod`    | `classlab-code-prod`    | `master`/`main`| `main-<sha>`    |

## Secrets

Le JWT de chaque env est un SealedSecret (`overlays/<env>/jwt-sealedsecret.yaml`), chiffré
dans Git et déchiffré en cluster par sealed-secrets. Voir `classlab-infra` pour `kubeseal`.
