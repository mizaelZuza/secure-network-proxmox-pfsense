# Configuração de Cluster Kubernetes em Proxmox com pfSense

Este diretório contém a documentação detalhada para a criação e configuração de um cluster Kubernetes de alta disponibilidade em um ambiente de virtualização Proxmox, utilizando pfSense como firewall e roteador.

## 📜 Conteúdo

Os guias estão divididos em dois arquivos principais:

### 1. 🚀 `setup-kubernetes.md`

Este é o **guia principal e completo**, um passo a passo que aborda toda a jornada de provisionamento da infraestrutura e instalação do cluster.

**Tópicos abordados:**
- **Arquitetura**: Definição da topologia de rede, endereçamento IP e recursos para as VMs.
- **Proxmox e pfSense**: Configuração da rede virtual (bridge), interfaces, aliases e regras de firewall.
- **Criação das VMs**: Instruções detalhadas para criar as máquinas virtuais para Load Balancers, Control Planes e Workers.
- **Configuração Base**: Instalação do Ubuntu Server 24.04, configuração de rede estática e pacotes essenciais.
- **Alta Disponibilidade (HA)**: Instalação e configuração do HAProxy e Keepalived para o balanceamento de carga da API do Kubernetes.
- **Preparação para o Kubernetes**: Desativação de swap, configuração de módulos de kernel, instalação do `containerd` e das ferramentas `kubeadm`, `kubelet` e `kubectl`.
- **Inicialização do Cluster**: Comandos para usar `kubeadm init` e `kubeadm join` para montar o cluster de forma sequencial e segura.
- **Pós-instalação**: Instalação do CNI (Flannel), testes de validação e guias opcionais para Dashboard, Storage e Monitoramento.

---

### 2. 🧠 `configuracoes-adicionais.md`

Este é um **guia de aprofundamento técnico** que complementa o guia principal, explicando o **"porquê"** por trás de configurações críticas para o funcionamento do Kubernetes. É ideal para entender a teoria e resolver problemas.

**Tópicos abordados:**
- **Módulos do Kernel**: Explicação detalhada sobre a necessidade dos módulos `overlay` (para o sistema de arquivos em camadas dos containers) e `br_netfilter` (para a comunicação de rede entre Pods e Services).
- **Parâmetros `sysctl`**: Justificativa para as configurações `net.bridge.bridge-nf-call-iptables` e `net.ipv4.ip_forward`.
- **NAT no pfSense**: Um guia focado em como configurar o NAT de saída (Outbound NAT) para permitir que as VMs do cluster acessem a internet.
- **HAProxy para o Dashboard**: Snippet de configuração para expor o Kubernetes Dashboard através do HAProxy.

## 🎯 Como Usar

1.  Comece pelo guia **`setup-kubernetes.md`** para construir a infraestrutura e o cluster passo a passo.
2.  Consulte o guia **`configuracoes-adicionais.md`** sempre que quiser entender a razão técnica por trás de uma configuração ou para auxiliar no troubleshooting de problemas de rede e de nós.
