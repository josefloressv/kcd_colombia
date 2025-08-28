# kcd_colombia
KCD Colombia GitOps


Resources

Adding the identity provider to AWS
https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws


# Helm Commands
```bash

# Validate helm chart
helm lint ./webapp-color

# Dry-run
helm install webapp-color ./webapp-color/ --set image.repository=562802099288.dkr.ecr.us-east-1.amazonaws.com/webapp-color-prod,image.tag=1.0.0 --dry-run --debug

# test internall app
 kubectl run curl --rm -it --image=curlimages/curl --restart=Never -- curl -s webapp-color:8080

# Exponse through Load Balancer without ingress for now
helm upgrade --install webapp-color ./webapp-color --set image.repository=562802099288.dkr.ecr.us-east-1.amazonaws.com/webapp-color-prod,image.tag=1.0.0,service.type=LoadBalancer --reuse-values
```

Validar AWS Load Balancer Controller
```bash
# Ver las CRDs instaladas
kubectl get crd | grep elbv2

# Ver el IngressClass administrado por el ALB Controller
kubectl get ingressclass

```

Configurar app con el dominio
values.eks.yaml
```yaml
image:
  repository: 562802099288.dkr.ecr.us-east-1.amazonaws.com/webapp-color-prod
  tag: 1.0.0
service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: "<ARN_DEL_CERTIFICADO_ACM>"
    # Opcional: timeouts, health-checks, grupo común si tendrás varios paths/apps:
    # alb.ingress.kubernetes.io/healthcheck-path: /
    # alb.ingress.kubernetes.io/group.name: web-public
  hosts:
    - host: eks.gitops.club
      paths:
        - path: /
          pathType: Prefix
  tls: []  # Con ALB, TLS se determina por el annotation del cert; esta sección es opcional/no requerida
```
```bash
# test the app
 k port-forward svc/webapp-color 8081:8080
 ```


```bash
# Upgrade the helm app
helm upgrade --install webapp-color ./webapp-color  -f values.eks.yaml

# listar ingress
k describe ingress

```

Validar external-dns
```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=external-dns
# Logs
kubectl -n kube-system logs deploy/external-dns -f
```

ArgoCD
```bash
# Install ArgoCD
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --version 7.4.0 \
  --set server.service.type=ClusterIP \
  --set server.insecure=true
  
# get the load balancer URL
kubectl -n argocd get svc argocd-server

# list applications
kubectl -n argocd get applications

# Port Forward to ArgoCD
k -n argocd port-forward services/argocd-server 8080:443

# get Load Balancd URL
kubectl -n argocd get svc argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'; echo || true

# get initial admin secret
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

kcd_colombia_platform
```bash
# Aplicar root app
kubectl apply -f addons/argocd/application.yaml
kubectl apply -f clusters/prod/root-app.yaml
```

Clean
```bash
# remove the root app
# remove argocd
helm -n argocd remove argocd
# o si no fue con helm
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# check resourses orphaned
k -n argocd get all
# wait until LoadBalancer is deleted and then delete the namespace
k delete ns argocd 
```
