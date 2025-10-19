# Visão Geral da Infraestrutura no Proxmox

Este projeto detalha a configuração do Proxmox VE como a base de hipervisor para uma infraestrutura de laboratório complexa, incluindo:

-   **pfSense** para firewall e roteamento.
-   **TrueNAS SCALE** para armazenamento NFS.
-   **Cluster Kubernetes** de alta disponibilidade.

O objetivo é utilizar o Proxmox para criar e gerenciar as redes virtuais e as máquinas virtuais (VMs) que hospedam esses serviços, simulando um ambiente de produção modular e segmentado.

##  Architektur no Proxmox

A configuração no Proxmox se concentra em dois pontos principais:

1.  **Redes Virtuais (Bridges)**: Criação de bridges de rede (`vmbr`) para segmentar o tráfego. Cada bridge representa uma VLAN ou uma zona de segurança (WAN, LAN, DMZ, etc.), isolando os diferentes componentes da infraestrutura.

2.  **Máquinas Virtuais (VMs)**: Provisionamento de todas as VMs com recursos de CPU, memória e disco específicos para suas funções. Isso inclui as VMs para o pfSense, TrueNAS, os Load Balancers, Control Planes e Workers do Kubernetes.

Para um guia detalhado de como configurar o ambiente Proxmox, consulte o arquivo `proxmox/setup-proxmox.md`.
