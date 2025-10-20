# Laboratório de Infraestrutura Completa com Proxmox, pfSense, TrueNAS e Kubernetes

Este repositório serve como um guia detalhado para a construção de uma infraestrutura de laboratório robusta e segura, simulando um ambiente de produção moderno. O projeto utiliza tecnologias open-source de ponta para virtualização, segurança de rede, armazenamento e orquestração de contêineres.

## 🎯 Objetivo do Projeto

O objetivo principal é documentar o processo de ponta a ponta para configurar um ambiente totalmente funcional, segmentado e seguro, utilizando:
- **Proxmox VE** como a plataforma de virtualização (hypervisor).
- **pfSense** como firewall, roteador e sistema de prevenção de intrusão (IDS/IPS).
- **TrueNAS SCALE** como servidor de armazenamento de rede (NAS), fornecendo storage persistente.
- **Kubernetes (K8s)** para orquestrar aplicações em contêineres de forma resiliente.

## 🏛️ Arquitetura da Infraestrutura

A arquitetura é projetada para ser modular e segmentada, onde cada componente tem uma função específica e se comunica de forma segura através de redes virtuais isoladas.

| Camada | Componente | Rede / Bridge | Sub-rede | Função / Conexão |
| :--- | :--- | :--- | :--- | :--- |
| **Externa** | Internet | - | - | Conexão com o provedor de internet. |
| **Host Físico** | Proxmox VE | `vmbr0` (WAN) | - | Hypervisor que hospeda todas as VMs e faz a ponte com a rede externa. |
| **Firewall** | VM: **pfSense** | `vmbr0` (WAN) | DHCP | Recebe o link da internet. |
| | | `vmbr1` (LAN Servidores) | `192.168.1.1/24` | Gateway para a rede de serviços (Storage). |
| | | `vmbr2` (LAN Clientes) | `192.168.2.1/24` | Gateway para a rede de usuários finais. |
| | | `vmbr3` (LAN Kubernetes) | `192.168.3.1/24` | Gateway para o cluster de contêineres. |
| **Serviços** | VM: **TrueNAS SCALE** | `vmbr1` | `192.168.1.0/24` | Fornece armazenamento via NFS para o Kubernetes. |
| | VM: **Cluster Kubernetes** | `vmbr3` | `192.168.3.0/24` | Executa as aplicações em contêineres. |

- **Proxmox VE**: É a camada base que hospeda todas as Máquinas Virtuais (VMs).
- **pfSense**: Atua como o "portão" da rede. Ele gerencia todo o tráfego entre a internet e as redes internas (Servidores, Clientes, Kubernetes), aplicando regras de firewall e monitorando ameaças com o Suricata.
- **TrueNAS SCALE**: Fornece armazenamento centralizado e resiliente (via RAIDZ) para o cluster Kubernetes através do protocolo NFS.
- **Kubernetes**: O cluster, isolado em sua própria rede, utiliza o armazenamento do TrueNAS para dados persistentes de suas aplicações.

## 🚀 Tecnologias Aplicadas

| Categoria | Tecnologia | Função |
| :--- | :--- | :--- |
| **Virtualização** | **Proxmox VE** | Hypervisor para criação e gerenciamento de VMs. |
| **Firewall & Rede** | **pfSense** | Roteamento, Firewall, DHCP e segmentação de rede. |
| **Segurança** | **Suricata** | Sistema de Detecção e Prevenção de Intrusão (IDS/IPS). |
| **Armazenamento** | **TrueNAS SCALE** | Storage de rede (NAS) com ZFS para resiliência. |
| **Protocolo de Storage**| **NFS** | Compartilhamento de arquivos em rede para o Kubernetes. |
| **Orquestração** | **Kubernetes (K8s)** | Gerenciamento de contêineres em alta disponibilidade. |
| **Sistema Operacional**| **Ubuntu Server** | SO base para os nós do Kubernetes. |

## 📂 Estrutura do Repositório

Este projeto é organizado em diretórios, cada um focando em uma parte específica da infraestrutura:

- **/proxmox**: Contém guias para a configuração inicial do Proxmox, incluindo a criação das redes virtuais (bridges) e o provisionamento de todas as VMs necessárias.
- **/firewall**: Documentação detalhada sobre a instalação e configuração do pfSense, criação de regras de firewall, aliases e a implementação do IDS/IPS com Suricata.
- **/storage**: Guias para instalar o TrueNAS SCALE em uma VM, configurar pools de armazenamento (RAIDZ) e expor o storage via NFS para o Kubernetes.
- **/k8s**: Detalhes sobre a montagem do cluster Kubernetes, incluindo a configuração dos nós (Control Plane, Workers) e a integração com o storage persistente.

## 🏁 Como Começar

Para construir a infraestrutura do zero, siga a ordem recomendada dos guias:

1.  **1º - Proxmox**: Comece configurando o hypervisor e as redes base.
2.  **2º - Firewall**: Implemente o pfSense para estabelecer as fundações de rede e segurança.
3.  **3º - Storage**: Configure o TrueNAS para preparar o armazenamento persistente.
4.  **4º - Kubernetes**: Com a infraestrutura de base pronta, construa o cluster Kubernetes.

## ✨ Principais Funcionalidades Implementadas

- **Segmentação de Rede**: Isolação completa entre as redes de servidores, clientes e do cluster Kubernetes, aumentando a segurança.
- **Defesa em Profundidade**: Múltiplas camadas de segurança com firewall (pfSense) e detecção de intrusão (Suricata).
- **Armazenamento Resiliente**: Uso de ZFS e RAIDZ no TrueNAS para proteger os dados contra falha de disco.
- **Storage Persistente para Contêineres**: Permite que aplicações no Kubernetes salvem dados que persistem além do ciclo de vida de um pod.
- **Infraestrutura como Código (Documentação)**: O repositório serve como uma documentação detalhada que permite recriar o ambiente de forma consistente.

> **Aviso**: Este projeto foi criado para fins de estudo e aprendizado. Para ambientes de produção, recomenda-se aprofundar as configurações de segurança, realizar testes de carga e considerar hardware dedicado.
