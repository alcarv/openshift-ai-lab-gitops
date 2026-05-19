# openshift-ai-lab-gitops

GitOps lab para instalar Red Hat OpenShift AI em OpenShift Local/CRC com suporte a GPU NVIDIA.

O bootstrap principal fica em `argocd-namespaces-app.yaml`. Depois que essa Application estiver aplicada no Argo CD, ela sincroniza os apps em `apps/` usando o padrao app-of-apps.

Documentacao de estudo: [docs/openshift-ai-gpu-crc.md](docs/openshift-ai-gpu-crc.md)
