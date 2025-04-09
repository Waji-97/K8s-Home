# Flux Setup
I decided to choose FluxCD over ArgoCD for my Kubernetes cluster as I wanted to be versatile with CD tools. (I already use ArgoCD in my workplace).

## Installing the flux CLI
Flux has a one tool installation package as below
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
[INFO]  Downloading metadata https://api.github.com/repos/fluxcd/flux2/releases/latest
[INFO]  Using 2.5.1 as release
[INFO]  Downloading hash https://github.com/fluxcd/flux2/releases/download/v2.5.1/flux_2.5.1_checksums.txt
[INFO]  Downloading binary https://github.com/fluxcd/flux2/releases/download/v2.5.1/flux_2.5.1_linux_amd64.tar.gz
[INFO]  Verifying binary download
[INFO]  Installing flux to /usr/local/bin/flux
```

## Bootstraping Flux
Before
We will check for prerequisites first
```bash
flux check --pre
► checking prerequisites
✔ Kubernetes 1.31.4 >=1.30.0-0
✔ prerequisites checks passed
```

The bootstrap command
```bash
flux bootstrap github --components-extra=image-reflector-controller,image-automation-controller --cluster-domain=homelab.local --owner=Waji-97 --repository=K8s-Home --branch=main --personal --token-auth --path=./apps
```

Running the command
```bash
flux bootstrap github --components-extra=image-reflector-controller,image-automation-controller --cluster-domain=homelab.local --owner=Waji-97 --repository=K8s-Home --branch=main --personal --token-auth --path=./apps
Please enter your GitHub personal access token (PAT): 
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/Waji-97/K8s-Home.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "main" ("a87856b17eba2362261212e85a39841e678ffb30")
► pushing component manifests to "https://github.com/Waji-97/K8s-Home.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("2f1a119b46a9e6305b23e25ffa2e192666c77298")
► pushing sync manifests to "https://github.com/Waji-97/K8s-Home.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ image-automation-controller: deployment ready
✔ image-reflector-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

We can git pull using the rebase option to prevent any merge conflicts from our local git
```bash
git pull --rebase
remote: Enumerating objects: 12, done.
remote: Counting objects: 100% (12/12), done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 11 (delta 0), reused 11 (delta 0), pack-reused 0 (from 0)
Unpacking objects: 100% (11/11), 66.15 KiB | 251.00 KiB/s, done.
From github.com:Waji-97/K8s-Home
   7904c25..2f1a119  main       -> origin/main
Updating 7904c25..2f1a119
Fast-forward
 apps/flux-system/gotk-components.yaml | 14446 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 apps/flux-system/gotk-sync.yaml       |    27 +
 apps/flux-system/kustomization.yaml   |     5 +
 3 files changed, 14478 insertions(+)
 create mode 100644 apps/flux-system/gotk-components.yaml
 create mode 100644 apps/flux-system/gotk-sync.yaml
 create mode 100644 apps/flux-system/kustomization.yaml
```

## Setting up SOPS
Before setting up SOPS, we need our age key. That can be generated using the `age-keygen` command. 
> To configure SOPS, `age` package & `sops` package are required. Install them from their official github release page.

Generating an age key is quite simple
```bash
age-keygen -o age.agekey
Public key: age100v6hmwlerqakxl32jl2acv5wn69tl9qzn577sv0z7qgy35nmsjqsg8zpl
```

Then we need to create a secret using the above file.
```bash
cat age.agekey | kubectl create secret generic sops-age -n flux-system --from-file=age.agekey=/dev/stdin
secret/sops-age created
```

After that, we need the decryption feature enabled under flux's Kustomization in `gotk-sync.yaml`
```bash
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./apps
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:                        ## ==> add these lines
    provider: sops
    secretRef:
      name: sops-age
```


To use encryption for the current user, we need the above age-key to be present under `.config` directory.
```bash
mkdir -p /home/waji/.config/sops/age
mv age.agekey /home/waji/.config/sops/age/keys.txt
```

Then create a `.sops.yaml` file in main that includes encryption regex. 

Finally, we can push these changes up to remote.
