# Guia de Setup do Proxmox para Infraestrutura de Laboratório

Este guia detalha os passos necessários para configurar o Proxmox VE como a base para hospedar as VMs do pfSense, Kubernetes e TrueNAS.

## 1. Configuração das Redes Virtuais (Bridges)

Para garantir o isolamento e a organização do tráfego de rede, criaremos bridges Linux distintas para cada segmento de rede. Conecte-se ao seu host Proxmox via SSH e edite o arquivo de configuração de rede.

```bash
ssh root@<IP_DO_SEU_PROXMOX>
nano /etc/network/interfaces
```

Adicione o seguinte conteúdo ao final do arquivo. Este exemplo assume que `vmbr0` já existe e está conectada à sua rede física (WAN para o pfSense).

```ini
# /etc/network/interfaces

# ... (configuração existente de vmbr0)

# Bridge para a rede de Servidores (ex: TrueNAS, etc.)
auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment "LAN_SERVIDORES - Rede para Servidores e Storage"

# Bridge para a rede de Clientes/Desktops
auto vmbr2
iface vmbr2 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment "LAN_CLIENTES - Rede para máquinas de usuários finais"

# Bridge para o Cluster Kubernetes
auto vmbr3
iface vmbr3 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment "LAN_K8S - Rede isolada para o cluster Kubernetes"
```

### Aplicar as Configurações de Rede

Após salvar o arquivo, reinicie o serviço de rede do Proxmox para aplicar as alterações. Isso pode ser feito pela interface web ou pelo comando:

```bash
# Cuidado: Este comando pode derrubar sua conexão SSH se houver erros no arquivo.
systemctl restart networking
```

Verifique se as novas bridges foram criadas com sucesso:

```bash
ip a | grep 'vmbr'
```

Você deverá ver as interfaces `vmbr1`, `vmbr2` e `vmbr3` listadas.

## 2. Upload das Imagens ISO

Antes de criar as VMs, faça o upload das imagens ISO necessárias para o storage local do Proxmox.

1.  Acesse a interface web do Proxmox.
2.  Navegue até **Datacenter -> (seu nó) -> local -> ISO Images**.
3.  Clique em **Upload** e selecione os seguintes arquivos ISO do seu computador:
    *   **pfSense**: Imagem de instalação do pfSense (ex: `pfSense-CE-2.7.2-RELEASE-amd64.iso`).
    *   **Ubuntu Server**: Imagem para os nós do Kubernetes (ex: `ubuntu-24.04-live-server-amd64.iso`).
    *   **TrueNAS SCALE**: Imagem de instalação do TrueNAS.

Alternativamente, você pode baixar as ISOs diretamente no shell do Proxmox:

```bash
cd /var/lib/vz/template/iso
wget <URL_DA_IMAGEM_ISO>
```

## 3. Provisionamento das Máquinas Virtuais (VMs)

Com as redes e as ISOs prontas, o próximo passo é criar as máquinas virtuais. A criação é feita através do assistente **"Create VM"** na interface web do Proxmox.

### Recomendações Gerais para a Criação das VMs:

-   **Guest OS**: Selecione o tipo de sistema operacional correspondente (Linux ou Other).
-   **System**: Para sistemas mais modernos como TrueNAS e Ubuntu 24.04, prefira `q35` como `Machine` e `OVMF (UEFI)` como `BIOS`.
-   **SCSI Controller**: Utilize `VirtIO SCSI single` para melhor performance.
-   **QEMU Guest Agent**: **Sempre habilite** esta opção na aba `System`. Lembre-se de instalar o agente dentro do sistema operacional da VM posteriormente (`sudo apt install qemu-guest-agent`).
-   **Disks**: Use `VirtIO Block` como `Bus/Device` para discos. Habilite `Discard` se estiver usando um SSD.
-   **CPU**: Para melhor desempenho, use o tipo `host`.
-   **Memory**: Não habilite `Ballooning` para VMs críticas como pfSense, TrueNAS e nós de controle do Kubernetes, garantindo que a memória alocada seja reservada.
-   **Network**: Utilize o modelo `VirtIO (paravirtualized)` para a melhor performance de rede.

### Conectando as VMs às Bridges Corretas:

Ao criar cada VM, na aba **Network**, certifique-se de selecionar a **Bridge** correta para a sua função:

-   **VM do pfSense**:
    -   `net0`: Conectada à `vmbr0` (WAN).
    -   `net1`: Conectada à `vmbr1` (LAN_SERVIDORES).
    -   `net2`: Conectada à `vmbr2` (LAN_CLIENTES).
    -   `net3`: Conectada à `vmbr3` (LAN_K8S).
-   **VM do TrueNAS**:
    -   `net0`: Conectada à `vmbr1` (LAN_SERVIDORES).
-   **VMs do Cluster Kubernetes (Load Balancers, Control Planes, Workers)**:
    -   `net0`: Todas conectadas à `vmbr3` (LAN_K8S).

## 4. Próximos Passos

Com a infraestrutura base no Proxmox configurada, você pode prosseguir com a instalação e configuração de cada serviço, seguindo os guias específicos em suas respectivas pastas:

-   `/firewall` para o pfSense.
-   `/storage` para o TrueNAS.
-   `/k8s` para o cluster Kubernetes.
