# Guia de Setup do Proxmox para Infraestrutura de Laboratório

Este guia detalha os passos necessários para configurar o Proxmox VE como a base para hospedar as VMs do pfSense, Kubernetes e TrueNAS.

## Índice

- [1. Configuração das Redes Virtuais (Bridges)](#1-configuração-das-redes-virtuais-bridges)
- [2. Upload das Imagens ISO](#2-upload-das-imagens-iso)
- [3. Provisionamento das Máquinas Virtuais](#3-provisionamento-das-máquinas-virtuais-vms)
- [4. Checklist Pós-Criação](#4-checklist-pós-criação-de-vm)
- [5. Boas Práticas e Dicas](#5-boas-práticas-e-dicas)
- [6. Troubleshooting](#6-troubleshooting)
- [7. Próximos Passos](#7-próximos-passos)

---

## 1. Configuração das Redes Virtuais (Bridges)

Para garantir o isolamento e a organização do tráfego de rede, criaremos bridges Linux distintas para cada segmento de rede.

### 1.1. Backup da Configuração Atual

⚠️ **IMPORTANTE**: Sempre faça backup antes de editar arquivos de rede!

```bash
ssh root@<IP_DO_SEU_PROXMOX>
cp /etc/network/interfaces /etc/network/interfaces.backup.$(date +%Y%m%d-%H%M%S)
```

### 1.2. Editar Configuração de Rede

```bash
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

### 1.3. Entendendo a Configuração das Bridges

```ini
auto vmbr1
iface vmbr1 inet manual        # ← Sem IP no Proxmox (bridge pura)
    bridge-ports none          # ← Bridge pura (sem interface física associada)
    bridge-stp off             # ← Spanning Tree Protocol desabilitado
    bridge-fd 0                # ← Forward delay = 0 (sem atraso)
    comment "LAN_SERVIDORES"
```

**Por que `inet manual`?**
- As bridges não precisam de IP no host Proxmox
- O pfSense será o gateway e servidor DHCP de cada rede
- Proxmox apenas conecta as VMs entre si através das bridges

**Resumo das redes planejadas:**

| Bridge | Rede Futura | Gateway (pfSense) | Uso |
|--------|-------------|-------------------|-----|
| vmbr0 | WAN | ISP | Internet |
| vmbr1 | 192.168.1.0/24 | 192.168.1.1 | LAN_SERVIDORES (TrueNAS) |
| vmbr2 | 192.168.2.0/24 | 192.168.2.1 | LAN_CLIENTES (Desktops) |
| vmbr3 | 192.168.3.0/24 | 192.168.3.1 | LAN_K8S (Cluster) |

> **Nota**: Os IPs serão configurados no pfSense, não no Proxmox.

### 1.4. Aplicar as Configurações de Rede

**Método 1: Usando `ifreload` (Recomendado)**

```bash
# Aplicar mudanças sem derrubar conexão SSH
ifreload -a

# Verificar se as bridges foram criadas
ip link show | grep vmbr
```

**Método 2: Subir interfaces individualmente**

Se `ifreload` não estiver disponível:

```bash
ifup vmbr1
ifup vmbr2
ifup vmbr3

# Verificar status
ip link show vmbr1
ip link show vmbr2
ip link show vmbr3
```

**Método 3: Via Interface Web (Mais Seguro)**

1. Acesse: **Datacenter → Node → System → Network**
2. Clique em **Create → Linux Bridge**
3. Configure cada bridge:
   - **Bridge ID**: vmbr1, vmbr2, vmbr3
   - **IPv4/CIDR**: Deixe em branco
   - **Bridge ports**: Deixe vazio
   - **Comment**: Adicione descrição
4. Clique em **Apply Configuration**

**Método 4: Reiniciar serviço de rede (Última opção)**

⚠️ **ATENÇÃO**: Pode derrubar sua conexão SSH!

```bash
systemctl restart networking
```

Se perder SSH, acesse pelo console web do Proxmox: **Datacenter → Node → >_Shell**

### 1.5. Verificar Configuração

```bash
# 1. Listar todas as bridges
ip link show type bridge

# 2. Ver detalhes das bridges com brctl
brctl show

# 3. Verificar estado (deve mostrar UP)
ip link show vmbr1 | grep "state UP"
ip link show vmbr2 | grep "state UP"
ip link show vmbr3 | grep "state UP"

# 4. Ver configuração completa
cat /etc/network/interfaces | grep -A 5 "vmbr"

# 5. Verificar na interface web
# Datacenter → Node → System → Network
# Todas as bridges devem aparecer com status "Active"
```

**Resultado esperado:**

```
vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP>
vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP>
vmbr2: <BROADCAST,MULTICAST,UP,LOWER_UP>
vmbr3: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

---

## 2. Upload das Imagens ISO

Antes de criar as VMs, faça o upload das imagens ISO necessárias para o storage local do Proxmox.

### 2.1. Via Interface Web (Recomendado)

1. Acesse a interface web do Proxmox
2. Navegue até **Datacenter → (seu nó) → local (ou storage de sua escolha) → ISO Images**
3. Clique em **Upload** e selecione os seguintes arquivos ISO do seu computador:
   - **pfSense**: Imagem de instalação do pfSense (ex: `pfSense-CE-2.7.2-RELEASE-amd64.iso`)
   - **Ubuntu Server**: Imagem para os nós do Kubernetes (ex: `ubuntu-24.04-live-server-amd64.iso`)
   - **TrueNAS SCALE**: Imagem de instalação do TrueNAS (ex: `TrueNAS-SCALE-24.10.iso`)

### 2.2. Verificar ISOs Disponíveis

```bash
# Via CLI
pvesm list local --content iso

# Ou na interface web
# Datacenter → Node → local → ISO Images
```

---

## 3. Provisionamento das Máquinas Virtuais (VMs)

Com as redes e as ISOs prontas, o próximo passo é criar as máquinas virtuais. A criação é feita através do assistente **"Create VM"** na interface web do Proxmox.

### 3.1. Recomendações Gerais para Criação de VMs

| Configuração | Recomendação | Observação |
|--------------|--------------|------------|
| **Guest OS** | Linux 6.x kernel ou Other | Selecione conforme o SO |
| **Machine Type** | q35 (moderno) ou i440fx (legado) | q35 para Ubuntu/TrueNAS |
| **BIOS** | OVMF (UEFI) ou SeaBIOS | UEFI para sistemas modernos |
| **SCSI Controller** | VirtIO SCSI single | Melhor performance |
| **QEMU Guest Agent** | ✅ Sempre habilitar | Exceto pfSense (não suporta) |
| **Disk Bus/Device** | VirtIO Block | Melhor performance |
| **Disk Discard** | ✅ Habilitar se SSD | Melhora gestão de espaço |
| **CPU Type** | host | Melhor performance |
| **CPU Flags** | Deixe padrão | Ou adicione flags específicas |
| **Memory Ballooning** | ❌ Desabilitar para VMs críticas | Garante memória reservada |
| **Network Model** | VirtIO (paravirtualized) | Melhor performance de rede |

### 3.2. Templates de Configuração por Tipo de VM

#### 🔥 pfSense (Firewall/Router)

```yaml
General:
  VM ID: 100
  Name: pfsense
  Resource Pool: (opcional)

OS:
  ISO Image: pfSense-CE-2.7.2-RELEASE-amd64.iso
  Type: Other
  
System:
  Graphic card: Default
  Machine: Default (i440fx)
  BIOS: Default (SeaBIOS)
  SCSI Controller: VirtIO SCSI single
  Qemu Agent: ❌ Disabled (pfSense não suporta nativamente)

Disks:
  scsi0: 32 GB
    Bus/Device: VirtIO Block
    Storage: local-lvm (ou seu storage preferido)
    Cache: Write back (ou Default)
    Discard: ✅ Se SSD
    IO thread: ✅ Enabled

CPU:
  Sockets: 1
  Cores: 2
  Type: host
  
Memory:
  Memory (MiB): 2048 (2GB)
  Minimum memory: 2048
  Ballooning Device: ❌ Disabled

Network:
  net0 (WAN):
    Bridge: vmbr0
    Model: VirtIO (paravirtualized)
    Firewall: ❌ Disabled (pfSense gerencia)
  net1 (LAN_SERVIDORES):
    Bridge: vmbr1
    Model: VirtIO
  net2 (LAN_CLIENTES):
    Bridge: vmbr2
    Model: VirtIO
  net3 (LAN_K8S):
    Bridge: vmbr3
    Model: VirtIO

Confirm:
  ✅ Start after created (opcional)
```

**Comandos via CLI (alternativa):**

```bash
qm create 100 --name pfsense --memory 2048 --balloon 0 --cores 2 --cpu host \
  --net0 virtio,bridge=vmbr0 \
  --net1 virtio,bridge=vmbr1 \
  --net2 virtio,bridge=vmbr2 \
  --net3 virtio,bridge=vmbr3 \
  --scsi0 local-lvm:32 --scsihw virtio-scsi-single \
  --ide2 local:iso/pfSense-CE-2.7.2-RELEASE-amd64.iso,media=cdrom \
  --boot order=scsi0 --ostype other
```

---

#### 💾 TrueNAS SCALE (Storage NFS)

```yaml
General:
  VM ID: 200
  Name: truenas
  
OS:
  ISO Image: TrueNAS-SCALE-24.10.0.iso
  Type: Linux 6.x - 2.6 Kernel

System:
  Graphic card: Default
  Machine: q35
  BIOS: OVMF (UEFI)
  Add EFI Disk: ✅ Yes
  EFI Storage: local-lvm
  SCSI Controller: VirtIO SCSI single
  Qemu Agent: ✅ Enabled

Disks:
  scsi0 (Boot): 32 GB
    Storage: local-lvm
    Bus/Device: VirtIO Block
    Cache: Write back
    Discard: ✅ Se SSD
    IO thread: ✅ Enabled
    
  scsi1 (Data Pool 1): 100 GB
    Storage: local-lvm
    Bus/Device: VirtIO Block
    Discard: ✅ Se SSD
    
  scsi2 (Data Pool 2): 100 GB
    Storage: local-lvm
    Bus/Device: VirtIO Block
    Discard: ✅ Se SSD
  
    scsi2 (Data Pool 3): 100 GB
    Storage: local-lvm
    Bus/Device: VirtIO Block
    Discard: ✅ Se SSD
  # Adicione mais discos conforme necessário para seus pools ZFS

CPU:
  Sockets: 1
  Cores: 4
  Type: host

Memory:
  Memory (MiB): 8192 (8GB mínimo, 16GB recomendado para ZFS)
  Minimum memory: 8192
  Ballooning Device: ❌ Disabled

Network:
  net0:
    Bridge: vmbr1 (LAN_SERVIDORES)
    Model: VirtIO (paravirtualized)
    Firewall: ❌ Disabled
```

**Comandos via CLI:**

```bash
qm create 200 --name truenas --memory 8192 --balloon 0 --cores 4 --cpu host \
  --net0 virtio,bridge=vmbr1 \
  --scsi0 local-lvm:32 --scsi1 local-lvm:100 --scsi2 local-lvm:100 --scsi3 local-lvm:100\
  --scsihw virtio-scsi-single --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1 --agent enabled=1 \
  --ide2 local:iso/TrueNAS-SCALE-24.10.0.iso,media=cdrom \
  --boot order=scsi0 --ostype l26
```

---

#### 🔄 HAProxy Load Balancers (2 VMs)

```yaml
General:
  VM IDs: 291, 292
  Names: k8s-lb-01, k8s-lb-02

OS:
  ISO Image: ubuntu-24.04-live-server-amd64.iso
  Type: Linux 6.x - 2.6 Kernel

System:
  Machine: q35
  BIOS: OVMF (UEFI)
  Add EFI Disk: ✅ Yes
  SCSI Controller: VirtIO SCSI single
  Qemu Agent: ✅ Enabled

Disks:
  scsi0: 20 GB
    Bus/Device: VirtIO Block
    Discard: ✅ Se SSD

CPU:
  Sockets: 1
  Cores: 2
  Type: host

Memory:
  Memory (MiB): 2048 (2GB)
  Ballooning Device: ❌ Disabled

Network:
  net0:
    Bridge: vmbr3 (LAN_K8S)
    Model: VirtIO
```

**Criar ambos LBs em sequência:**

```bash
# LB-01
qm create 291 --name k8s-lb-01 --memory 2048 --balloon 0 --cores 2 --cpu host \
  --net0 virtio,bridge=vmbr3 --scsi0 local-lvm:20 \
  --scsihw virtio-scsi-single --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1 --agent enabled=1 \
  --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
  --boot order=scsi0 --ostype l26

# LB-02
qm create 292 --name k8s-lb-02 --memory 2048 --balloon 0 --cores 2 --cpu host \
  --net0 virtio,bridge=vmbr3 --scsi0 local-lvm:20 \
  --scsihw virtio-scsi-single --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1 --agent enabled=1 \
  --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
  --boot order=scsi0 --ostype l26
```

---

#### ☸️ Kubernetes Control Planes (3 VMs)

```yaml
General:
  VM IDs: 301, 302, 303
  Names: k8s-control-01, k8s-control-02, k8s-control-03

OS:
  ISO Image: ubuntu-24.04-live-server-amd64.iso
  Type: Linux 6.x - 2.6 Kernel

System:
  Machine: q35
  BIOS: OVMF (UEFI)
  Add EFI Disk: ✅ Yes
  SCSI Controller: VirtIO SCSI single
  Qemu Agent: ✅ Enabled

Disks:
  scsi0: 50 GB
    Bus/Device: VirtIO Block
    Discard: ✅ Se SSD
    IO thread: ✅ Enabled

CPU:
  Sockets: 1
  Cores: 2 (mínimo, 4 recomendado)
  Type: host

Memory:
  Memory (MiB): 4096 (4GB mínimo, 8GB recomendado)
  Ballooning Device: ❌ Disabled

Network:
  net0:
    Bridge: vmbr3 (LAN_K8S)
    Model: VirtIO
```

**Criar os 3 Control Planes:**

```bash
for i in {1..3}; do
  qm create 30$i --name k8s-control-0$i --memory 4096 --balloon 0 --cores 2 --cpu host \
    --net0 virtio,bridge=vmbr3 --scsi0 local-lvm:50 \
    --scsihw virtio-scsi-single --machine q35 --bios ovmf \
    --efidisk0 local-lvm:1 --agent enabled=1 \
    --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
    --boot order=scsi0 --ostype l26
done
```

---

#### 🔧 Kubernetes Workers (2 VMs)

```yaml
General:
  VM IDs: 311, 312
  Names: k8s-worker-01, k8s-worker-02

OS:
  ISO Image: ubuntu-24.04-live-server-amd64.iso
  Type: Linux 6.x - 2.6 Kernel

System:
  Machine: q35
  BIOS: OVMF (UEFI)
  Add EFI Disk: ✅ Yes
  SCSI Controller: VirtIO SCSI single
  Qemu Agent: ✅ Enabled

Disks:
  scsi0: 100 GB (ou mais, dependendo das cargas de trabalho)
    Bus/Device: VirtIO Block
    Discard: ✅ Se SSD
    IO thread: ✅ Enabled

CPU:
  Sockets: 1
  Cores: 4 (mínimo, 8 recomendado para produção)
  Type: host

Memory:
  Memory (MiB): 8192 (8GB mínimo, 16GB+ recomendado)
  Ballooning Device: ❌ Disabled

Network:
  net0:
    Bridge: vmbr3 (LAN_K8S)
    Model: VirtIO
```

**Criar os 2 Workers:**

```bash
for i in {1..2}; do
  qm create 31$i --name k8s-worker-0$i --memory 8192 --balloon 0 --cores 4 --cpu host \
    --net0 virtio,bridge=vmbr3 --scsi0 local-lvm:100 \
    --scsihw virtio-scsi-single --machine q35 --bios ovmf \
    --efidisk0 local-lvm:1 --agent enabled=1 \
    --ide2 local:iso/ubuntu-24.04-live-server-amd64.iso,media=cdrom \
    --boot order=scsi0 --ostype l26
done
```

---

### 3.3. Resumo das VMs Criadas

| VM ID | Nome | Tipo | CPU | RAM | Disco | Bridge |
|-------|------|------|-----|-----|-------|--------|
| 100 | pfsense | Firewall | 2 | 2GB | 32GB | vmbr0-3 |
| 200 | truenas | Storage | 4 | 8GB | 32+100+100GB+100GB | vmbr1 |
| 291 | k8s-lb-01 | Load Balancer | 2 | 2GB | 20GB | vmbr3 |
| 292 | k8s-lb-02 | Load Balancer | 2 | 2GB | 20GB | vmbr3 |
| 301 | k8s-control-01 | Control Plane | 2 | 4GB | 50GB | vmbr3 |
| 302 | k8s-control-02 | Control Plane | 2 | 4GB | 50GB | vmbr3 |
| 303 | k8s-control-03 | Control Plane | 2 | 4GB | 50GB | vmbr3 |
| 311 | k8s-worker-01 | Worker | 4 | 8GB | 100GB | vmbr3 |
| 312 | k8s-worker-02 | Worker | 4 | 8GB | 100GB | vmbr3 |

**Total de recursos necessários:**
- **CPUs**: 24 cores
- **RAM**: 40 GB
- **Storage**: ~694 GB

´**OBS: Neste exemplo segui a sugestão da documentação das ferramentas. É possivel alterar a quantidade de memoria e nucleos de processador conforme disponibilidade/necessidade.**´

### 3.4. Conectando as VMs às Bridges Corretas

⚠️ **Atenção crítica**: Ao criar cada VM, na aba **Network**, certifique-se de selecionar a **Bridge** correta para sua função:

- **VM do pfSense**: 4 interfaces de rede
  - `net0`: vmbr0 (WAN - Internet)
  - `net1`: vmbr1 (LAN_SERVIDORES)
  - `net2`: vmbr2 (LAN_CLIENTES)
  - `net3`: vmbr3 (LAN_K8S)

- **VM do TrueNAS**:
  - `net0`: vmbr1 (LAN_SERVIDORES)

- **VMs do Cluster Kubernetes** (Load Balancers, Control Planes, Workers):
  - `net0`: vmbr3 (LAN_K8S)

### 3.5. Instalar QEMU Guest Agent nas VMs

⚠️ **Importante**: Após instalar o sistema operacional em cada VM (exceto pfSense), instale o QEMU Guest Agent.

**Para Ubuntu/Debian (Load Balancers, Control Planes, Workers):**

```bash
# Após boot da VM e instalação do Ubuntu
sudo apt update
sudo apt install qemu-guest-agent -y

# Habilitar e iniciar serviço
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent

# Verificar se está rodando
sudo systemctl status qemu-guest-agent

# Ver versão
qemu-ga --version
```

**Para TrueNAS SCALE:**

O TrueNAS já vem com o guest agent, mas verifique se está habilitado:

1. Acesse TrueNAS Web UI
2. **System Settings → Advanced**
3. Procure por "QEMU Guest Agent"
4. Certifique-se que está habilitado

**Benefícios do Guest Agent:**
- ✅ Shutdown/reboot graceful pelo Proxmox
- ✅ Informações de IP/hostname na interface web do Proxmox
- ✅ Snapshots consistentes (freeze/thaw do filesystem)
- ✅ Sincronização de relógio
- ✅ Melhor integração Proxmox ↔ VM

---

## 4. Checklist Pós-Criação de VM

Após criar e instalar o SO em cada VM, siga este checklist:

### 4.1. Verificações Básicas

- [ ] **Sistema operacional instalado** e funcionando
- [ ] **Hostname configurado** corretamente
  ```bash
  # Ver hostname atual
  hostnamectl
  
  # Configurar se necessário
  sudo hostnamectl set-hostname k8s-control-01
  ```

- [ ] **QEMU Guest Agent** instalado e rodando (exceto pfSense)
  ```bash
  sudo systemctl status qemu-guest-agent
  ```

- [ ] **IP configurado** (estático ou via DHCP)
  ```bash
  ip addr show
  ```

- [ ] **Conectividade de rede** testada
  ```bash
  # Ping para gateway (após configurar pfSense)
  ping -c 3 192.168.3.1
  
  # Ping para internet (se já configurado)
  ping -c 3 8.8.8.8
  ```

- [ ] **Atualizações aplicadas**
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- [ ] **Configuração de `/etc/hosts`** (se aplicável)
  ```bash
  sudo nano /etc/hosts
  # Adicionar entradas dos outros nodes
  ```

- [ ] **SSH habilitado** e funcionando
  ```bash
  sudo systemctl status ssh
  ```

- [ ] **Firewall configurado** (UFW ou iptables, se necessário)
  ```bash
  # Ubuntu - habilitar UFW depois de configurar regras
  sudo ufw status
  ```

### 4.2. No Proxmox

- [ ] **Snapshot inicial criado** (estado limpo pós-instalação)
  ```bash
  # Via CLI
  qm snapshot 301 clean-install --description "Fresh Ubuntu install"
  
  # Ou via web: VM → Snapshots → Take Snapshot
  ```

- [ ] **Tags adicionadas** para organização
  - Web UI: VM → Options → Tags
  - Exemplos: `k8s-control`, `k8s-worker`, `infrastructure`

- [ ] **Notes/Description** preenchido
  - Web UI: VM → Notes
  - Adicionar informações úteis (IP, função, etc.)

- [ ] **Backup configurado** (opcional mas recomendado)
  ```bash
  # Configurar via: Datacenter → Backup
  # Ou manualmente:
  vzdump 301 --mode snapshot --compress zstd --storage local
  ```

### 4.3. Documentação

Mantenha um registro das VMs criadas:

```markdown
## Inventário de VMs

### pfSense (VM 100)
- **IPs**: 
  - WAN: DHCP (do ISP)
  - LAN_SERVIDORES: 192.168.1.1/24
  - LAN_CLIENTES: 192.168.2.1/24
  - LAN_K8S: 192.168.3.1/24
- **Credenciais**: admin / (senha segura)
- **Função**: Firewall e roteamento entre redes

### TrueNAS (VM 200)
- **IP**: 192.168.1.10 (LAN_SERVIDORES)
- **Função**: Storage NFS para Kubernetes
- **Pools**: tank (2x100GB em mirror)

### Load Balancers
- **k8s-lb-01** (291): 192.168.3.11
- **k8s-lb-02** (292): 192.168.3.12
- **VIP Keepalived**: 192.168.3.10

### Control Planes
- **k8s-control-01** (301): 192.168.3.21
- **k8s-control-02** (302): 192.168.3.22
- **k8s-control-03** (303): 192.168.3.23

### Workers
- **k8s-worker-01** (311): 192.168.3.31
- **k8s-worker-02** (312): 192.168.3.32
```

---

## 5. Boas Práticas e Dicas

### 5.1. Snapshots antes de Mudanças Críticas

Sempre crie snapshots antes de:
- Atualizar kernel
- Fazer mudanças de rede
- Instalar software crítico
- Alterar configurações do Kubernetes

```bash
# Via CLI do Proxmox
qm snapshot <VMID> <snapshot-name> --description "Descrição"

# Exemplo: Antes de atualizar kernel no control-plane-01
qm snapshot 301 pre-kernel-5.19 --description "Antes de atualizar kernel para 5.19"

# Listar snapshots
qm lis