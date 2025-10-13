# Guia de Configurações Adicionais do Kubernetes

Este arquivo centraliza configurações detalhadas e explicações aprofundadas que complementam o guia principal de instalação do cluster (`setup-kubernetes.md`).

## 📋 Sumário
1. [Entendendo Módulos do Kernel e Rede no Kubernetes](#módulos-do-kernel-e-rede-no-kubernetes)
2. [Configuração de NAT no pfSense para a Rede Kubernetes](#configuração-de-nat-no-pfsense-para-kubernetes)
3. [Configuração HAProxy para Kubernetes Dashboard](#configuração-haproxy-para-kubernetes-dashboard)

---

## 🧠 Entendendo Módulos do Kernel e Rede no Kubernetes {#módulos-do-kernel-e-rede-no-kubernetes}

### 📚 Índice
- [Visão Geral](#visão-geral)
- [Módulo overlay](#módulo-overlay)
- [Módulo br_netfilter](#módulo-br_netfilter)
- [Parâmetros sysctl](#parâmetros-sysctl)
- [Onde Aplicar as Configurações](#onde-aplicar-as-configurações)
- [Configuração de /etc/hosts](#configuração-de-etchosts)
- [Verificação e Troubleshooting](#verificação-e-troubleshooting)

---

### 🎯 Visão Geral

Este documento explica **por que** os módulos do kernel `overlay` e `br_netfilter` são necessários para o funcionamento do Kubernetes, e **onde** cada configuração deve ser aplicada na sua infraestrutura.

#### Arquitetura da Infraestrutura

```
┌─────────────┐
│   PfSense   │ (192.168.3.1)
└──────┬──────┘
       │
       ├─── VIP: 192.168.3.10 (Keepalived) 
       │         ↓
       │    ┌────────────────────────┐
       │    │ HAProxy + Keepalived   │
       │    │ - LB-01: 192.168.3.11  │
       │    │ - LB-02: 192.168.3.12  │
       │    └────────┬───────────────┘
       │             │
       │    ┌────────┴─────────────────────┐
       │    │                              │
       │    │      Control Planes          │
       │    │  - CP-01: 192.168.3.21       │
       │    │  - CP-02: 192.168.3.22       │
       │    │  - CP-03: 192.168.3.23       │
       │    └──────────────────────────────┘
       │             │
       │    ┌────────┴───────────┐
       │    │                    │
       │    │      Workers       │
       │    │  - W-01: 192.168.3.31  │
       │    │  - W-02: 192.168.3.32  │
       └────┴────────────────────┴────
```

---

### 📦 Módulo `overlay`

#### O que é?

O `overlay` é um driver de filesystem que permite criar camadas (layers) de sistemas de arquivos sobrepostos.

#### Por que é necessário?

**Containers usam imagens em camadas:**

```
┌──────────────────────────────────────┐
│  Container em execução               │  ← Camada READ/WRITE
│  (camada gravável - ephemeral)       │
├──────────────────────────────────────┤
│  Suas configurações personalizadas   │  ← Camada READ-ONLY
│  (ex: nginx.conf, scripts)           │
├──────────────────────────────────────┤
│  Instalação do Nginx                 │  ← Camada READ-ONLY
│  (binários, bibliotecas)             │
├──────────────────────────────────────┤
│  Imagem base Ubuntu                  │  ← Camada READ-ONLY
│  (sistema operacional mínimo)        │
└──────────────────────────────────────┘
```

**Como funciona:**

1. **Eficiência de armazenamento**: Múltiplos containers podem compartilhar as mesmas camadas base
2. **Rapidez**: Não precisa copiar toda a imagem para criar um container
3. **Isolamento**: Cada container tem sua própria camada gravável

#### Exemplo Prático

```bash
# Sem overlay - NÃO FUNCIONA
sudo modprobe -r overlay
sudo docker run nginx
# Erro: "overlay2 not found"

# Com overlay - FUNCIONA
sudo modprobe overlay
sudo docker run nginx
# Container inicia normalmente ✅
```

#### Alternativas (não recomendadas)

- `aufs` - Antigo, descontinuado
- `devicemapper` - Lento, problemas de performance
- `btrfs` - Requer filesystem btrfs específico

---

### 🌉 Módulo `br_netfilter`

#### O que é?

Permite que o **iptables** (firewall do Linux) processe tráfego que passa por **bridges de rede** (network bridges).

#### Por que é CRÍTICO para Kubernetes?

##### Arquitetura de Rede do Kubernetes

```
┌─────────────────────────────────────────────────┐
│                   NODE 1                        │
│                                                 │
│  ┌──────────┐       ┌──────────┐              │
│  │  Pod A   │       │  Pod B   │              │
│  │10.244.1.5│       │10.244.1.6│              │
│  └────┬─────┘       └────┬─────┘              │
│       │                  │                     │
│       └──────┬───────────┘                     │
│              │                                 │
│         ┌────▼────┐                            │
│         │  cni0   │  ← Bridge Virtual         │
│         │ bridge  │                            │
│         └────┬────┘                            │
│              │                                 │
│         ┌────▼────┐                            │
│         │  eth0   │  ← Interface Física       │
│         └────┬────┘                            │
└──────────────┼──────────────────────────────────┘
               │
               ▼
        Rede Física → Outros Nodes
```

##### O Problema SEM `br_netfilter`

Quando um pacote passa pela bridge (`cni0`), ele **ignora** o iptables e vai direto para a rede física.

**Consequências:**

1. ❌ **Services não funcionam** - O kube-proxy não consegue fazer load balancing
2. ❌ **Network Policies são ignoradas** - Regras de firewall entre pods falham
3. ❌ **NAT não funciona** - Comunicação entre nodes quebra
4. ❌ **ClusterIP não resolve** - Requisições para Services falham

##### Fluxo de uma requisição HTTP para um Service

**COM `br_netfilter` habilitado:**

```
1. Cliente faz request para: http://my-service.default.svc.cluster.local
   (DNS resolve para ClusterIP: 10.96.0.10)

2. Pacote chega na bridge cni0 do node
   │
   ▼
3. br_netfilter PERMITE que iptables intercepte o pacote ✅
   │
   ▼
4. iptables (regras do kube-proxy) fazem DNAT:
   10.96.0.10 → 10.244.1.5 (IP real do Pod)
   │
   ▼
5. Pacote é roteado através do overlay network para o node correto
   │
   ▼
6. Chega no Pod destino ✅
```

**SEM `br_netfilter`:**

```
1. Cliente faz request para: http://my-service.default.svc.cluster.local
   (DNS resolve para ClusterIP: 10.96.0.10)

2. Pacote chega na bridge cni0 do node
   │
   ▼
3. Pacote PASSA DIRETO pela bridge, ignorando iptables ❌
   │
   ▼
4. Pacote sai pela eth0 com destino 10.96.0.10
   │
   ▼
5. ❌ FALHA - IP 10.96.0.10 não existe fisicamente na rede
```

#### Como o kube-proxy usa iptables

O kube-proxy cria regras como estas:

```bash
# Ver regras criadas pelo kube-proxy
sudo iptables -t nat -L -n | grep KUBE

# Exemplo de regra para um Service:
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m tcp --dport 80 \
   -j KUBE-SVC-XXXXXXXXXXX

# Esta regra faz DNAT (Destination NAT):
# 10.96.0.10:80 → 10.244.1.5:8080 (Pod real)
```

**Sem `br_netfilter`, essas regras são ignoradas para tráfego bridged!**

---

### ⚙️ Parâmetros sysctl

#### `net.bridge.bridge-nf-call-iptables = 1`

**O que faz:**
- Ativa o processamento de pacotes IPv4 que passam por bridges através do iptables

**Por que é necessário:**
- Mesmo com o módulo `br_netfilter` carregado, o kernel precisa ser **explicitamente instruído** a chamar o iptables
- `= 1` significa "sim, processe tráfego de bridge pelo iptables"
- `= 0` significaria "ignore iptables para tráfego de bridge"

#### `net.bridge.bridge-nf-call-ip6tables = 1`

**O que faz:**
- Mesma função que acima, mas para IPv6

**Por que é necessário:**
- Kubernetes suporta dual-stack (IPv4 + IPv6)
- Garante que regras de firewall funcionem para ambos os protocolos

#### `net.ipv4.ip_forward = 1`

**O que faz:**
- Permite que o Linux **roteie pacotes** entre interfaces de rede

**Por que é necessário:**
- Transforma o node em um mini-roteador
- Permite que pacotes destinados a outros nodes sejam **encaminhados** ao invés de **descartados**
- Essencial para comunicação Pod-to-Pod entre nodes diferentes

**Exemplo:**

```bash
# SEM ip_forward (= 0)
Pod-A (Node1) → tenta enviar pacote → Pod-B (Node2)
❌ Node1 descarta o pacote: "não é para mim e não posso rotear"

# COM ip_forward (= 1)
Pod-A (Node1) → tenta enviar pacote → Pod-B (Node2)
✅ Node1 encaminha o pacote: "não é para mim, mas vou rotear para Node2"
```

---

### 🎯 Onde Aplicar as Configurações

#### Tabela de Componentes

| Componente | Módulos Kernel | Parâmetros sysctl | kubelet | /etc/hosts |
|------------|----------------|-------------------|---------|------------|
| **Control Planes (3x)** | ✅ Sim | ✅ Sim | ✅ Sim | ✅ Sim |
| **Workers (2x)** | ✅ Sim | ✅ Sim | ✅ Sim | ✅ Sim |
| **Load Balancers (2x)** | ❌ Não | ❌ Não | ❌ Não | 🟡 Opcional |
| **PfSense** | ❌ Não | ❌ Não | ❌ Não | 🟡 Opcional |

#### Por que NÃO nos Load Balancers?

**Load Balancers (HAProxy + Keepalived) não fazem parte do cluster Kubernetes:**

- Não executam Pods
- Não precisam de container runtime
- Não precisam de módulos overlay/br_netfilter
- Apenas balanceiam tráfego TCP na camada 4 (L4)

**O que eles precisam:**

```bash
# Apenas HAProxy e Keepalived
sudo apt install -y haproxy keepalived

# Configurar HAProxy para balancear API Server (porta 6443)
# Configurar Keepalived para fornecer VIP (alta disponibilidade)
```

---

### 📝 Configuração de /etc/hosts

#### ❌ Erro Comum

```bash
# ERRADO - nomes inconsistentes com hostname real
192.168.3.21    k8s-master01
192.168.3.22    k8s-control-plane-02
192.168.3.31    worker-node-1
```

**Problema:** O Kubernetes usa o **hostname real** da VM para identificar nodes. Se houver inconsistência, você terá:
- Nodes com nomes duplicados
- Certificados TLS inválidos
- Dificuldade para identificar nodes no `kubectl get nodes`

#### ✅ Configuração Correta

**Regra de ouro:** O primeiro nome após o IP deve ser o **hostname real** da VM (retornado pelo comando `hostname`)

```bash
# Verificar hostname em cada VM
hostname
# Saída esperada: control-plane-01

hostnamectl
# Status completo do hostname
```

#### Modelo para todas as VMs

```bash
sudo nano /etc/hosts
```

```bash
# Loopback
127.0.0.1       localhost
127.0.1.1       control-plane-01    # ← AJUSTAR em cada VM (hostname local)

# Infraestrutura
192.168.3.1     pfsense
192.168.3.10    vip-k8s-api         # VIP do Keepalived

# Load Balancers
192.168.3.11    lb-node01
192.168.3.12    lb-node02

# Control Planes (usar hostname REAL de cada VM)
192.168.3.21    control-plane-01
192.168.3.22    control-plane-02
192.168.3.23    control-plane-03

# Workers (usar hostname REAL de cada VM)
192.168.3.31    worker-01
192.168.3.32    worker-02
```

#### Checklist de Consistência

Execute em **cada VM**:

```bash
# 1. Verificar hostname atual
hostname

# 2. Verificar /etc/hostname
cat /etc/hostname

# 3. Se necessário, ajustar hostname
sudo hostnamectl set-hostname control-plane-01

# 4. Reiniciar para garantir
sudo reboot

# 5. Após reboot, verificar novamente
hostname
hostnamectl

# 6. Editar /etc/hosts conforme modelo acima
sudo nano /etc/hosts
```

#### Importância para Kubernetes

Quando você executa `kubectl get nodes`, o resultado será:

```bash
NAME                STATUS   ROLE           AGE     VERSION
control-plane-01    Ready    control-plane  5m      v1.28.0
control-plane-02    Ready    control-plane  4m      v1.28.0
control-plane-03    Ready    control-plane  3m      v1.28.0
worker-01           Ready    <none>         2m      v1.28.0
worker-02           Ready    <none>         1m      v1.28.0
```

Os nomes que aparecem aqui são os **hostnames** das VMs, não os nomes do `/etc/hosts`. Por isso a consistência é crítica!

---

### 🔍 Verificação e Troubleshooting

#### Verificar módulos carregados

```bash
# Verificar se módulos estão carregados
lsmod | grep overlay
lsmod | grep br_netfilter

# Saída esperada:
# overlay    212992  1
# br_netfilter  32768  0
# bridge    421888  1 br_netfilter
```

#### Verificar parâmetros sysctl

```bash
# Verificar cada parâmetro
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward

# Saída esperada (todos = 1):
# net.bridge.bridge-nf-call-iptables = 1
# net.bridge.bridge-nf-call-ip6tables = 1
# net.ipv4.ip_forward = 1
```

#### Verificar configuração completa

```bash
# Mostrar tudo de uma vez
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

#### Testar conectividade

```bash
# Ping entre nodes usando hostname
ping control-plane-01
ping worker-01

# Testar resolução de nomes
getent hosts control-plane-01

# Verificar se VIP está acessível
ping vip-k8s-api
curl -k https://vip-k8s-api:6443
```

#### Troubleshooting comum

##### Problema: Services não funcionam

```bash
# Verificar se br_netfilter está ativo
lsmod | grep br_netfilter

# Verificar parâmetros
sysctl net.bridge.bridge-nf-call-iptables
```

##### Problema: Pods não conseguem se comunicar entre nodes

```bash
# Verificar ip_forward
sysctl net.ipv4.ip_forward

# Deve retornar: net.ipv4.ip_forward = 1
```

##### Problema: Container runtime não inicia

```bash
# Verificar overlay
lsmod | grep overlay

# Verificar logs do containerd
sudo journalctl -u containerd -n 50
```

---

### 📚 Referências

- [Documentação Oficial Kubernetes - Prerequisites](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Linux Bridge Documentation](https://wiki.linuxfoundation.org/networking/bridge)
- [OverlayFS Documentation](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)
- [Netfilter Documentation](https://www.netfilter.org/documentation/)

---
---

## 🔥 Configuração de NAT no pfSense para Kubernetes {#configuração-de-nat-no-pfsense-para-kubernetes}

### Problema Comum

Após criar a interface LAN_K8S e configurar as regras de firewall, as VMs podem ter conectividade local (ping para o gateway), mas não conseguem acessar a internet. Isso ocorre porque **falta configuração de NAT (Network Address Translation)** para a nova rede.

### Sintomas

- ✅ Ping para o gateway (192.168.3.1) funciona
- ✅ Conectividade entre VMs da mesma rede funciona
- ❌ Ping para IPs externos (8.8.8.8, 1.1.1.1) falha
- ❌ Resolução de DNS externa não funciona
- ❌ Downloads e atualizações (apt update) falham

### Verificação do Problema

#### No pfSense, acesse:

**Firewall → NAT → Outbound**

Verifique o modo atual:
- Se estiver em **"Automatic outbound NAT rule generation"**, o problema não é NAT
- Se estiver em **"Manual Outbound NAT rule generation"** ou **"Hybrid Outbound NAT rule generation"**, verifique se existe regra para a rede 192.168.3.0/24

---


### Solução 1: Modo Automático (Recomendado para Laboratório)

O modo automático cria regras de NAT automaticamente para todas as interfaces configuradas.

#### Passos:

1. Acesse: **Firewall → NAT → Outbound**

2. Selecione: **⦿ Automatic outbound NAT rule generation**

3. Clique em **Save**

4. Clique em **Apply Changes**

5. Verifique as regras criadas automaticamente na seção "Automatic Rules":
   ```
   ✅ WAN | 192.168.1.0/24 → * | * | WAN address
   ✅ WAN | 192.168.2.0/24 → * | * | WAN address
   ✅ WAN | 192.168.3.0/24 → * | * | WAN address
   ```

#### Testar na VM:

```bash
ping -c 3 8.8.8.8
ping -c 3 google.com
curl -I https://google.com
```

---


### Solução 2: Modo Manual (Para Controle Granular)

Use este modo se precisar de controle fino sobre quais redes podem ter NAT ou se quiser configurações específicas.

#### Passos:

1. Acesse: **Firewall → NAT → Outbound**

2. Se ainda não estiver em modo manual, mude para:
   **⦿ Manual Outbound NAT rule generation**

3. Clique em **Save**

4. As regras existentes serão convertidas. Agora adicione a regra para LAN_K8S:

#### Adicionar Regra NAT para LAN_K8S:

1. Clique em **Add** ↑ (adicionar no topo)

2. Configure:

**Interface:**
```
Interface: WAN
```

**Protocol:**
```
Address Family: IPv4
Protocol: Any
```

**Source:**
```
Source: Network
Network: 192.168.3.0 / 24
```

**Destination:**
```
Destination: Any
```

**Translation:**
```
Translation / Target: Interface Address
```

**Misc:**
```
Description: NAT LAN_K8S (Kubernetes) para WAN
Static Port: [ ] (desmarcado)
```

3. **Save**

4. **Apply Changes**

#### Ordem das Regras NAT:

Após criar, suas regras devem ficar assim:

```
1. ✅ WAN | 192.168.3.0/24 → * | * | WAN address | NAT LAN_K8S
2. ✅ WAN | 192.168.1.0/24 → * | * | WAN address | NAT LAN Servidores
3. ✅ WAN | 192.168.2.0/24 → * | * | WAN address | NAT LAN Clientes
4. ✅ WAN | 127.0.0.0/8 → * | 500 (ISAKMP) | WAN address | (Regra ISAKMP)
5. ✅ WAN | 127.0.0.0/8 → * | * | WAN address | (localhost)
```

---


### Solução 3: Modo Híbrido (Automático + Manual)

Este modo permite regras automáticas E regras manuais adicionais.

#### Passos:

1. **Firewall → NAT → Outbound**

2. Selecione: **⦿ Hybrid Outbound NAT rule generation**

3. **Save → Apply Changes**

4. Adicione regras manuais específicas se necessário (mesmo processo da Solução 2)

---


### Validação e Testes

#### Teste 1: Resetar estados de conexão

Após configurar o NAT, limpe as conexões antigas:

1. **Diagnostics → States → Reset States**
2. Marque: **Reset all states**
3. Clique em **Reset**

#### Teste 2: Verificar conectividade na VM

```bash
# Teste básico de rede
ping -c 3 192.168.3.1    # Gateway
ping -c 3 8.8.8.8        # Internet (Google DNS)
ping -c 3 1.1.1.1        # Internet (Cloudflare DNS)

# Teste de DNS
nslookup google.com
dig google.com

# Teste HTTP/HTTPS
curl -I http://google.com
curl -I https://google.com

# Teste de download
wget -O /dev/null http://speedtest.tele2.net/1MB.zip
```

#### Teste 3: Verificar IP público visto pela VM

```bash
curl ifconfig.me
```

Deve retornar o **IP público da sua WAN** (o mesmo que o pfSense usa para sair à internet).

#### Teste 4: Verificar logs de NAT (opcional)

1. **Status → System Logs → Firewall**
2. Aba: **NAT**
3. Durante os testes, você verá as traduções de endereço acontecendo

---


### Troubleshooting

#### Problema: NAT configurado mas internet não funciona

**Possíveis causas:**

1. **Regras de firewall bloqueando:**
   - Verifique **Firewall → Rules → LAN_K8S**
   - Deve ter regra **Pass** permitindo tráfego de `LAN_K8S net` para `Any`

2. **Gateway WAN offline:**
   - Verifique **System → Routing → Gateways**
   - Gateway WAN deve estar **Online** (bolinha verde)

3. **DNS não configurado:**
   - Na VM: `cat /etc/resolv.conf`
   - Deve ter: `nameserver 192.168.3.1`

4. **Rota padrão ausente na VM:**
   ```bash
ip route show
   # Deve ter: default via 192.168.3.1
   ```

#### Problema: Apenas algumas VMs funcionam

- Verifique se todas as VMs estão na rede correta (192.168.3.0/24)
- Verifique se todas têm o gateway configurado (192.168.3.1)
- Execute `sudo netplan apply` em cada VM

#### Problema: Internet lenta após NAT

- **Provável causa:** Fragmentação de pacotes
- **Solução:** Ajustar MSS Clamping

**Firewall → Rules → LAN_K8S → Regra de saída → Advanced:**
```
Tag: (vazio)
Max state entries: (deixe padrão)
Max new connections: (deixe padrão)
State Type: Keep state
No XMLRPC Sync: [ ]
Schedule: (none)
Gateway: default (ou seu gateway WAN)
Advanced Options:
  Max MSS: 1420 (ajuste se necessário)
```

---


### Exemplo Completo de Configuração

#### Resumo das configurações necessárias:

**1. Interface LAN_K8S configurada:**
- IP: 192.168.3.1/24
- Interface ativa

**2. Regras de Firewall (LAN_K8S):**
```
✅ Pass | TCP/UDP | LAN_K8S net → LAN_K8S address | Port 53 (DNS)
✅ Pass | Any | LAN_K8S net → LAN_K8S net (interno)
✅ Pass | TCP | LAN_K8S net → Any | Port 80,443 (web)
✅ Pass | UDP | LAN_K8S net → Any | Port 123 (NTP)
✅ Pass | ICMP | LAN_K8S net → Any (ping)
```

**3. NAT Outbound:**
```
✅ Modo: Automatic (ou Manual com regra para 192.168.3.0/24)
✅ Regra: WAN | 192.168.3.0/24 → * | WAN address
```

**4. Routing:**
```
✅ Gateway WAN: Online
✅ Rota padrão configurada no pfSense
```

---


### Checklist de Verificação

Após configurar o NAT:

- [ ] NAT configurado (Automatic ou Manual com regra para 192.168.3.0/24)
- [ ] Estados de conexão resetados
- [ ] VM consegue pingar 192.168.3.1 (gateway)
- [ ] VM consegue pingar 8.8.8.8 (internet)
- [ ] VM consegue resolver DNS (nslookup google.com)
- [ ] VM consegue baixar da internet (curl/wget funcionando)
- [ ] IP público visto pela VM é o mesmo da WAN do pfSense

---


### Boas Práticas

1. **Use modo Automatic** para laboratório/estudo (mais simples)
2. **Use modo Manual** para produção (mais controle)
3. **Documente** qualquer regra manual criada
4. **Teste após cada mudança** antes de prosseguir
5. **Faça backup** da configuração do pfSense após funcionar:
   - **Diagnostics → Backup & Restore → Backup Configuration**

---


### Referências

- **pfSense NAT Documentation:** https://docs.netgate.com/pfsense/en/latest/nat/
- **Outbound NAT:** https://docs.netgate.com/pfsense/en/latest/nat/outbound.html

---
---

## 🚀 Configuração HAProxy para Kubernetes Dashboard {#configuração-haproxy-para-kubernetes-dashboard}

# Adicione ao final de /etc/haproxy/haproxy.cfg

#---------------------------------------------------------------------
# Frontend Dashboard Kubernetes (porta 8443)
#---------------------------------------------------------------------
frontend dashboard-frontend
    bind *:8443
    mode tcp
    option tcplog
    default_backend dashboard-backend

#---------------------------------------------------------------------
# Backend Dashboard com todos os nodes
#---------------------------------------------------------------------
backend dashboard-backend
    mode tcp
    option tcp-check
    balance roundrobin
    
    # Control Planes
    server cp01 192.168.3.21:30443 check fall 3 rise 2
    server cp02 192.168.3.22:30443 check fall 3 rise 2
    server cp03 192.168.3.23:30443 check fall 3 rise 2
    
    # Workers
    server w01  192.168.3.31:30443 check fall 3 rise 2
    server w02  192.168.3.32:30443 check fall 3 rise 2

#---------------------------------------------------------------------
# Opcional: Estatísticas do HAProxy
#---------------------------------------------------------------------
listen stats
    bind *:9000
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if TRUE
    stats auth admin:password  # Mude a senha!

