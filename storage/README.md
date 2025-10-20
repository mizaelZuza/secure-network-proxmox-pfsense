#  Storage com TrueNAS SCALE para Proxmox e Kubernetes

Este diretório contém a documentação para configurar um servidor de armazenamento **TrueNAS SCALE** virtualizado no **Proxmox VE**, com o objetivo de fornecer storage persistente via **NFS** para um cluster **Kubernetes**.

## 📚 Guias

O processo foi dividido em duas partes principais:

1.  **[Instalação e Configuração do TrueNAS](./configuracoes-adicionais.md)**:
    *   **Objetivo**: Detalha a criação da VM do TrueNAS no Proxmox, a configuração dos discos (1 para o SO e 3 para um pool RAIDZ1) e a instalação inicial do sistema.
    *   **Quando usar**: Siga este guia primeiro para preparar o ambiente de virtualização e o storage base.

2.  **[Configuração do NFS para Kubernetes](./setup-truenas.md)**:
    *   **Objetivo**: Descreve como configurar um *Dataset* no TrueNAS, ativar o serviço NFS e criar um compartilhamento para ser consumido pelo Kubernetes. Inclui também as regras de firewall necessárias no pfSense e os manifestos (`.yaml`) para criar o `StorageClass`, `PersistentVolume` (PV) e `PersistentVolumeClaim` (PVC).
    *   **Quando usar**: Após ter o TrueNAS instalado e o pool de armazenamento criado, siga este guia para integrar o storage ao seu cluster.

## 🚀 Fluxo de Trabalho

O fluxo recomendado é:

1.  Comece pelo guia `configuracoes-adicionais.md` para ter uma instância do TrueNAS funcional.
2.  Continue com o guia `setup-truenas.md` para expor o armazenamento via NFS e conectá-lo ao Kubernetes.
