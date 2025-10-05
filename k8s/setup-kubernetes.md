# Guia Completo: Setup de VMs Ubuntu 24.04 para Cluster Kubernetes no Proxmox

## Sumário
1. [Arquitetura do Ambiente](#arquitetura)
2. [Preparação do Proxmox](#preparacao-proxmox)
3. [Download e Preparação da ISO Ubuntu](#iso-ubuntu)
4. [Criação das VMs (Load Balancers)](#vms-load-balancers)
5. [Criação das VMs (Control Planes)](#vms-control-planes)
6. [Criação das VMs (Workers)](#vms-workers)
7. [Configuração Base Ubuntu (Todas VMs)](#config-base-ubuntu)
8. [Configuração de Rede Estática](#config-rede)
9. [Instalação HAProxy + Keepalived](#haproxy-keepalived)
10. [Preparação para Kubernetes](#prep-kubernetes)
11. [Checklist Final](#checklist)

---

## Arquitetura do Ambiente {#arquitetura}

### Topologia de Rede
```
Internet (Provedor)
    ↓
192.168.0.0/24 (WAN - vmbr0)
    ↓
[pfSense Firewall]
    ├─→ 192.168.1.0/24 (LAN Servidores - vmbr1)
    ├─→ 192.168.2.0/24 (LAN Clientes - vmbr2)
    └─→ 192.168.3.0/24 (LAN_K8S - vmbr3) ← NOVA REDE
```

### Endereçamento IP do Cluster

| Hostname | Função | IP | vCPUs | RAM | Disco | Bridge |
|----------|--------|-----|-------|-----|-------|--------|
| **pfSense** | Gateway | 192.168.3.1 | - | - | - | vmbr3 |
| **VIP (Keepalived)** | API K8s HA | 192.168.3.10 | - | - | - | - |
| **lb-node01** | Load Balancer 1 | 192.168.3.11 | 2 | 2GB | 10GB | vmbr3 |
| **lb-node02** | Load Balancer 2 | 192.168.3.12 | 2 | 2GB | 10GB | vmbr3 |
| **k8s-control01** | Control Plane 1 | 192.168.3.21 | 2 | 4GB | 20GB | vmbr3 |
| **k8s-control02** | Control Plane 2 | 192.168.3.22 | 2 | 4GB | 20GB | vmbr3 |
| **k8s-control03** | Control Plane 3 | 192.168.3.23 | 2 | 4GB | 20GB | vmbr3 |
| **k8s-worker01** | Worker Node 1 | 192.168.3.31 | 4 | 8GB | 50GB | vmbr3 |
| **k8s-worker02** | Worker Node 2 | 192.168.3.32 | 4 | 8GB | 50GB | vmbr3 |

**Total de recursos necessários:**
- vCPUs: 20
- RAM: 34 GB
- Disco: 170 GB

---

## Preparação do Proxmox {#preparacao-proxmox}

### Passo 1: Criar a Bridge vmbr3 (Rede K8S)

SSH no host Proxmox:

```bash
ssh root@<IP_PROXMOX>
```

Edite a configuração de rede:

```bash
nano /etc/network/interfaces
```

Adicione ao final do arquivo:

```bash
# Rede isolada para Kubernetes
auto vmbr3
iface vmbr3 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment "LAN_K8S - Rede isolada para cluster Kubernetes"
```

Salve e aplique:

```bash
systemctl restart networking

# Verifique
ip a | grep vmbr3
```

### Passo 2: Configurar Interface no pfSense

**Via Web GUI do pfSense:**

1. Acesse pfSense: `https://192.168.0.100` (ou seu IP)
2. **Interfaces → Assignments**
3. Clique em **Add** para adicionar a nova interface
4. Clique no nome da nova interface (geralmente `OPT2`)
5. Configure:
   ```
   Enable: ✅
   Description: LAN_K8S
   IPv4 Configuration Type: Static IPv4
   IPv4 Address: 192.168.3.1 / 24
   ```
6. **Save → Apply Changes**

### Passo 3: Configurar DHCP (Opcional - usaremos IP estático)

**Services → DHCP Server → LAN_K8S:**

```
Enable: ❌ (desabilitado - usaremos IPs estáticos)
```

### Passo 4: Criar Aliases no pfSense

**Firewall → Aliases:**

#### Alias 1: K8S_NET
```
Name: K8S_NET
Type: Network(s)
Network: 192.168.3.0/24
Description: Rede do cluster Kubernetes
```

#### Alias 2: K8S_API_SERVERS
```
Name: K8S_API_SERVERS
Type: Host(s)
Hosts: 
  192.168.3.21
  192.168.3.22
  192.168.3.23
Description: Control Planes Kubernetes
```

#### Alias 3: K8S_LB
```
Name: K8S_LB
Type: Host(s)
Hosts:
  192.168.3.11
  192.168.3.12
Description: Load Balancers HAProxy
```

**Save → Apply Changes**

### Passo 5: Configurar Regras de Firewall

**Firewall → Rules → LAN_K8S:**

#### Regra 1: DNS ao pfSense
```
Action: Pass
Interface: LAN_K8S
Protocol: TCP/UDP
Source: LAN_K8S net
Destination: LAN_K8S address
Destination Port: 53 (DNS)
Description: K8S → pfSense: DNS
```

#### Regra 2: Comunicação interna K8S
```
Action: Pass
Interface: LAN_K8S
Protocol: Any
Source: LAN_K8S net
Destination: LAN_K8S net
Description: K8S → K8S: Comunicação interna cluster
```

#### Regra 3: Acesso à Internet (HTTP/HTTPS)
```
Action: Pass
Interface: LAN_K8S
Protocol: TCP
Source: LAN_K8S net
Destination: Any
Destination Port: 80, 443
Description: K8S → Internet: Download de imagens
```

#### Regra 4: NTP
```
Action: Pass
Interface: LAN_K8S
Protocol: UDP
Source: LAN_K8S net
Destination: Any
Destination Port: 123
Description: K8S → Internet: Sincronização de tempo
```

#### Regra 5: ICMP
```
Action: Pass
Interface: LAN_K8S
Protocol: ICMP
Source: LAN_K8S net
Destination: Any
Description: K8S → Any: Ping para diagnóstico
```

#### Regra 6: Acesso API K8s da LAN Servidores (CI/CD)
```
Action: Pass
Interface: LAN (Servidores)
Protocol: TCP
Source: LAN net (192.168.1.0/24)
Destination: 192.168.3.10 (VIP)
Destination Port: 6443
Description: Servidores → K8S API: Acesso para CI/CD
```

**Apply Changes**

---

## Download e Preparação da ISO Ubuntu {#iso-ubuntu}

### Passo 1: Download Ubuntu Server 24.04 LTS

No host Proxmox ou baixe e faça upload:

```bash
# Baixar direto no Proxmox
cd /var/lib/vz/template/iso

wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# Verificar
ls -lh ubuntu-24.04-live-server-amd64.iso
```

**Alternativa:** Baixe no seu PC e faça upload via Web GUI:
- **Datacenter → seu-node → local → ISO Images → Upload**

---

## Criação das VMs (Load Balancers) {#vms-load-balancers}

Vamos criar as 2 VMs de Load Balancer primeiro.

### Via Web GUI do Proxmox:

#### VM 1: lb-node01

1. Clique em **Create VM**
2. **General:**
   ```
   Node: (seu node)
   VM ID: 301
   Name: lb-node01
   ```

3. **OS:**
   ```
   Use CD/DVD disc image file (iso)
   Storage: local
   ISO image: ubuntu-24.04-live-server-amd64.iso
   Guest OS:
     Type: Linux
     Version: 6.x - 2.6 Kernel
   ```

4. **System:**
   ```
   Graphic card: Default
   Machine: Default (i440fx)
   BIOS: Default (SeaBIOS)
   SCSI Controller: VirtIO SCSI single
   Qemu Agent: ✅ (habilite)
   ```

5. **Disks:**
   ```
   Bus/Device: VirtIO Block
   Storage: local-lvm (ou seu storage)
   Disk size (GiB): 10
   Cache: Default (No cache)
   Discard: ✅ (se usar SSD)
   SSD emulation: ✅ (opcional)
   ```

6. **CPU:**
   ```
   Sockets: 1
   Cores: 2
   Type: host (ou x86-64-v2-AES)
   ```

7. **Memory:**
   ```
   Memory (MiB): 2048 (2GB)
   Minimum memory: 2048
   Ballooning Device: ✅
   ```

8. **Network:**
   ```
   Bridge: vmbr3
   VLAN Tag: (vazio)
   Firewall: ✅
   Model: VirtIO (paravirtualized)
   MAC address: (auto)
   ```

9. **Confirm:**
   - Revise tudo
   - ❌ **NÃO marque "Start after created"**
   - **Finish**

#### VM 2: lb-node02

Repita o processo acima com:
- **VM ID:** 302
- **Name:** lb-node02
- Demais configurações **idênticas**

---

## Criação das VMs (Control Planes) {#vms-control-planes}

### VM 3: k8s-control01

1. **Create VM**
2. **General:**
   ```
   VM ID: 311
   Name: k8s-control01
   ```

3. **OS:** (mesmo ISO do Ubuntu)

4. **System:** (igual load balancer)

5. **Disks:**
   ```
   Disk size (GiB): 20
   ```

6. **CPU:**
   ```
   Cores: 2
   ```

7. **Memory:**
   ```
   Memory (MiB): 4096 (4GB)
   ```

8. **Network:**
   ```
   Bridge: vmbr3
   Model: VirtIO
   ```

### VM 4 e 5: k8s-control02 e k8s-control03

Repita para:
- **VM ID:** 312 e 313
- **Name:** k8s-control02 e k8s-control03
- Configurações idênticas ao control01

---

## Criação das VMs (Workers) {#vms-workers}

### VM 6: k8s-worker01

1. **Create VM**
2. **General:**
   ```
   VM ID: 321
   Name: k8s-worker01
   ```

3. **OS:** (mesmo ISO)

4. **System:** (padrão)

5. **Disks:**
   ```
   Disk size (GiB): 50
   ```

6. **CPU:**
   ```
   Cores: 4
   ```

7. **Memory:**
   ```
   Memory (MiB): 8192 (8GB)
   ```

8. **Network:**
   ```
   Bridge: vmbr3
   Model: VirtIO
   ```

### VM 7: k8s-worker02

Repita com:
- **VM ID:** 322
- **Name:** k8s-worker02
- Configurações idênticas ao worker01

---

## Configuração Base Ubuntu (Todas VMs) {#config-base-ubuntu}

Execute este processo em **TODAS as 7 VMs**.

### Passo 1: Iniciar VM e Instalação

1. Selecione a VM no Proxmox
2. Clique em **Start**
3. Abra o **Console**
4. Aguarde o boot do instalador Ubuntu

### Passo 2: Instalação do Ubuntu Server

#### Tela 1: Idioma
```
English (ou Português se preferir)
```

#### Tela 2: Keyboard
```
Layout: English (US) ou Portuguese (Brazil)
```

#### Tela 3: Type of install
```
⦿ Ubuntu Server (minimized)
```

#### Tela 4: Network connections

**IMPORTANTE:** Aqui você configurará o IP estático.

**Para lb-node01 (exemplo):**

1. Selecione `ens18` (ou nome da interface)
2. Pressione **Enter**
3. Escolha: **Edit IPv4**
4. Selecione: **Manual**
5. Configure:
   ```
   Subnet: 192.168.3.0/24
   Address: 192.168.3.11
   Gateway: 192.168.3.1
   Name servers: 192.168.3.1,1.1.1.1
   Search domains: (deixe vazio ou "local")
   ```
6. **Save**

**Repita para cada VM com o IP correspondente:**
- lb-node01: 192.168.3.11
- lb-node02: 192.168.3.12
- k8s-control01: 192.168.3.21
- k8s-control02: 192.168.3.22
- k8s-control03: 192.168.3.23
- k8s-worker01: 192.168.3.31
- k8s-worker02: 192.168.3.32

#### Tela 5: Proxy
```
(deixe vazio)
```

#### Tela 6: Mirror
```
(aceite o padrão)
```

#### Tela 7: Storage
```
⦿ Use an entire disk
Disco: (selecione o disco disponível)
Set up this disk as an LVM group: ✅
```

#### Tela 8: Storage confirmation
```
Continue
```

#### Tela 9: Profile setup

**Configure conforme sua preferência:**
```
Your name: Mizael (ou seu nome)
Your server's name: lb-node01 (conforme a VM)
Pick a username: mizael
Choose a password: **********
Confirm your password: **********
```

**IMPORTANTE:** Anote usuário e senha!

#### Tela 10: SSH Setup
```
✅ Install OpenSSH server
⬜ Import SSH identity: (deixe desmarcado por enquanto)
```

#### Tela 11: Featured Server Snaps
```
(não selecione nada - instalação minimal)
```

#### Aguarde a instalação

Quando aparecer **"Install complete!"**:
```
Reboot Now
```

### Passo 3: Remover ISO da VM

**Antes que a VM reinicie:**

1. No Proxmox, selecione a VM
2. **Hardware → CD/DVD Drive**
3. Clique em **Edit**
4. Selecione: **Do not use any media**
5. **OK**

Isso evita que a VM boote novamente pelo instalador.

### Passo 4: Primeiro Boot

Aguarde o boot e faça login:
```
username: mizael (ou o que você criou)
password: (sua senha)
```

### Passo 5: Atualizar Sistema

```bash
sudo apt update
sudo apt upgrade -y
sudo apt dist-upgrade -y
```

### Passo 6: Instalar pacotes essenciais

```bash
sudo apt install -y \
    curl \
    wget \
    vim \
    git \
    net-tools \
    htop \
    iotop \
    qemu-guest-agent
```

### Passo 7: Habilitar QEMU Guest Agent

```bash
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

### Passo 8: Configurar hostname (caso não tenha configurado na instalação)

**Para lb-node01 (exemplo):**
```bash
sudo hostnamectl set-hostname lb-node01
```

Edite `/etc/hosts`:
```bash
sudo nano /etc/hosts
```

Adicione:
```
192.168.3.11 lb-node01
```

### Passo 9: Verificar rede

```bash
ip a
ping -c 3 192.168.3.1
ping -c 3 1.1.1.1
ping -c 3 google.com
```

Tudo deve funcionar.

---

## Configuração de Rede Estática (Ajustes se necessário) {#config-rede}

Se você configurou durante a instalação, este passo é apenas validação.

### Verificar configuração atual:

```bash
cat /etc/netplan/50-cloud-init.yaml
```

**Deve estar assim (exemplo lb-node01):**

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 192.168.3.11/24
      nameservers:
        addresses:
          - 192.168.3.1
          - 1.1.1.1
        search: []
      routes:
        - to: default
          via: 192.168.3.1
```

Se precisar ajustar:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Aplique:

```bash
sudo netplan apply
```

---

## Instalação HAProxy + Keepalived (Apenas LB Nodes) {#haproxy-keepalived}

Execute **APENAS** em `lb-node01` e `lb-node02`.

### Passo 1: Instalar pacotes

```bash
sudo apt install -y haproxy keepalived
```

### Passo 2: Configurar HAProxy

**Em AMBOS lb-node01 e lb-node02:**

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
```

**Substitua todo o conteúdo por:**

```cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 2000

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend kubernetes_api_frontend
    bind *:6443
    mode tcp
    default_backend kubernetes_api_backend

backend kubernetes_api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    default-server inter 10s fall 2 rise 2
    server control01 192.168.3.21:6443 check
    server control02 192.168.3.22:6443 check
    server control03 192.168.3.23:6443 check

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:admin
```

**Salve e saia (Ctrl+O, Enter, Ctrl+X)**

**Verificar configuração:**

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Deve retornar: `Configuration file is valid`

**Habilitar e iniciar:**

```bash
sudo systemctl enable haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

### Passo 3: Configurar Keepalived

#### No lb-node01 (MASTER):

```bash
sudo nano /etc/keepalived/keepalived.conf
```

**Conteúdo:**

```cfg
vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens18
    virtual_router_id 51
    priority 101
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass K8sHAP4ssw0rd
    }
    
    virtual_ipaddress {
        192.168.3.10/24 dev ens18
    }
    
    track_script {
        check_haproxy
    }
}
```

#### No lb-node02 (BACKUP):

```bash
sudo nano /etc/keepalived/keepalived.conf
```

**Conteúdo:**

```cfg
vrrp_script check_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass K8sHAP4ssw0rd
    }
    
    virtual_ipaddress {
        192.168.3.10/24 dev ens18
    }
    
    track_script {
        check_haproxy
    }
}
```

**Diferença:** `state BACKUP` e `priority 100`

**Habilitar em ambos:**

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
```

### Passo 4: Verificar VIP

**No lb-node01:**

```bash
ip a | grep 192.168.3.10
```

Deve aparecer:
```
inet 192.168.3.10/24 scope global secondary ens18
```

**Teste de failover:**

```bash
# No lb-node01, pare o keepalived
sudo systemctl stop keepalived

# Verifique que o VIP sumiu
ip a | grep 192.168.3.10

# No lb-node02, verifique que o VIP apareceu
ssh mizael@192.168.3.12
ip a | grep 192.168.3.10
```

**Reative no lb-node01:**

```bash
sudo systemctl start keepalived
```

---

## Preparação para Kubernetes (Control Planes + Workers) {#prep-kubernetes}

Execute em **TODAS** as VMs (control planes e workers).

### Passo 1: Desabilitar Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Verificar:

```bash
free -h
# Linha "Swap" deve estar zerada
```

### Passo 2: Configurar módulos do kernel

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Passo 3: Configurar parâmetros sysctl

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Verificar:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### Passo 4: Instalar containerd

```bash
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Ajustar para usar systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Passo 5: Instalar kubeadm, kubelet, kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Passo 6: Configurar kubelet

```bash
sudo systemctl enable kubelet
```

**Ainda NÃO vai iniciar - é normal. Só iniciará após o `kubeadm init/join`.**

### Passo 7: Configurar /etc/hosts (em todas VMs)

```bash
sudo nano /etc/hosts
```

Adicione:

```
192.168.3.1     pfsense
192.168.3.10    k8s-api vip
192.168.3.11    lb-node01
192.168.3.12    lb-node02
192.168.3.21    k8s-control01
192.168.3.22    k8s-control02
192.168.3.23    k8s-control03
192.168.3.31    k8s-worker01
192.168.3.32    k8s-worker02
```

---

## Checklist Final {#checklist}

### Proxmox:
- [ ] Bridge vmbr3 criada
- [ ] Todas as 7 VMs criadas
- [ ] VMs conectadas à vmbr3
- [ ] QEMU Guest Agent habilitado em todas VMs

### pfSense:
- [ ] Interface LAN_K8S configurada (192.168.3.1/24)
- [ ] Aliases criados (K8S_NET, K8S_API_SERVERS, K8S_LB)
- [ ] Regras de firewall configuradas
- [ ] Conectividade testada

### Load Balancers (lb-node01 e lb-node02):
- [ ] Ubuntu 24.04 instalado
- [ ] Rede estática configurada
- [ ] HAProxy instalado e configurado
- [ ] Keepalived instalado e configurado
- [ ] VIP (192.168.3.10) ativo
- [ ] Failover testado

### Control Planes (k8s-control01, 02, 03):
- [ ] Ubuntu 24.04 instalado
- [ ] Rede estática configurada
- [ ] Swap desabilitado
- [ ] Módulos do kernel carregados
- [ ] containerd instalado
- [ ] kubeadm, kubelet, kubectl instalados
- [ ] /etc/hosts configurado

### Workers (k8s-worker01, 02):
- [ ] Ubuntu 24.04 instalado
- [ ] Rede estática configurada
- [ ] Swap desabilitado
- [ ] Módulos do kernel carregados
- [ ] containerd instalado
- [ ] kubeadm, kubelet, kubectl instalados
- [ ] /etc/hosts configurado

### Testes de Conectividade:

Execute de qualquer VM do cluster:

```bash
# Testar gateway
ping -c 3 192.168.3.1

# Testar VIP
ping -c 3 192.168.3.10

# Testar internet
ping -c 3 1.1.1.1
ping -c 3 google.com

# Testar DNS
nslookup google.com
```

---

## Próximos Passos

Agora que toda a infraestrutura está pronta, você pode:

1. **Inicializar o cluster Kubernetes** no primeiro control plane
2. **Adicionar os demais control planes** ao cluster
3. **Adicionar os workers** ao cluster
4. **Instalar CNI** (Calico, Flannel, ou Cilium)
5. **Testar deployment** de uma aplicação

**Comando para iniciar (no k8s-control01):**

```bash
sudo kubeadm init \
  --control-plane-endpoint "192.168.3.10:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
```

Guarde o output com os comandos `kubeadm join` para adicionar os demais nós!

---

## Inicialização do Cluster Kubernetes

### Passo 1: Inicializar o Primeiro Control Plane

**Execute APENAS no k8s-control01:**

```bash
sudo kubeadm init \
  --control-plane-endpoint "192.168.3.10:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.3.21
```

**Parâmetros explicados:**
- `--control-plane-endpoint`: VIP do HAProxy/Keepalived
- `--upload-certs`: Permite adicionar outros control planes facilmente
- `--pod-network-cidr`: Rede dos pods (necessário para Flannel/Calico)
- `--apiserver-advertise-address`: IP do control plane atual

**IMPORTANTE:** Guarde TODO o output! Ele contém 3 comandos importantes:

1. Comando para configurar kubectl
2. Comando para adicionar control planes
3. Comando para adicionar workers

**Exemplo de output:**

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You can now join any number of control-plane nodes by running the following command as root:

  kubeadm join 192.168.3.10:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:xxxx \
    --control-plane --certificate-key yyyy

Then you can join any number of worker nodes by running the following as root:

  kubeadm join 192.168.3.10:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:xxxx
```

### Passo 2: Configurar kubectl no control01

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verifique:

```bash
kubectl get nodes
```

Deve mostrar:

```
NAME            STATUS     ROLES           AGE   VERSION
k8s-control01   NotReady   control-plane   1m    v1.31.x
```

**Status "NotReady" é normal** - falta instalar o CNI (rede dos pods).

### Passo 3: Instalar CNI (Flannel)

**No k8s-control01:**

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Aguarde ~30 segundos e verifique:

```bash
kubectl get nodes
```

Agora deve mostrar:

```
NAME            STATUS   ROLES           AGE   VERSION
k8s-control01   Ready    control-plane   2m    v1.31.x
```

Verifique os pods do sistema:

```bash
kubectl get pods -n kube-system
```

Todos devem estar `Running` em alguns minutos.

### Passo 4: Adicionar os demais Control Planes

**No k8s-control02 e k8s-control03:**

Use o comando `kubeadm join` que você guardou do output do `kubeadm init`, com a flag `--control-plane`:

```bash
sudo kubeadm join 192.168.3.10:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxx \
  --control-plane --certificate-key yyyyyyyyyyyy \
  --apiserver-advertise-address=192.168.3.22  # Mude para .23 no control03
```

**Após o join bem-sucedido em cada control plane, configure o kubectl:**

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Verifique do control01:**

```bash
kubectl get nodes
```

Deve mostrar os 3 control planes:

```
NAME            STATUS   ROLES           AGE   VERSION
k8s-control01   Ready    control-plane   5m    v1.31.x
k8s-control02   Ready    control-plane   2m    v1.31.x
k8s-control03   Ready    control-plane   1m    v1.31.x
```

### Passo 5: Adicionar os Workers

**No k8s-worker01 e k8s-worker02:**

Use o comando `kubeadm join` **SEM** a flag `--control-plane`:

```bash
sudo kubeadm join 192.168.3.10:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxx
```

**Verifique do control01:**

```bash
kubectl get nodes
```

Deve mostrar todos os nós:

```
NAME            STATUS   ROLES           AGE   VERSION
k8s-control01   Ready    control-plane   10m   v1.31.x
k8s-control02   Ready    control-plane   7m    v1.31.x
k8s-control03   Ready    control-plane   6m    v1.31.x
k8s-worker01    Ready    <none>          2m    v1.31.x
k8s-worker02    Ready    <none>          1m    v1.31.x
```

### Passo 6: Labeling dos Workers (Opcional)

Para melhor organização:

```bash
kubectl label node k8s-worker01 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker02 node-role.kubernetes.io/worker=worker
```

Agora `kubectl get nodes` mostrará:

```
NAME            STATUS   ROLES           AGE   VERSION
k8s-control01   Ready    control-plane   10m   v1.31.x
k8s-control02   Ready    control-plane   7m    v1.31.x
k8s-control03   Ready    control-plane   6m    v1.31.x
k8s-worker01    Ready    worker          2m    v1.31.x
k8s-worker02    Ready    worker          1m    v1.31.x
```

---

## Testes e Validação do Cluster

### Teste 1: Deploy de aplicação teste

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=3
```

Verifique:

```bash
kubectl get deployments
kubectl get pods -o wide
```

Os pods devem estar distribuídos entre os workers.

### Teste 2: Expor como serviço

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```

Verifique a porta atribuída:

```bash
kubectl get svc nginx
```

Output exemplo:

```
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.96.123.45    <none>        80:31234/TCP   5s
```

### Teste 3: Acessar a aplicação

De qualquer VM do cluster ou do pfSense:

```bash
curl http://192.168.3.31:31234  # IP de qualquer worker + NodePort
```

Deve retornar o HTML do Nginx.

### Teste 4: Verificar alta disponibilidade

**Simular falha de um control plane:**

```bash
# No Proxmox, desligue o k8s-control01
# No k8s-control02, execute:
kubectl get nodes
```

Deve continuar funcionando normalmente!

### Teste 5: Verificar failover do Load Balancer

```bash
# No lb-node01, pare o keepalived
sudo systemctl stop keepalived

# Verifique que o VIP migrou para lb-node02
ssh mizael@192.168.3.12
ip a | grep 192.168.3.10

# Do control plane, o cluster deve continuar acessível
kubectl get nodes
```

---

## Instalação do Kubernetes Dashboard (Opcional)

### Passo 1: Deploy do Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### Passo 2: Criar usuário admin

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

### Passo 3: Obter token de acesso

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Guarde o token gerado.

### Passo 4: Acessar o Dashboard

**Criar port-forward:**

```bash
kubectl proxy --address='192.168.3.21' --accept-hosts='.*'
```

**Acessar do seu PC (na rede 192.168.1.x ou 192.168.2.x):**

Primeiro, adicione regra no pfSense:

**Firewall → Rules → LAN (ou LAN_CLIENTES):**

```
Action: Pass
Protocol: TCP
Source: LAN net (sua rede)
Destination: 192.168.3.21
Destination Port: 8001
Description: Acesso ao Dashboard K8s
```

Então acesse:

```
http://192.168.3.21:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Cole o token quando solicitado.

---

## Configuração de Storage (Opcional)

Para persistência de dados, você pode usar:

### Opção 1: Local Path Provisioner (Simples para laboratório)

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Definir como StorageClass padrão
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Opção 2: NFS (Se tiver servidor NFS)

**No servidor NFS (pode ser uma VM na LAN_SERVIDORES):**

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/k8s
sudo chown nobody:nogroup /srv/nfs/k8s
sudo chmod 777 /srv/nfs/k8s

echo "/srv/nfs/k8s 192.168.3.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

**No cluster:**

```bash
# Instalar NFS client nas VMs
# Em TODOS os nodes (control planes e workers):
sudo apt install -y nfs-common

# Instalar NFS Subdir External Provisioner
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.1.x \
    --set nfs.path=/srv/nfs/k8s
```

---

## Monitoramento com Prometheus e Grafana (Opcional)

### Usando Helm:

```bash
# Instalar Helm (no control01)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Adicionar repositório
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Instalar kube-prometheus-stack
kubectl create namespace monitoring
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Verificar
kubectl get pods -n monitoring
```

**Acessar Grafana:**

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address='192.168.3.21'
```

Acesse: `http://192.168.3.21:3000`
- User: `admin`
- Password: `prom-operator`

---

## Comandos Úteis para Administração

### Verificar saúde do cluster

```bash
kubectl cluster-info
kubectl get componentstatuses
kubectl get nodes
kubectl get pods --all-namespaces
```

### Ver certificados

```bash
sudo kubeadm certs check-expiration
```

### Gerar novo token para join (se expirar)

```bash
kubeadm token create --print-join-command
```

### Drenar um nó para manutenção

```bash
kubectl drain k8s-worker01 --ignore-daemonsets --delete-emptydir-data
```

### Reativar nó após manutenção

```bash
kubectl uncordon k8s-worker01
```

### Ver logs de um pod

```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -f  # follow
```

### Executar shell em um pod

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
```

### Ver eventos do cluster

```bash
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

---

## Backup e Restore

### Backup do etcd (Dados do cluster)

```bash
# No control01
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db

# Copiar para local seguro
scp /tmp/etcd-backup-*.db usuario@servidor-backup:/backups/
```

### Restore do etcd

```bash
sudo ETCDCTL_API=3 etcdctl \
  --data-dir=/var/lib/etcd-restore \
  snapshot restore /tmp/etcd-backup-XXXXXXXX.db

# Parar kubelet
sudo systemctl stop kubelet

# Mover dados restaurados
sudo mv /var/lib/etcd /var/lib/etcd.old
sudo mv /var/lib/etcd-restore /var/lib/etcd

# Reiniciar
sudo systemctl start kubelet
```

---

## Troubleshooting Comum

### Problema: Pod em estado "Pending"

**Verificar:**

```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Causas comuns:**
- Recursos insuficientes (CPU/RAM)
- Problemas com PersistentVolumeClaim
- Node selector não encontrado

### Problema: Pod em "CrashLoopBackOff"

**Ver logs:**

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

**Causas comuns:**
- Erro na aplicação
- ConfigMap/Secret ausente
- Problema de permissão

### Problema: Nodes em "NotReady"

**Verificar kubelet:**

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

**Causas comuns:**
- CNI não instalado/quebrado
- containerd parado
- Problemas de rede

### Problema: CoreDNS pods em "Pending"

**Verificar CNI:**

```bash
kubectl get pods -n kube-system | grep flannel
```

Se Flannel não estiver rodando, reinstale:

```bash
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

## Melhores Práticas

### Segurança

1. **Manter sistema atualizado:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Atualizar Kubernetes regularmente:**
   ```bash
   sudo kubeadm upgrade plan
   sudo kubeadm upgrade apply v1.31.x
   ```

3. **Usar RBAC (Role-Based Access Control):**
   - Criar service accounts específicas
   - Nunca usar cluster-admin em produção

4. **Network Policies:**
   - Implementar segmentação dentro do cluster
   - Controlar tráfego entre namespaces

5. **Secrets management:**
   - Usar External Secrets Operator
   - Ou HashiCorp Vault

### Monitoramento

1. **Logs centralizados:**
   - ELK Stack (Elasticsearch, Logstash, Kibana)
   - Ou Loki + Grafana

2. **Métricas:**
   - Prometheus + Grafana
   - Alertmanager para notificações

3. **Alertas configurados para:**
   - Uso de CPU/RAM > 80%
   - Disco > 85%
   - Pods em CrashLoop
   - Nodes NotReady

### Backup

1. **Backup regular do etcd** (diário)
2. **Backup das configurações** (manifests YAML)
3. **Testar restore periodicamente**

---

## Recursos Adicionais

### Documentação Oficial

- **Kubernetes:** https://kubernetes.io/docs/
- **kubeadm:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- **Flannel:** https://github.com/flannel-io/flannel
- **HAProxy:** https://www.haproxy.org/
- **Keepalived:** https://www.keepalived.org/

### Ferramentas Úteis

- **k9s:** Terminal UI para Kubernetes
  ```bash
  wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
  tar -xzf k9s_Linux_amd64.tar.gz
  sudo mv k9s /usr/local/bin/
  ```

- **kubectx/kubens:** Alternar entre contextos e namespaces
  ```bash
  sudo apt install kubectx
  ```

- **Helm:** Gerenciador de pacotes do Kubernetes
  ```bash
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  ```

### Cursos e Certificações

- **CKA (Certified Kubernetes Administrator)**
- **CKAD (Certified Kubernetes Application Developer)**
- **CKS (Certified Kubernetes Security Specialist)**

---

## Próximas Evoluções Sugeridas

1. **Implementar Ingress Controller** (Nginx/Traefik)
2. **Service Mesh** (Istio/Linkerd)
3. **GitOps** (ArgoCD/Flux)
4. **CI/CD** (Jenkins/GitLab CI integrado)
5. **Observabilidade completa** (Jaeger para tracing)
6. **Multi-cluster** (Rancher/Tanzu)

---

## Conclusão

Você agora tem um **cluster Kubernetes de alta disponibilidade** rodando em infraestrutura virtualizada local com:

- Segmentação de rede via pfSense
- Load balancing com HAProxy
- Failover automático com Keepalived
- 3 control planes para HA
- 2 workers para workload
- Rede de pods via Flannel
- Pronto para produção em ambiente de laboratório

Este ambiente serve como base sólida para estudos de:
- Arquitetura de microserviços
- Orquestração de containers
- Automação e DevOps
- Segurança de infraestrutura
- Preparação para certificações Kubernetes

**Bons estudos e sucesso na sua jornada DevOps!**