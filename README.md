<div align="center">

# Aprendiendo GitOps desde ECS: Ruta a Kubernetes con ArgoCD & GitHub Actions

Repositorio principal (material de la charla KCD Colombia). Aquí centralizamos diagramas, conceptos, comandos y vínculos a los demás repos usados en la demo/migración.

</div>

## Tabla de Contenido
1. Objetivo & Alcance
2. Topología de Repositorios (GitOps Layering)
3. Flujo GitOps End‑to‑End
4. Migración: De ECS (Blue/Green) a EKS + ArgoCD
5. Infraestructura (Terraform) – orden de despliegue
6. Pre‑requisitos & Setup local
7. Pipeline CI/CD (GitHub Actions) & Triggers
8. Uso de Helm & Charts
9. ArgoCD: Apps, Sync y Rollbacks
10. DNS, Ingress y ALB
11. Comandos de Operación / Troubleshooting
12. Limpieza (Tear‑down)
13. Diagramas Referenciados
14. Buenas Prácticas y Lecciones
15. Siguientes Pasos / Extensiones

---

## 1. Objetivo & Alcance
Mostrar una ruta práctica para evolucionar un flujo de despliegue existente en **Amazon ECS (blue/green)** hacia un modelo **GitOps en Amazon EKS** usando **ArgoCD**, **Helm** y **GitHub Actions** con autenticación OIDC a AWS. El objetivo es que puedas replicar el stack completo y entender claramente la separación de responsabilidades por repositorio.

## 2. Topología de Repositorios
| Rol | Repositorio | Propósito Principal | Contenido Clave |
|-----|-------------|--------------------|-----------------|
| App (Code) | `kcd_colombia_webapp_color` | Código fuente, Dockerfile, workflows CI/CD | Flask app, Helm chart base, workflows (`cicd.yaml`) |
| Environment / Config | `kcd_colombia_env` | Valores Helm versionados por ambiente (image tags) | `clusters/prod/apps/webapp-color.values.yaml` |
| Platform / GitOps | `kcd_colombia_platform` | Definiciones ArgoCD (Apps, Projects, Addons) | `clusters/prod/root-app.yaml`, addons (ALB controller, external-dns) |
| Infra (Provisioning) | `kcd_colombia_infra` | Terraform para VPC, ECR, EKS, IAM, Route53, ACM, etc. | `terraform/{core,eks,apps}` módulos |
| Docs (este) | `kcd_colombia` | Diagramas, guía de la charla, explicación conceptual | `*-flowchart.md`, `webapp-responsabilidades.md` |

URLs:
> - https://github.com/josefloressv/kcd_colombia_webapp_color
> - https://github.com/josefloressv/kcd_colombia_env
> - https://github.com/josefloressv/kcd_colombia_platform
> - https://github.com/josefloressv/kcd_colombia_infra

### Patrón de responsabilidades
Ver diagrama: [webapp-responsabilidades.md](./webapp-responsabilidades.md) (capas Application / Environment / Platform / Runtime). Cada repo solo contiene el “source of truth” de su dominio; ArgoCD y Terraform convergen en el cluster final. También puedes revisar la estructura de archivos de la app en [webapp-estructura.md](./webapp-estructura.md).

## 3. Flujo GitOps End‑to‑End (Resumen)
1. Dev hace commit + tag SemVer (`1.2.3`) en App Repo.
2. GitHub Actions (workflow `cicd.yaml`) construye imagen y publica en ECR.
3. El mismo pipeline actualiza el archivo de valores en `kcd_colombia_env` con el nuevo `image.tag`.
4. ArgoCD (observando `env` y `platform`) detecta el commit y sincroniza el release Helm en el cluster EKS.
5. ALB Ingress expone el servicio en `https://eks.gitops.club/` (resuelto vía Route53 + external-dns).

Ver diagrama detallado: [eks-flowchart.md](./eks-flowchart.md).

## 4. Migración: ECS → EKS
ECS (deploy blue/green + cutover) continúa conviviendo mientras se valida EKS. La misma imagen construida se reutiliza para ambos destinos durante transición (etapa opcional). El GitOps reemplaza la lógica manual de “cutover” con reconciliación declarativa.

## 5. Infraestructura (Terraform)
1. `terraform/core`: Cluster de ECS, VPC, subnets, Route53 zone, certificados ACM, ECR repos.
2. `terraform/apps` ECS portions para ECS service y tasks
3. `terraform/eks`: Cluster EKS + Node Groups + IRSA roles.

Ejecución rápida (ejemplo):
```bash
cd kcd_colombia_infra/
# EKS
./deploy-infra.sh eks prod apply

# ECS
./deploy-infra.sh core prod apply
./deploy-infra.sh apps/webapp-bg prod apply
```

## 6. Pre‑requisitos & Setup local
Herramientas:
* `git`, `gh` (opcional)
* `awscli` v2 con perfil configurado
* `terraform` >= 1.5
* `kubectl` >= 1.29
* `helm` >= 3.12
* `jq`, `yq` (utilidades)
* Dominio delegado en Route53 para `gitops.club` (o adapta el host)

AWS / Seguridad:
* Configurar **OIDC GitHub Actions → AWS**: ver doc oficial https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
* Crear rol IAM con trust policy para `token.actions.githubusercontent.com` y limitar a repos/tag pattern.

GitHub Secrets (App Repo):
| Secret | Descripción |
|--------|------------|
| `ECR_URI` | URI base del repositorio ECR (sin tag) |
| `AWS_ROLE_ARN` | ARN del rol asumido por GitHub OIDC |
| `ENV_REPO_PAT` | PAT con permisos de push sobre `kcd_colombia_env` |

## 7. Pipeline CI/CD (GitHub Actions)
Workflow `cicd.yaml` (App Repo) se dispara por tag SemVer y ejecuta jobs:
1. `build`: build & push (exporta `image_tag` = SHA corto o tag).
2. `deploy_to_ecs` y `cutover`: (fase ECS opcional legacy).
3. `deploy_to_eks`: actualiza `kcd_colombia_env` (commit de `webapp-color.values.yaml`).

Diagramas: [github-workflow-flowchart.md](./github-workflow-flowchart.md) y [github-workflow-secuence.md](./github-workflow-secuence.md).

### Tagging Strategy
* Crear tag: `git tag 1.0.0 && git push origin 1.0.0`
* Mantener versiones incrementales para reproducibilidad.

## 8. Uso de Helm & Charts
El chart vive en el App Repo (`helm/webapp-color`). Valores de runtime sensibles (imagen, ingress) se sobreescriben desde el **Env Repo**.

Validaciones:
```bash
# Lint chart
helm lint helm/webapp-color

# Render (dry-run)
helm upgrade --install webapp-color helm/webapp-color \
  --namespace webapp --create-namespace \
  -f ../kcd_colombia_env/clusters/prod/apps/webapp-color.values.yaml \
  --dry-run --debug
```

## 9. ArgoCD: Apps & Operación
Bootstrap simplificado (desde `kcd_colombia_platform`):
```bash
cd kcd_colombia_platform
./bootstrap-argocd.sh
```
Root app de Addons (desde `kcd_colombia_platform`):
```bash
kubectl apply -f kcd_colombia_platform/clusters/prod/root-app.yaml
kubectl -n argocd get applications
```
Port-forward UI:
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
Password inicial:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```
Root Apps (desde `kcd_colombia_platform`):
```bash
kubectl apply -f kcd_colombia_platform/clusters/prod/apps-root.yaml
kubectl -n argocd get applications
```
Operaciones:
```bash
kubectl -n argocd get app
argocd app history webapp-color (si usas CLI argocd)
kubectl -n argocd describe application webapp-color
```
Rollback (ejemplo CLI):
```bash
argocd app rollback webapp-color 2
```

## 10. DNS, Ingress y ALB
El **AWS Load Balancer Controller** administra ALBs basados en Ingress. `external-dns` reconcilia registros Route53. Host objetivo: `eks.gitops.club`.

Verificación:
```bash
kubectl get ingress -A
kubectl get crd | grep elbv2
kubectl get ingressclass
kubectl -n kube-system get pods -l app.kubernetes.io/name=external-dns
kubectl -n kube-system logs deploy/external-dns | tail -n 20
```

Fragmento de valores (referencia):
```yaml
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: <ACM_CERT_ARN>
  hosts:
    - host: eks.gitops.club
      paths:
        - path: /
          pathType: Prefix
```

## 11. Comandos de Operación / Troubleshooting
```bash
# Curl interno
kubectl run tmp-curl --rm -it --image=curlimages/curl --restart=Never -- curl -s webapp-color:8080/

# Port-forward para debug
kubectl port-forward svc/webapp-color 8081:8080

# Estado del rollout
kubectl rollout status deploy/webapp-color

# Eventos Ingress / Service
kubectl describe ingress webapp-color
kubectl describe svc webapp-color

# Ver manifiestos renderizados por ArgoCD
kubectl -n argocd get applications webapp-color -o yaml | yq '.status.operationState.syncResult.resources'

# Forzar resync (annotation change)
kubectl -n argocd annotate app/webapp-color gitops.force.sync=$(date +%s) --overwrite

# Confirmar image tag en pods
kubectl get pods -l app=webapp-color -o jsonpath='{..image}' | tr ' ' '\n'
```

## 12. Limpieza (Tear‑down)
Orden inverso para evitar recursos huérfanos:
```bash
# ArgoCD Apps (opcional si usas app-of-apps)
kubectl delete -f kcd_colombia_platform/clusters/prod/root-app.yaml

# ArgoCD (si lo instalaste con Helm directamente)
helm -n argocd uninstall argocd || true
kubectl delete ns argocd --wait=false

# Terraform (eks luego core)
cd kcd_colombia_infra/terraform/eks && terraform destroy -var-file=config/prod.tfvars
cd ../core && terraform destroy -var-file=config/prod.tfvars
```

## 13. Diagramas Referenciados
| Archivo | Descripción |
|---------|-------------|
| [`eks-flowchart.md`](./eks-flowchart.md) | Flujo GitOps EKS + ArgoCD + Helm |
| [`github-workflow-flowchart.md`](./github-workflow-flowchart.md) | Pipeline CI/CD simplificado |
| [`github-workflow-secuence.md`](./github-workflow-secuence.md) | Secuencia detallada de jobs GitHub Actions |
| [`webapp-responsabilidades.md`](./webapp-responsabilidades.md) | Capas y responsabilidades por repositorio |
| [`webapp-estructura.md`](./webapp-estructura.md) | Estructura de la aplicación y chart |
| [`ecs-flowchart.md`](./ecs-flowchart.md) | Flujo en ECS (estado previo) |
| [`ecs-infra-flowchart.md`](./ecs-infra-flowchart.md) | Infra ECS |
| [`ecs-sequence.md`](./ecs-sequence.md) | Secuencia de despliegue ECS |

## 14. Buenas Prácticas & Lecciones
* **Separación de concerns**: código ≠ configuración ≠ plataforma ≠ infraestructura.
* **Inmutabilidad**: solo los tags SemVer producen despliegues; evita usar `latest`.
* **Reconciliación observable**: monitorea ArgoCD Application health & sync status.
* **Seguridad**: OIDC en vez de credenciales estáticas; roles con permisos mínimos (ECR: pull/push, SSM/Secrets si aplica, etc.).
* **Rollback rápido**: mantener historia de image tags + versiones de valores.
* **Drift detection**: ArgoCD notifica diferencias; prohibe cambios manuales salvo excepciones.

## 15. Siguientes Pasos / Extensiones
* Añadir Progressive Delivery (Argo Rollouts / Flagger).
* Incorporar pruebas de carga automatizadas antes de actualizar `env`.
* Firmar imágenes (Cosign) y validar en admisión (OPA/Gatekeeper o Kyverno).
* Añadir escaneo de vulnerabilidades (Trivy) en job build.
* Multi‑environment GitOps (stg, prod) con folders por ambiente y promotion basada en PRs.

---
### Referencias
* OIDC GitHub → AWS: ver documentación enlazada arriba.
* ArgoCD: https://argo-cd.readthedocs.io
* Helm: https://helm.sh/docs/

---
Si encuentras un gap en la guía abre un issue / PR en este repo de documentación.

