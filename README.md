# argoCD Notes

## setup argo

```bash
curl -s https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml | sed -e 's/#- name: KUBELET_ROOT_DIR/- name: KUBELET_ROOT_DIR/g' -e 's$#  value: /var/lib/rancher/k3s/agent/kubelet$  value: /var/lib/kubelet$g' | kubectl apply -f -

kubectl apply -f https://raw.githubusercontent.com/clemenko/k8s_yaml/master/traefik_crd_deployment.yml

kubectl apply -f https://raw.githubusercontent.com/clemenko/k8s_yaml/master/stackrox_traefik_crd.yml

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

#kubectl apply -f ../k8s_yaml/argocd.yml

cat << EOF | kubectl apply -f -
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: argocd-ingressroute
  namespace: argocd
spec:
  entryPoints:
    - tcp
  routes:
    - match: HostSNI(\`argo.dockr.life\`)
      services:
        - name: argocd-server
          port: 443
  tls:
    passthrough: true
EOF


# kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2 | pbcopy

rm -rf ~/.argocd/config

argocd login argo.dockr.life:443 --username admin --insecure --password $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2 )

kubectl create ns guestbook
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace guestbook


kubectl create namespace stackrox
argocd app create stackrox-central-services --repo https://github.com/stackrox/helm-charts.git --path 3.0.52.0/central-services --dest-server https://kubernetes.default.svc --dest-namespace stackrox --values values.yaml -p imagePullSecrets.allowNone=true

```