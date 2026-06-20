---
title: "Config Compilada como Artefacto OCI: Atomicidad en Producción con Kargo"
weight: 1
tags: ["oci", "config-artifacts", "kargo", "argocd", "supply-chain", "gitops", "cosign", "ghcr", "helm", "starlark"]
date: "2026-06-20T00:00:00+00:00"
showDateUpdated: false
cascade:
  showDate: false
  showAuthor: false
  invertPagination: true
---

# Config Compilada como Artefacto OCI: Atomicidad en Producción con Kargo

{{< lead >}}
El problema completo de auto-sync + atomicidad + multi-region. Las cinco alternativas que evalué. La solución con artefactos OCI. El PromotionTask completo. Los resultados del test. Y por qué tu config compilada merece el mismo trato que tu imagen de Docker.
{{< /lead >}}

Llevo semanas resolviendo un problema que parecía simple y terminó siendo un rediseño de cómo la config compilada llega a producción. El resumen corto: auto-sync en producción + config como archivo en git + múltiples regiones = una bomba de tiempo. Aquí está todo lo que probé, lo que descarte, y lo que se convirtió en un ADR.

## El problema completo

Producción corre con ArgoCD en auto-sync. `selfHeal: true`, `prune: true`. Si algo en el repo de gitops cambia, **ArgoCD lo aplica, sin preguntar, sin esperar**.

Kargo controla las promociones, en staging, auto-promote, en producción, gate manual por región, US y CA se promueven de forma independiente. Cada región con su propio gate, su propio timing.

La compilación de config viene de Starlark. `service.star` + `deploy/prd-region.star` produce el values.yaml final: replicas, resources, probes, secrets, env vars, **todo**. Kargo solo actualiza `image.tag` via `yaml-update` durante la promoción.

El problema: si el pipeline de CD commitea el values.yaml compilado al path que ArgoCD vigila, el despliegue arranca antes de que Kargo promueva la imagen. Config nueva con imagen vieja. En staging no importa porque Kargo promueve en segundos. En producción con gate manual, la ventana es de horas.

Y hay otro problema, dos escritores al mismo repo: el pipeline de CD (config compilada) y Kargo (image tag). Sin transacción atómica. Si ambos escriben al mismo tiempo, conflictos de merge, si uno escribe antes, ventana de inconsistencia.

¿Y multi-region? US y CA necesitan artefactos de config diferentes, mismo servicio, misma versión, diferente región. Si los dos se promueven al mismo tiempo, git-push concurrente al mismo repo.

Tres problemas entrelazados, resolver uno sin los otros tres no sirve.

![ArgoCD Sync Policy](/images/oci-config-artifacts-kargo/argocd-sync-policy.png)
---

## Cinco alternativas, solo una sobrevivió

Antes de llegar a la solución, pasé por cuatro caminos que no funcionaron. Cada uno tenía mérito parcial y un defecto que me hizo descartarlo.

### A. Commit directo al path de producción (como staging)

Lo que ya teníamos en staging. El pipeline commitea config compilada al path que ArgoCD vigila, Kargo promueve después.

**Por qué no:** No es atómico. Config llega primero, imagen después. Auto-sync dispara despliegue con config nueva e imagen vieja, en staging funciona porque Kargo promueve en segundos pero en producción con gates manuales, la ventana puede ser de horas. Además, dos escritores al mismo repo sin coordinación.

### B. Directorio staged en el gitops repo

Commitear config compilada a un directorio intermedio (`/staged/prd/`) que ArgoCD no vigila. Kargo la copia al path final durante la promoción.

**Por qué no:** Mutable. Alguien puede editar el staged file entre el commit y la promoción. Contamina el historial de git con archivos intermedios, sin firmas, sin digest, path-filter frágil. Y sigues con dos escritores.

### C. Container custom en el cluster

Un step custom en Kargo que ejecuta el CLI que construimos desde Platform Engineering para compilar config desde Starlark al momento de la promoción, directamente en el cluster.

**Por qué no:** Carga operacional, cada actualización del CLI requiere rebuild del container. Acopla la promoción a una versión específica del CLI, amplía la superficie de ataque del cluster. El tooling de compilación no pertenece al plano de control de CD.

### D. workflow_dispatch de GitHub Actions desde Kargo

Un step custom en Kargo que dispara un workflow de GitHub Actions para commitear la config compilada al gitops repo.

**Por qué no:** Dependencia externa, asíncrono. La promoción en Kargo no sabe si el workflow terminó y no es atómico. Si GHA está caído, la promoción se cuelga, agregando latencia innecesaria o timeouts.

### E. Artefactos OCI de config (la que ganó)

Compilar config al momento del release, empaquetarla como artefacto OCI, subirla al registry con firma de Cosign. Kargo la descarga al momento de la promoción y la mete al gitops repo junto con el image tag en un solo commit atómico.

**Por qué ganó:** Inmutable, firmada y auditable. Un solo escritor al gitops repo (Kargo). Config e imagen en un commit. Cero ventana de inconsistencia. Sin tooling nuevo en el cluster. La misma infraestructura de registry que ya usas para imágenes.

{{< mermaid >}}
graph TD
    subgraph "Evaluación de alternativas"
        A["A: Commit directo<br/>(como staging)"] -->|No atómico| X[RECHAZADA]
        B["B: Directorio staged<br/>en gitops"] -->|Mutable, sin firmas| X
        C["C: devex-cli<br/>en cluster"] -->|Carga operacional| X
        D["D: GHA dispatch<br/>desde Kargo"] -->|Asíncrono, no atómico| X
        E["E: Artefactos OCI<br/>de config"] -->|Inmutable + atómico| V[ELEGIDA]
    end
{{< /mermaid >}}

---

## La arquitectura

Dos pipelines. Separación total de responsabilidades.

El pipeline de release (CD) compila, empaqueta y firma. Nunca toca el gitops repo. Kargo descarga, verifica, escribe y despliega. Un solo escritor mediante un commit atómico.

{{< mermaid >}}
sequenceDiagram
    participant CI as Release Pipeline
    participant GHCR as GHCR
    participant Kargo as Kargo (PRD)
    participant Git as devex-gitops
    participant Argo as ArgoCD

    Note over CI: semver tag -> pipeline arranca
    CI->>CI: retag imagen
    CI->>CI: compilar Starlark (todos los envs)
    CI->>GHCR: oras push config artifact
    CI->>GHCR: cosign sign (OIDC keyless)
    Note over GHCR: artefacto firmado, pinned por digest

    Note over Kargo: promoción manual aprobada
    Kargo->>GHCR: cosign verify (abort si falla)
    Kargo->>GHCR: oci-download @digest
    Kargo->>Git: copy values + yaml-update image.tag
    Kargo->>Git: git-commit (config + imagen, UN commit)
    Kargo->>Git: git-push
    Kargo->>Argo: argocd-update
    Argo->>Argo: auto-sync despliega
{{< /mermaid >}}

Lo importante: el pipeline de CD **nunca** escribe al devex-gitops, solo Kargo.

![Kargo Pipeline View](/images/oci-config-artifacts-kargo/kargo-pipeline-view.png)

---

## Patrón uniforme para todos los ambientes

El mismo flujo aplica para development, staging y producción. Lo que cambia es el trigger y el gate de promoción.

| Ambiente | Trigger | Tag del artefacto OCI | Promoción Kargo |
|----------|---------|----------------------|-----------------|
| dev | Push a main | `{svc}-sha-{hash}-dev-us-east-2` | Auto-promote |
| stg | Semver tag | `{svc}-v1.2.0-stg-us-east-2` | Auto-promote |
| prd-us | Semver tag | `{svc}-v1.2.0-prd-us-east-2` | Manual |
| prd-ca | Semver tag | `{svc}-v1.2.0-prd-ca-central-1` | Manual |

Un patrón, sin excepciones. Sin paths especiales para staging, sin workarounds para producción. Development y staging son el ensayo de exactamente lo que va a pasar en producción.

---

## El pipeline step: oras push + cosign

Esto va en tu workflow de release de GitHub Actions. Después de compilar la config con Starlark para cada región:

```bash
oras push \
  "ghcr.io/yourorg/devex-configs:${SERVICE}-${VERSION}-${REGION}" \
  --artifact-type application/vnd.devex.config.v1+json \
  "values.yaml:application/vnd.devex.config.values.v1+yaml"

DIGEST="$(crane digest ghcr.io/yourorg/devex-configs:${SERVICE}-${VERSION}-${REGION})"
cosign sign --yes "ghcr.io/yourorg/devex-configs@${DIGEST}"
```

Tres líneas. `oras push` sube el values.yaml como artefacto OCI con un media type custom. `crane digest` obtiene el digest exacto. `cosign sign` firma con OIDC del pipeline (Sigstore keyless, identidad del workflow de GHA).

El artefacto queda en el mismo GHCR donde ya tienes tus imágenes. Mismo auth, mismo garbage collection, mismo billing.

![GHCR Config Artifacts](/images/oci-config-artifacts-kargo/ghcr-config-artifacts.png)

---

## El PromotionTask completo de Kargo

Este es el PromotionTask que Kargo ejecuta para cada promoción a producción. Cada región tiene su propia instancia del Stage, pero comparten el mismo task.

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-prd-region
  namespace: devex-kargo
spec:
  vars:
    - name: service
    - name: region
    - name: imageRepo
    - name: gitopsRepo
      value: https://github.com/yourorg/devex-gitops.git
    - name: configRepo
      value: ghcr.io/yourorg/devex-configs
    - name: valuesPath
      value: services/${{ vars.service }}/${{ vars.region }}/values.yaml
    - name: version
      value: ${{ imageFrom(vars.imageRepo).Tag }}
    - name: configTag
      value: ${{ vars.service }}-${{ vars.version }}-${{ vars.region }}
  steps:
    # 1. Clonar gitops repo
    - uses: git-clone
      config:
        repoURL: ${{ vars.gitopsRepo }}
        checkout:
          - branch: main
            path: ./gitops

    # 2. Descargar artefacto OCI de config
    - uses: oci-download
      as: download-config
      config:
        imageRef: ${{ vars.configRepo }}:${{ vars.configTag }}
        mediaType: application/vnd.devex.config.values.v1+yaml
        outPath: ./config/values.yaml

    # 3. Copiar config al path del gitops repo
    - uses: copy
      config:
        inPath: ./config/values.yaml
        outPath: ./gitops/${{ vars.valuesPath }}

    # 4. Actualizar image.tag (del Freight)
    - uses: yaml-update
      config:
        path: ./gitops/${{ vars.valuesPath }}
        updates:
          - key: image.tag
            value: ${{ vars.version }}

    # 5. Commit atómico: config + image.tag en UNO
    - uses: git-commit
      as: commit
      config:
        path: ./gitops
        message: |
          chore(${{ vars.service }}): promote ${{ vars.region }} to ${{ vars.version }}

          config-artifact: ${{ vars.configRepo }}:${{ vars.configTag }}

    # 6. Push
    - uses: git-push
      config:
        path: ./gitops

    # 7. Sync ArgoCD apuntando al commit exacto
    - uses: argocd-update
      config:
        apps:
          - name: ${{ vars.service }}-${{ vars.region }}
            sources:
              - repoURL: ${{ vars.gitopsRepo }}
                desiredCommit: ${{ task.outputs.commit.commit }}
```

Siete pasos. Los que importan son dos: **oci-download** (descarga el artefacto de config desde el registry) y **git-commit** (config + image.tag en un solo commit atómico).

El commit message incluye el tag del artefacto. Cuando un auditor te pregunta "¿qué config se desplegó en esta promoción?", la respuesta está en el commit.

---

## Resultados del test

8 componentes validados antes de ir a producción:

| Componente                               | Status |
| ---------------------------------------- | ------ |
| oci-download con artefactos arbitrarios  | PASS   |
| copy sobrescribiendo archivos existentes | PASS   |
| ORAS -> oci-download compatibilidad      | PASS   |
| ArgoCD + Argo Rollouts                   | PASS   |
| argocd-update + multi-source             | PASS   |
| GHCR como OCI artifact store             | PASS   |
| Kargo auth a GHCR                        | PASS   |
| git-push concurrente (US + CA)           | PASS   |

El end-to-end completo: 7 pasos, ~5 segundos.

| Step | Alias           | Duración | Resultado                                 |
| ---- | --------------- | -------- | ----------------------------------------- |
| 1    | clone           | 1s       | Clonó devex-gitops                        |
| 2    | download-config | 1s       | Artefacto OCI descargado de GHCR          |
| 3    | apply-config    | <1s      | values.yaml reemplazado con contenido OCI |
| 4    | update-image    | <1s      | image.tag actualizado desde Freight       |
| 5    | commit          | <1s      | Commit atómico (config + image.tag)       |
| 6    | push            | 2s       | Push a branch                             |
| 7    | argocd-update   | <1s      | ArgoCD sincroniza al commit exacto        |

~5 segundos para una promoción completa. Config descargada, aplicada, commiteada junto con el tag de imagen, pusheada, y sincronizada. Sin humano en el loop y sin dependencias externas.

Un dato extra que costó un rato de debug: el timeout de `argocd-update` en staging no era un bug de Kargo (issue #4020 como sospechábamos inicialmente). Era un pull secret faltante en GHCR. Kyverno no estaba generando el secret en el namespace del servicio. ArgoCD intentaba sincronizar, el pod no podía jalar la imagen, timeout. La solución fue una ClusterPolicy de Kyverno, no un cambio en Kargo.

![Kargo Promotion Steps](/images/oci-config-artifacts-kargo/kargo-promotion-steps.png)

---

## Cambios de config a producción: solo `git tag`

Un cambio de config a producción **es** un release. Mismo pipeline, mismo flujo, cero paths especiales.

Ejemplo: necesitas subir replicas de 3 a 5 en un servicio para prd-us.

1. Cambias `deploy/prd-us-east-2.star` (replicas 3 -> 5)
2. PR, review, merge a main
3. `git tag v1.2.1` -> el pipeline de release corre
4. El retag de imagen es idempotente (mismo digest, misma imagen)
5. Se compila config nueva, se sube como artefacto OCI, se firma
6. Kargo crea Freight -> STG auto-promote (es un no-op, config de staging no cambió) -> PRD manual promote

El mismo pipeline que usas para un deploy con código nuevo funciona para un cambio de config sin código. El retag de imagen produce el mismo digest, el artefacto OCI tiene la config nueva, Kargo promueve cuando tú apruebas.

Sin workarounds, sin paths especiales, sin "editar el values.yaml directo y rezar".

---

## Trade-offs: OCI gitless vs direct push

Nada es gratis. Aquí está lo que ganas y lo que pagas.

| Aspecto                       | OCI Gitless                                        | Direct gitops push (como STG)                           |
| ----------------------------- | -------------------------------------------------- | ------------------------------------------------------- |
| Atomicidad                    | Config + imagen en un commit                       | Config llega primero, imagen después. Ventana de riesgo |
| Escritor al gitops            | Solo Kargo (un escritor)                           | CI + Kargo (dos escritores, conflictos posibles)        |
| Debugging                     | `oras pull` para inspeccionar config (skill nuevo) | `git show` (ya conocido)                                |
| Complejidad del PromotionTask | 6 steps (oci-download + copy + yaml-update + ...)  | 5 steps, uno menos                                      |
| Dependencias de CI            | ORAS en pipeline                                   | Nada nuevo                                              |
| Infra extra                   | OCI repo devex-configs + workflow de GC            | Nada                                                    |
| Inmutabilidad                 | Registry es append-only, firmable con Cosign       | Git commit es mutable (force push, amend)               |
| Rollback                      | `oras pull` de cualquier versión previa            | `git show` del commit, depende del historial de git     |

El costo real es uno: tu equipo necesita aprender a inspeccionar artefactos OCI. `oras pull` en vez de `git show`. No es difícil, pero es un skill nuevo. Si alguien necesita ver qué config se desplegó, ahora va al registry en vez de al historial de git.

El beneficio real también es uno: **certeza**. Sabes que lo que compilaste es exactamente lo que se desplegó. Sin ambiguedad, sin "¿alguien editó ese archivo entre el build y el deploy?".

---

## Escalabilidad: GHCR hoy, Harbor cuando lo necesites

¿Cuánto cuesta esto en almacenamiento? Casi nada, un values.yaml pesa kilobytes.

| Escala | Tags/mes | Almacenamiento/año | Registry |
|--------|----------|---------------------|----------|
| 20 servicios (hoy) | ~160 | ~4 MB | GHCR |
| 100 servicios | ~800 | ~20 MB | GHCR |
| 500 servicios | ~4k | ~100 MB | GHCR |
| 1k servicios | ~8k | ~1.9 GB | GHCR (monitorear cuota) |
| 3k+ servicios | ~24k | ~5.7 GB | Evaluar Harbor |

GHCR escala de manera seria, no necesitamos Harbor hoy, probablemente no lo necesitemos mañana, pero sabemos que está ahí en caso de que sí.

**¿Cuándo sí necesitas Harbor?**
- Regulación de residencia de datos que exige control de infraestructura
- Auditoría regulatoria que requiere immutabilidad de tags como requisito de compliance
- Multi-region artifact locality (Harbor replica US <-> CA)
- 500+ servicios con políticas de retención complejas

La migración es un cambio de URL por servicio en la variable `configRepo` del PromotionTask. Misma convención OCI, mismo flujo de Kargo, cero cambios en la lógica de promoción.

### Estrategia de limpieza de tags
No quieres acumular tags eternamente. Pero tampoco quieres borrar algo que necesitas para rollback.

| Regla                                   | Retención    | Razón                        |
| --------------------------------------- | ------------ | ---------------------------- |
| Actualmente desplegado en cualquier env | NUNCA borrar | Seguridad de rollback        |
| Promovido a PRD (alguna vez)            | >= 13 meses  | Ventana de auditoría SOC2    |
| Nunca promovido a PRD                   | 90 días      | Release candidates obsoletos |

Un workflow de GHA que corre semanal, consulta los Freight activos de Kargo, y limpia lo que no está en uso ni dentro de la ventana de retención.

---

## Compliance: audit trail de punta a punta

Si trabajas en healthcare, fintech, o cualquier industria regulada, esto te importa. Cada paso del flujo deja un registro verificable.

{{< mermaid >}}
graph TD
    A["release tag (vX.Y.Z)"] --> B["pipeline run<br/>(GHA run id, source commit SHA)"]
    B --> C["OCI push<br/>(digest sha256:..., cosign signature, SLSA L3)"]
    C --> D["Kyverno verify<br/>(admission policy valida firma)"]
    D --> E["Kargo promotion<br/>(Promotion CR: Freight id, approver, timestamp)"]
    E --> F["git commit<br/>(config-artifact tag)"]
    F --> G["ArgoCD sync<br/>(synced revision = ese commit)"]
{{< /mermaid >}}

Desde el tag de release hasta el sync de ArgoCD, cada eslabón apunta al anterior. Si un auditor te pregunta "¿qué exactamente se desplegó en producción el martes a las 3pm?", trazas la cadena completa.

El git commit en el gitops repo incluye el tag del artefacto, el tag apunta al artefacto en GHCR, el artefacto tiene una firma de Cosign con la identidad OIDC del pipeline, la firma apunta al workflow de GHA, el workflow apunta al commit de código fuente y el commit tiene el tag semver.

**Cadena completa, verificable y sin gaps.**

### Verificación de firmas: dónde y cómo

Cosign firma el artefacto en CD. ¿Dónde se verifica?

Kargo OSS no soporta steps custom con containers arbitrarios (eso es Akuity Platform Enterprise). Pero la verificación no necesita vivir dentro de Kargo. Hay dos opciones reales:

**Opción 1: Kyverno como admission policy.** Una ValidatingPolicy que intercepte las Promotions de Kargo y verifique la firma del artefacto OCI antes de admitirlas. Si la firma no es válida, la Promotion ni arranca. El enforcement queda en el control plane de Kubernetes, no en el pipeline.

**Opción 2: Verify en CI, confiar en el tag.** El pipeline de release firma con Cosign via OIDC. El tag del artefacto incluye la versión del release, y el push solo lo puede hacer el workflow de release (OIDC identity). Como el tag nunca se re-publica (un version = un tag = un digest), la inmutabilidad es por proceso. No es lo mismo que verificación criptográfica en la ruta de promoción, pero para muchos equipos es suficiente.

Nosotros validamos Cosign sign + verify localmente en durante la implementación. El round-trip funciona: `cosign sign` en CI, `cosign verify` con el issuer OIDC correcto. La pieza pendiente es el enforcement automático, que implementamos con Kyverno.

### Modos de falla

| Falla | Comportamiento |
|-------|---------------|
| Tag no existe o artefacto ausente | `oci-download` falla, Kargo aborta la promoción |
| git-push non-fast-forward | Promoción falla; concurrency limit previene la race condition |
| Artefacto sin firma (con Kyverno) | Kyverno rechaza la Promotion antes de que arranque |

Si Kargo dice que la promoción fue exitosa, el artefacto se descargó, la config y el image.tag se commitearon juntos, y ArgoCD sincronizó.

El aislamiento por región lo garantiza una ValidatingPolicy de Kyverno, no es solo una convención de naming. Si alguien intenta aplicar config de US en el namespace de CA, Kyverno lo rechaza a nivel de admission control.

---

## Precedente en la industria

Esto no es una idea nueva, es un patrón que la industria lleva años moviendo hacia mainstream.

| Fuente | Link | Insight |
|--------|------|---------|
| Google Cloud | [Config as OCI artifacts with Config Sync](https://cloud.google.com/blog/products/containers-kubernetes/gitops-with-oci-artifacts-and-config-sync) | Config como artefactos OCI, sync desde registry, no desde git |
| KubeCon EU 2025 | [Introduction to Gitless GitOps](https://dev.to/t-kikuc/introduction-to-gitless-gitops-a-new-oci-centric-and-secure-architecture-2pgi) | "Configuration is Part of the Supply Chain" |
| CNCF/Harbor | [Harbor as Universal OCI Hub](https://goharbor.io/blog/harbor-as-universal-oci-hub/) | Replicación multi-zona de config + imágenes + firmas |
| ArgoCD 3.1+ | [Native OCI source](https://argo-cd.readthedocs.io/en/stable/user-guide/oci/) | Soporte nativo de `oci://` como source de manifiestos K8s |
| Netflix/Spinnaker | [Declarative Delivery Configs](https://spinnaker.io/docs/guides/user/managed-delivery/delivery-configs/) | Promoción declarativa de artefactos inmutables entre ambientes |
| Flux CD | [Push OCI artifacts with Flux CLI](https://oneuptime.com/blog/post/2026-03-05-push-oci-artifacts-flux-cli/view) | `flux push artifact` para config como OCI |
| Flux CD | [Gitless GitOps with OCI](https://oneuptime.com/blog/post/2026-03-05-gitless-gitops-oci-artifacts-flux/view) | GitOps completo sin git como transport |
| codecentric | [Air-gapped GitOps with OCI](https://www.codecentric.de/en/knowledge-hub/blog/max-isolated-kubernetes-gitops-with-fluxcd-and-oci-repositories) | Deployments air-gapped via OCI |
| Kargo | [Community request for this pattern](https://github.com/akuity/kargo/issues/4864) | La comunidad pidió exactamente este patrón |
| Kargo | [oci-download step reference](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/oci-download) | Documentación del step que hace esto posible |

Google lo documenta, flux lo implementa, Harbor lo soporta nativamente, ArgoCD 3.1 lo trae de fábrica, Netflix lo hace desde hace años con Spinnaker y la comunidad de Kargo lo pidió explícitamente.

No es bleeding edge, es convergencia.

---
## Lo que queda

Config compilada no es un archivo, es build output. Merece el mismo tratamiento que una imagen de Docker: inmutable, versionada, firmada, verificada antes de desplegar.

Si tienes auto-sync en producción y tu config compilada vive como un archivo en git, la pregunta no es si vas a tener un incidente de atomicidad, es si en verdad quieres vivir cuando menos te lo esperes.

La solución no es apagar auto-syncn i dejar de compilar config, ni agregar path-filters frágiles. Es tratar tu config como lo que es: un artefacto de tu supply chain, empaquetarla, firmarla, dejar que tu controlador de promoción la descargue y la aplique atómicamente con la imagen.

Un pipeline, un patrón, todos los ambientes. Cero escritores extra al gitops repo, cero ventanas de inconsistencia, ~5 segundos de punta a punta.

`git tag v1.2.1`, lo demás es automático.
