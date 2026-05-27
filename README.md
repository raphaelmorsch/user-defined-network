# User Defined Network para migração de VMs

Este repositório contém manifests Kubernetes/OpenShift para criar uma rede secundária baseada em OVN-Kubernetes, usando `ClusterUserDefinedNetwork`, `localnet` e VLAN 371. A intenção é disponibilizar uma rede de transferência no namespace `vm-migration`, normalmente usada por workloads de migração de VMs que precisam de uma interface adicional fora da rede padrão do cluster.

## Visão geral

O fluxo configurado aqui faz quatro coisas principais:

1. Cria ou documenta o namespace `vm-migration`.
2. Marca esse namespace com o label `migration-network: vlan371`.
3. Configura nos nós worker o mapeamento OVN entre uma rede física chamada `physnet371` e a bridge `br-ex`.
4. Cria uma `ClusterUserDefinedNetwork` chamada `vlan371-transfer`, que seleciona namespaces com o label acima e entrega uma rede secundária `localnet` na VLAN 371.

Quando a `ClusterUserDefinedNetwork` é criada, o controlador do OVN-Kubernetes cria automaticamente o `NetworkAttachmentDefinition` correspondente dentro do namespace selecionado.

## Arquivos

### `ns.yaml`

Define o namespace `vm-migration`.

Pontos importantes:

- `metadata.name: vm-migration`: nome do namespace onde os workloads de migração serão executados.
- `migration-network: vlan371`: label usado pela `ClusterUserDefinedNetwork` para selecionar esse namespace.
- Labels e annotations de segurança do OpenShift: mantêm o namespace alinhado ao perfil `restricted` de Pod Security/SCC.

Esse label é essencial. Sem ele, a `ClusterUserDefinedNetwork` não associa a rede ao namespace.

### `nncp.yaml`

Define uma `NodeNetworkConfigurationPolicy` do NMState.

Ela configura o mapeamento:

```yaml
localnet: physnet371
bridge: br-ex
```

Esse mapeamento informa ao OVN que a rede física `physnet371` sai pela bridge `br-ex` nos nós worker.

Pontos importantes:

- `kind: NodeNetworkConfigurationPolicy`: recurso gerenciado pelo Kubernetes NMState Operator.
- `nodeSelector.node-role.kubernetes.io/worker: ''`: aplica a configuração aos nós worker.
- `ovn.bridge-mappings`: registra o vínculo entre o nome lógico da rede física e a bridge local.

A `physicalNetworkName` usada no `cudn.yaml` precisa bater com o `localnet` definido aqui: `physnet371`.

### `cudn.yaml`

Define a `ClusterUserDefinedNetwork` principal.

Ela cria uma rede secundária OVN-Kubernetes com topologia `Localnet`, usando VLAN 371.

Configuração principal:

- `metadata.name: vlan371-transfer`: nome da rede definida pelo usuário.
- `namespaceSelector.matchLabels.migration-network: vlan371`: aplica a rede aos namespaces que tenham esse label.
- `topology: Localnet`: conecta a rede OVN a uma rede física externa ao cluster.
- `role: Secondary`: a rede é uma interface adicional, não a rede padrão dos pods.
- `physicalNetworkName: physnet371`: nome lógico da rede física, mapeado em `nncp.yaml`.
- `vlan.access.id: 371`: VLAN usada no tráfego dessa rede.
- `mtu: 1500`: MTU configurada para a rede.
- `ipam.mode: Enabled`: o OVN-Kubernetes gerencia endereçamento IP para essa rede.
- `subnets: 161.68.120.72/29`: faixa disponível para a rede.
- `excludeSubnets`: endereços removidos do pool de alocação.

Com a subnet `161.68.120.72/29`, o bloco cobre os endereços `161.68.120.72` a `161.68.120.79`. O manifest exclui:

- `161.68.120.72/32`
- `161.68.120.73/32`
- `161.68.120.75/32`
- `161.68.120.76/32`
- `161.68.120.77/32`
- `161.68.120.78/32`
- `161.68.120.79/32`

Na prática, sobra para alocação automática o endereço `161.68.120.74`, considerando somente o que está declarado neste manifest.

### `nad.yaml`

Mostra o `NetworkAttachmentDefinition` gerado para a rede `vlan371-transfer` no namespace `vm-migration`.

Este arquivo serve como referência do estado criado pelo controlador. Ele não deve ser aplicado nem editado manualmente, porque:

- O `NetworkAttachmentDefinition` é criado automaticamente a partir da `ClusterUserDefinedNetwork`.
- O conteúdo é gerenciado pelo controlador do OVN-Kubernetes.
- O recurso tem `ownerReferences` apontando para a `ClusterUserDefinedNetwork`.
- O recurso tem finalizer de proteção `k8s.ovn.org/user-defined-network-protection`.

O trecho `spec.config` confirma os valores derivados do `cudn.yaml`, incluindo:

- `name: cluster_udn_vlan371-transfer`
- `netAttachDefName: vm-migration/vlan371-transfer`
- `physicalNetworkName: physnet371`
- `role: secondary`
- `topology: localnet`
- `vlanID: 371`
- `subnets: 161.68.120.72/29`
- `excludeSubnets` com os endereços excluídos

## Ordem recomendada de aplicação

Use a seguinte ordem:

```bash
oc apply -f ns.yaml
oc apply -f nncp.yaml
oc apply -f cudn.yaml
```

Não aplique `nad.yaml` manualmente.

Depois que a `ClusterUserDefinedNetwork` estiver pronta, o controlador deve criar o `NetworkAttachmentDefinition` no namespace `vm-migration`.

## Validação

Verifique se o namespace tem o label esperado:

```bash
oc get namespace vm-migration --show-labels
```

Verifique a política NMState:

```bash
oc get nncp nncp-physnet371
oc get nnce
```

Verifique a `ClusterUserDefinedNetwork`:

```bash
oc get clusteruserdefinednetwork vlan371-transfer
oc describe clusteruserdefinednetwork vlan371-transfer
```

Verifique se o `NetworkAttachmentDefinition` foi criado automaticamente:

```bash
oc get network-attachment-definition -n vm-migration
oc get network-attachment-definition vlan371-transfer -n vm-migration -o yaml
```

## Como usar a rede em um workload

Workloads no namespace `vm-migration` podem solicitar essa rede secundária via annotation Multus, usando o nome do `NetworkAttachmentDefinition`:

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: vlan371-transfer
```

Para VMs com KubeVirt/OpenShift Virtualization, essa rede normalmente é referenciada na especificação da VM como uma interface adicional ligada ao NetworkAttachmentDefinition `vlan371-transfer`.

## Dependências esperadas

Este conjunto de manifests assume que o cluster possui:

- OpenShift com OVN-Kubernetes.
- Suporte a `ClusterUserDefinedNetwork` no grupo `k8s.ovn.org/v1`.
- Kubernetes NMState Operator instalado para processar `NodeNetworkConfigurationPolicy`.
- Multus CNI disponível para anexar redes secundárias.
- Bridge `br-ex` existente nos nós worker.
- Conectividade física/trunk/access compatível com a VLAN 371 no caminho de rede dos nós.

## Relação entre os recursos

```text
ns.yaml
  cria namespace vm-migration
  adiciona label migration-network=vlan371

nncp.yaml
  configura physnet371 -> br-ex nos workers

cudn.yaml
  seleciona namespaces com migration-network=vlan371
  cria rede localnet physnet371 com VLAN 371
  controlador gera o NAD no namespace selecionado

nad.yaml
  representa o NAD gerado automaticamente
  deve ser tratado como referência/consulta, não como fonte editável
```

## Observações operacionais

- Se o label do namespace mudar, a `ClusterUserDefinedNetwork` pode deixar de selecionar o namespace.
- Se o `physicalNetworkName` do `cudn.yaml` não existir no bridge mapping do `nncp.yaml`, a rede localnet não terá o mapeamento físico esperado.
- Se a VLAN 371 não estiver disponível corretamente na rede física dos nós, os pods ou VMs podem receber a interface secundária, mas não conseguirão comunicar fora do cluster.
- Como quase todos os endereços do `/29` estão em `excludeSubnets`, há pouquíssimos IPs disponíveis para IPAM.
