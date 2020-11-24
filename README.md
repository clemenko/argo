# argoCD Notes

## setup argo

```bash
curl -s https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml | sed -e 's/#- name: KUBELET_ROOT_DIR/- name: KUBELET_ROOT_DIR/g' -e 's$#  value: /var/lib/rancher/k3s/agent/kubelet$  value: /var/lib/kubelet$g' | kubectl apply -f -

kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

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

# stackrox argo stuff

echo -n " - help.stackrox.com username : "; read username; echo
echo -n " - help.stackrox.com password : "; read -s password; echo

kubectl create namespace stackrox

argocd app create stackrox-central-services --repo https://github.com/clemenko/argo.git --path 3.0.52.0/central-services --dest-server https://kubernetes.default.svc --dest-namespace stackrox --values values.yaml -p imagePullSecrets.username=$username -p imagePullSecrets.password=$password -p central.adminPassword.value=Pa22word
#--helm-set-file licenseKey=stackrox.lic -p imagePullSecrets.allowNone=true

# adding sensor
argocd app create stackrox-sensor --repo https://github.com/clemenko/argo.git --path 3.0.52.0/sensor --dest-server https://kubernetes.default.svc --dest-namespace stackrox --values values.yaml -p imagePullSecrets.username=$username -p imagePullSecrets.password=$password -p central.adminPassword.value=Pa22word

# add traefik ingressroutetcp
kubectl apply -f https://raw.githubusercontent.com/clemenko/k8s_yaml/master/stackrox_traefik_crd.yml

```