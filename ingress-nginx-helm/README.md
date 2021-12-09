
## I don't care, Chris, just get to the argo part!!!

[Here you go](#argocd)

## Description

This is an example using kustomize to add registry-access secret. The
values.yaml provides for us to tell our deployment/ds what to use as a
imagePullSecret, it doesn't provide a template for one or a spot in the
values.yaml to add a registry pull secret.

Having said so, this is just an example, I do not recommend adding your
registry pull secret to your git repo. In production, we store all secrets in
vault and allow them to be pulled in CI/CD and populated. For secrets needed by
a container, we use vault-injector and a series of annotations to pull those
secrets. This example exists to demonstrate the following:

- Items added via the post-renderer for helm are still managed by helm and
  tracked for deletion on uninstall. That also means they will be tracked by
  ArgoCD for changes in the Web UI.
- The post render method is simple to use


## Testing locally

Although this is set up to be immedately used with ArgoCD, you may want to see
for your self locally. This is simple enough, we just have to do some of the
things that argo handles for us first:

1. Pull all dependency charts. Since argo deployment jobs are ephimeral (Did I
spell that right?), you don't maintain repositories on argo and instead use a
Chart.yaml to tell it what to pull and use, unlike using helm locally. They may
have added maintaining repos, but I started playing with ArgoCD ~2 years ago,
so I'm used to just throwing a Chart.yaml together for each app.

```bash
helm dependency build
```

2. If you just want to see and let Argo do the show, you can render the
template and see what you get! A quick note about this, it won't show you the
final annotations on the registry-access secret. It's just a quirk of
templating. Helm will add those on install, which Argo will handle for us!

```bash
helm template ingress-nginx . --post-renderer ./kustomize --values values.yaml | less
```

3a. If you want to deploy with helm to get the feel for what is happening, run
this:

```bash
helm upgrade --install ingress-nginx . --post-renderer ./kustomize --values values.yaml
### Lets checkout that kustomize secret and see the annotations
kubectl get secret registry-access -o yaml
```

```yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJkb2NrZXIuZXhhbXBsZS5jb20iOnsidXNlcm5hbWUiOiJsb2NhbC1kb2NrZXItdXNlciIsInBhc3N3b3JkIjoiU3VwZXJTZWNyZXRQYXNzd29yZCJ9fX0K
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: ingress-nginx
    meta.helm.sh/release-namespace: ingress-nginx
  creationTimestamp: "2021-12-09T02:11:34Z"
  labels:
    app.kubernetes.io/managed-by: Helm
    manager: Go-http-client
    operation: Update
  name: registry-access
  namespace: ingress-nginx
type: kubernetes.io/dockerconfigjson
```

I just redacted the managed fields, and the link stuffs to keep this simple.
Notice that helm added it's management annotations, and if you deployed with
Argo, it'll be checking for changes and tell you if someone edited the
resource!!!


## ArgoCD
### Lets just do ArgoCD Deployment!!!

1. Install Plugin

Here's our plugin for ArgoCD: (Install in ArgoCD namespace argocd-cm ConfigMap)
```yaml
  configManagementPlugins: |
    - name: kustomized-helm
      init:
        command: ["/bin/sh", "-c"]
        args: ["helm2 init --client-only && helm2 dependency build"]
      generate:
        command: [sh, -c]
        args: ["helm2 template . > all.yaml && kustomize build"]
    - name: kustomized-helm3
      init:
        command: ["/bin/sh", "-c"]
        args: ["helm dependency build || true"]
      generate:
        command: [sh, -c]
        args:
        - |
          helm template $ARGOCD_APP_NAME . --namespace $ARGOCD_APP_NAMESPACE > all.yaml && sed -e "s/custom-prefix/$ARGOCD_APP_NAME/g" -i kustomization.yaml && kustomize build
```

There's a plugin for helm2 and one for helm3, and I just checked. Argo will
track the resource from kustomize for changes, but unlike an manual helm
deploy, it doesn't add the helm allotations. O well, at least it's being
tracked. 

Make sure you add the configManagementPlugins to argocd-cm, since I couldn't
find a place in the WebUI to add it, then create the app using the yaml I
provided and finally... just relax and check it out.

2. Install App


Here's our app
```yaml
project: default
source:
  repoURL: 'https://github.com/darkdragn/helm-kustomize-exaples.git'
  path: ingress-nginx-helm
  targetRevision: HEAD
  plugin:
    name: kustomized-helm3
destination:
  server: 'https://kubernetes.default.svc'
  namespace: ingress-nginx
```

3. Check it out in ArgoCD
4. Profit???

## Todo

- Update kustomization to use customPrefix
- add patch for deploy/ds to update the name of the registry access via
  kustomize

This update will be a good demonstration of kustomize's patch capability. The
main goal will be demonstrating how you can remove sensitize data from
values.yaml, potentially into app.yaml parameters. 

I encourage you guys to play with ArgoCD's plugins for fun! Never be afraid to
make something better!
