# OpenShift AI com GPU NVIDIA no CRC

Este repositorio configura, via OpenShift GitOps/Argo CD, um laboratorio local para Red Hat OpenShift AI usando uma GPU NVIDIA RTX 3060 exposta ao cluster OpenShift Local/CRC.

## Visao geral

O Argo CD aplica este fluxo:

1. `argocd-namespaces-app.yaml` cria a Application raiz `openshift-ai-lab`.
2. `apps/` cria as Applications filhas.
3. `namespaces/` cria os namespaces usados pelos Operators e pelo projeto de estudo.
4. `operators/` instala os Operators via OLM:
   - Node Feature Discovery, no namespace `openshift-nfd`.
   - NVIDIA GPU Operator, no namespace `nvidia-gpu-operator`.
   - Red Hat OpenShift AI Operator, no namespace `redhat-ods-operator`.
5. `instances/` cria os recursos operacionais:
   - `NodeFeatureDiscovery` para descobrir a GPU no node.
   - `ClusterPolicy` para instalar driver, device plugin, toolkit e metricas NVIDIA.
   - `DSCInitialization`, `DataScienceCluster` e `AcceleratorProfile` do OpenShift AI.
6. `projects/ai-gpu-lab/` cria um Job CUDA simples para validar o uso de `nvidia.com/gpu: 1`.

## O que fica fora do Argo CD

O Argo CD nao consegue passar a placa fisica da sua maquina para a VM do CRC. Essa etapa precisa ser feita no host com libvirt/virt-manager antes do OpenShift enxergar a RTX 3060.

Tambem nao coloquei credenciais, pull secrets ou usuarios no Git. Esses itens devem ser tratados fora do repositorio ou com um fluxo seguro como Sealed Secrets, External Secrets ou SOPS.

## Pre-requisitos no host

1. Linux com suporte a virtualizacao e IOMMU habilitado na BIOS/UEFI.
2. `crc`, `oc`, QEMU/libvirt e `virt-manager` instalados.
3. Pull secret valido da Red Hat no CRC.
4. Recursos suficientes para o CRC. Para OpenShift AI, use o maximo que sua maquina permitir. Um ponto de partida para laboratorio:

```bash
crc config set cpus 18
crc config set memory 32000
crc config set disk-size 200
crc setup
crc start
```

OpenShift AI consome bastante CPU e memoria. Em CRC, a configuracao deste repositorio deixa somente dashboard e workbenches como `Managed`; KServe, pipelines, model registry, Ray, CodeFlare e outros componentes ficam como `Removed` para reduzir carga.

## Expor a RTX 3060 para a VM do CRC

1. Pare o CRC se ele estiver rodando:

```bash
crc stop
```

2. Identifique os dispositivos PCI da GPU e do audio HDMI da GPU:

```bash
lspci -nn | grep -i nvidia
```

3. Garanta que IOMMU esta ativo no host:

```bash
dmesg | grep -e DMAR -e IOMMU
```

4. Inicie o CRC:

```bash
crc start
```

5. Abra o `virt-manager`, selecione a VM do CRC, clique em detalhes da VM e adicione a GPU como `PCI Host Device`. Em placas NVIDIA, normalmente voce precisa adicionar a funcao VGA/3D e a funcao de audio HDMI da mesma placa.

6. Reinicie a VM do CRC depois de anexar os dispositivos:

```bash
crc stop
crc start
```

Essa abordagem e adequada para laboratorio. Ela depende do seu hardware, grupos IOMMU e de como o host esta usando a GPU. Se a RTX 3060 for a unica GPU do host e estiver dirigindo seu monitor, o passthrough pode exigir configuracao VFIO mais cuidadosa no host.

## Aplicar o bootstrap no Argo CD

Depois de logar no cluster:

```bash
oc login --token=<token> --server=<api-do-crc>
```

Se o Argo CD ja estiver apontando para a raiz do repositorio, basta fazer commit/push e aguardar a sincronizacao.

Se precisar aplicar manualmente a Application raiz:

```bash
oc apply -f argocd-namespaces-app.yaml
```

Verifique as Applications:

```bash
oc get applications -n openshift-gitops
```

## Validacoes

Verifique os Operators:

```bash
oc get csv -n openshift-nfd
oc get csv -n nvidia-gpu-operator
oc get csv -n redhat-ods-operator
```

Verifique o NFD:

```bash
oc get pods -n openshift-nfd
oc get node -o json | jq '.items[0].metadata.labels | with_entries(select(.key | startswith("feature.node.kubernetes.io")))'
```

Verifique a GPU no node:

```bash
oc describe node $(oc get nodes -o name | head -n1) | grep -A12 -E "Capacity|Allocatable|nvidia.com/gpu"
```

O esperado e aparecer algo como:

```text
nvidia.com/gpu: 1
```

Verifique o NVIDIA GPU Operator:

```bash
oc get pods,daemonset -n nvidia-gpu-operator
oc get clusterpolicy gpu-cluster-policy -o yaml
```

Valide o Job CUDA:

```bash
oc get job,pod -n ai-gpu-lab
oc logs job/cuda-vectoradd -n ai-gpu-lab
```

O log esperado contem uma mensagem de sucesso do exemplo `VectorAdd`.

Verifique o OpenShift AI:

```bash
oc get dscinitialization
oc get datasciencecluster default-dsc -o yaml
oc get pods -n redhat-ods-applications
oc get acceleratorprofile -n redhat-ods-applications
```

No dashboard do OpenShift AI, o projeto `ai-gpu-lab` deve aparecer como Data Science Project, e o perfil `NVIDIA GPU` deve estar disponivel ao criar um workbench.

## Por que os componentes foram escolhidos assim

- `stable-2.25` no OpenShift AI: canal numerado e conservador para laboratorio em OpenShift Local, evitando salto automatico para outra major.
- `dashboard` e `workbenches` como `Managed`: suficientes para estudar OpenShift AI e criar notebooks com GPU.
- KServe, ModelMesh, pipelines, Ray, CodeFlare, Kueue e model registry como `Removed`: esses componentes sao uteis, mas aumentam muito o consumo e alguns exigem operadores/dependencias adicionais.
- NFD antes do NVIDIA GPU Operator: o NFD rotula o node com as caracteristicas de hardware para que a stack NVIDIA consiga atuar corretamente.
- `ClusterPolicy` separado da Subscription: primeiro OLM instala o Operator, depois a CR `ClusterPolicy` declara a stack NVIDIA desejada.
- `AcceleratorProfile` com `identifier: nvidia.com/gpu`: esse e o recurso Kubernetes que workloads usam para requisitar GPU.

## Fontes principais

- Red Hat OpenShift AI Self-Managed 2.25 - instalacao, canais, `DataScienceCluster` e aceleradores: https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.25/html-single/installing_and_uninstalling_openshift_ai_self-managed/index
- Red Hat OpenShift AI 2.25 - accelerator profiles: https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.25/html/working_with_accelerators/working-with-accelerator-profiles_accelerators
- NVIDIA GPU Operator on OpenShift: https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/install-gpu-ocp.html
- Red Hat hardware accelerators / GPU Operator configuration: https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/hardware_accelerators/index
- Red Hat Developer - GPU acceleration in OpenShift Local: https://developers.redhat.com/articles/2025/11/27/how-enable-nvidia-gpu-acceleration-openshift-local
