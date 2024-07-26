###############################################################################################
### HUB #######################################################################################
###############################################################################################

# kubectl create namespace gke-gateway

# touch gke-gateway-external.yaml
---
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: gke-gateway-external
  namespace: gke-gateway
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All

# touch asm-ingressgateway-external-httproute.yaml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: argo-cd
  namespace: gke-gateway
spec:
  parentRefs:
    - kind: Gateway
      name: gke-gateway-external
      namespace: gke-gateway
  rules:
    - backendRefs:
        - kind: Service
          name: argo-cd-argocd-server
          port: 80

# kubectl apply -f gke-gateway-external.yaml

# kubectl apply -f asm-ingressgateway-external-httproute.yaml

###############################################################################################
### INSTALL ARGOCD ############################################################################
###############################################################################################

# helm repo add argo https://argoproj.github.io/argo-helm

# helm install argo-cd argo/argo-cd

# kubectl -n default get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# -> {{{{{{ aC56TT7ocT9NdVlM

# kubectl patch svc argo-cd-argocd-server -p '{"spec": {"type": "LoadBalancer"}}'

# kubectl get services (for the ip external)
# -> {{{{{{ https://104.155.58.127

###############################################################################################
###### SPOKE ##################################################################################
###############################################################################################

# touch secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: argocd
  name: argocd-token
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd
secrets:
  - name: argocd-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: argocd
    namespace: default

# kubectl apply -f secret.yaml

# kubectl get secrets argocd-token -oyaml

# -> {{{{{{ echo "TOKEN" | base64 -d

#############################################################################################
###### HUB ##################################################################################
#############################################################################################
---
# -> {{{{{{ Get control plane ip in the state file (search by Control plane address range)

# touch secret_sploke.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: sbox-gkesspk-cluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  config: >
    {
      "bearerToken": "TOKEN",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "CERTIFICATE"
      }
    }
  name: sbox-gkesspk
  server: https://192.168.3.2  https://192.168.2.2
type: Opaque

# kubectl apply -f secret_spoke.yaml

# --kubectl create namespace infra

# touch AppProject.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra-dev-stage
spec:
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  destinations:
    - namespace: infra
      server: 192.168.2.0/28
  sourceRepos:
    - https://github.com/rferreim/AppProjectTest.git
    - https://helm.releases.hashicorp.com

# touch applicationSet.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-git
spec:
  goTemplate: true
  goTemplateOptions:
    - missingkey=error
  generators:
    - matrix:
        generators:
          - clusters:
              selector:
                matchLabels:
                  argocd.argoproj.io/secret-type: cluster

# kubectl apply -f AppProject.yaml

# kubectl apply -f applicationSet.yaml
