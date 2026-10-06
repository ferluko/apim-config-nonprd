# CLAUDE.md — apim-config-nonprd

Repo de configuración **de prueba** del tier no productivo (ADR-gitops-apis-y-boveda §4–§5): lo que Argo CD aplica
en los clusters de APIM. **Público** (Argo lo lee por HTTPS anónimo: `paas-arqlab` no sale por el 22).

Contexto del programa: `~/Documents/Galicia/mc/doc/01_apim/RHCL-Kuadrant/api-toolkit-spec/PROMPT-claude-code-api-toolkit.md`.
Quién escribe acá: el **API Subscriber** (`~/Documents/Galicia/mc/bgal-api-sub`) y, en el futuro, el **API Publisher**.

## Árbol
| Path | Qué | Quién escribe |
|---|---|---|
| `catalog/plans.yaml` | Planes válidos (`bronze`, `gold`, `no-limits`) con su `maxTps`; el Subscriber elige el plan desde el TPS de la solicitud de ServiceNow (0.4.0) y rechaza planes fuera del catálogo (422) | Humano (PR) |
| `clusters/<c>/apis/<api-ns>/<api-id>/v<major>/` | Publicación de la API (AuthPolicy, RLP, HTTPRoute, …). En el lab solo hay un `README.md`: la API `poc-cred-api/greeting-echo` la desplegó la PoC; la carpeta define el placement | API Publisher |
| `clusters/<c>/subscriptions/<api-ns>/<api-id>/<consumer-ns>[--prev].yaml` | ExternalSecret de suscripción (y su `-prev` durante una rotación, fijado a una versión de Vault) | **Solo el API Subscriber** |
| `clusters/<c>/subscriptions/README.md` | Ancla del directorio: sin él, la última baja borraría la carpeta y el ExternalSecret quedaría vivo (ADR-gitops §5.2) | Nunca se borra |

## Reglas
- **No editar ni borrar a mano** archivos de `subscriptions/`: altas, cambios de plan y bajas van por la API (`POST/PATCH/DELETE /v1/subscriptions`). El Subscriber reconstruye su estado desde este repo (archivos + trailers de los commits): un cambio manual lo desincroniza de Vault y de la API Consumers.
- Los commits del Subscriber llevan trailers (`Apim-Operation`, `Apim-Kind`, `Apim-Target`, `Idempotency-Key`, `Request-Hash`, `Rotation-Id`, `Vault-Version`, …): son su base de datos. No reescribir historia: **nunca force-push, rebase de `main` ni squash**.
- Desde el Subscriber 0.3.0, cada operación tiene además un **commit de cierre** (`close:` o `rollback:`, trailers `Apim-Closes` y `Apim-Result`). Un `rollback:` es el Subscriber compensando un alta o un cambio de plan que no confirmó en todo el placement (borra el alta o vuelve al plan anterior): no es un cambio manual ni un error del repo.
- Nada de secretos: los ExternalSecrets solo referencian `consumers/<ns>/credentials`; la key la baja ESO.
- Manifiestos ya renderizados (sin Kustomize ni Helm en este repo).

## Argo (lab)
- ApplicationSet `apim-subscriptions` en `openshift-gitops` de `paas-arqlab` (manifiesto en `bgal-api-sub/deploy/argocd/lab/`): una Application `subs-paas-arqlab` sobre `clusters/paas-arqlab/subscriptions` (recurse), destino `apim-credentials`, AppProject que solo admite `ExternalSecret`.
- El Subscriber pide refresh dirigido tras cada push (espera de Argo 3–4 s; HALLAZGOS H-20).

## Estado conocido
Quedan vivas suscripciones `e2e-consumidor{2,3,4}-lab[-b]` del flujo 0.1 (credenciales dadas de alta por el Subscriber viejo). Limpiarlas con `DELETE /v1/subscriptions/sub-greeting-echo--<ns>` del Subscriber, no a mano.
