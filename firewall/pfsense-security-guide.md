# Guia Completo: Implementação de Firewall pfSense Seguro + IDS/IPS

## 📋 Sumário
1. [Arquitetura da Rede](#arquitetura)
2. [Políticas de Segurança](#politicas)
3. [Criação de Aliases](#aliases)
4. [Regras de Firewall - LAN (Servidores)](#regras-lan)
5. [Regras de Firewall - LAN_DESKTOPS (Clientes)](#regras-desktops)
6. [Regras de Firewall - WAN](#regras-wan)
7. [Regras de Firewall - LAN_K8S (Kubernetes)](#regras-k8s)
8. [Regras de NAT (Network Address Translation)](#regras-nat)
9. [Configurações de VPN](#regras-vpn)
10. [Testes e Validação](#testes)
11. [Monitoramento e Logs](#monitoramento)
12. [Manutenção e Boas Práticas](#manutencao)

---

## 🏗️ Arquitetura da Rede {#arquitetura}

### Topologia Implementada:
```
Internet (Provedor)
    ↓
192.168.0.0/24 (vmbr0 - WAN)
    ↓
[pfSense Firewall]
    ├─→ 192.168.1.0/24 (vmbr1 - LAN Servidores)
    └─→ 192.168.2.0/24 (vmbr2 - LAN Clientes)
    └─→ 192.168.3.0/24 (vmbr3 - LAN Kubernetes)
```

### Endereçamento IP:
| Interface | Descrição | Bridge | IP do pfSense | Rede | IPv6 | Função |
|-----------|-------------|--------|---------------|----------------|------|----------|
| WAN | WAN | vmbr0 | 192.168.0.100 | 192.168.0.0/24 | - | Internet |
| LAN | LAN | vmbr1 | 192.168.1.1 | 192.168.1.0/24 | Track | Servidores |
| OPT1 | LAN_DESKTOPS| vmbr2 | 192.168.2.1 | 192.168.2.0/24 | - | Clientes |
| OPT2 | LAN_K8S | vmbr3 | 192.168.3.1 | 192.168.3.0/24 | - | Kubernetes |

> **Nota Importante:** Para que o restante do guia funcione corretamente, renomeie as interfaces no pfSense. Navegue até **Interfaces → Assignments**, clique na interface **OPT1** e mude seu nome para `LAN_DESKTOPS`. Repita o processo para **OPT2**, renomeando-a para `LAN_K8S`.

---

## 🛡️ Políticas de Segurança {#politicas}

### Princípios Aplicados:
- **Menor Privilégio**: Liberar apenas o necessário
- **Segmentação**: Isolar servidores de clientes
- **Defesa em Profundidade**: Múltiplas camadas de segurança
- **Log e Auditoria**: Registrar tentativas de acesso

### Matriz de Acesso:

| Origem → Destino | Internet | pfSense | Servidores (LAN) | Clientes (Desktops) |
|------------------|----------|---------|------------------|---------------------|
| **Servidores** | HTTP/HTTPS, DNS, NTP, ICMP | DNS, WebGUI | SSH, RDP | ❌ Bloqueado |
| **Clientes** | HTTP/HTTPS, DNS, ICMP | DNS, WebGUI | HTTP/HTTPS, SSH, RDP | Entre si OK |
| **Internet** | - | Apenas admin (192.168.0.0/24) | ❌ Bloqueado | ❌ Bloqueado |

---

## 🏷️ Criação de Aliases {#aliases}

Aliases facilitam a gestão e manutenção das regras.

### Passo 1: Acessar Aliases
1. Faça login no pfSense
2. Navegue: **Firewall → Aliases**
3. Clique na aba **IP**

### Passo 2: Criar Alias - Rede de Servidores

1. Clique em **Add** (botão com ↑)
2. Preencha:
   ```
   Name: LAN_SERVIDORES_NET
   Description: Rede dos servidores (vmbr1)
   Type: Network(s)
   ```
3. Em **Network or FQDN**:
   ```
   Network: 192.168.1.0/24
   ```
4. **Save**

### Passo 3: Criar Alias - Rede de Clientes

1. **Add** novamente
2. Preencha:
   ```
   Name: LAN_DESKTOPS_NET
   Description: Rede dos clientes/desktops (vmbr2)
   Type: Network(s)
   Network: 192.168.2.0/24
   ```
3. **Save**

### Passo 4: Criar Alias - Rede Kubernetes

1. **Add** novamente
2. Preencha:
   ```
   Name: LAN_K8S_NET
   Description: Rede do Kubernetes (vmbr3)
   Type: Network(s)
   Network: 192.168.3.0/24
   ```
3. **Save**

### Passo 5: Criar Alias - Portas Web

1. Clique na aba **Ports**
2. **Add**
3. Preencha:
   ```
   Name: Portas_Web
   Description: Portas HTTP e HTTPS
   Type: Port(s)
   Port: 80
   ```
4. Clique em **Add Port** e adicione:
   ```
   Port: 443
   ```
5. **Save**

### Passo 6: Criar Alias - Portas de Administração

1. Na aba **Ports**, clique em **Add**
2. Preencha:
   ```
   Name: Portas_Admin
   Description: SSH e RDP
   Type: Port(s)
   Port: 22
   ```
3. **Add Port** e adicione:
   ```
   Port: 3389
   ```
4. **Save**

### Passo 7: Criar Alias - Serviços Permitidos

1. **Add** em Ports
2. Preencha:
   ```
   Name: Portas_Servicos_LAN
   Description: Serviços permitidos nos servidores
   Type: Port(s)
   ```
3. Adicione as portas:
   ```
   80    (HTTP)
   443   (HTTPS)
   22    (SSH)
   3389  (RDP)
   ```
4. **Save**



### Passo 8: Criar Alias - API Kubernetes

1. Na aba **Ports**, clique em **Add**
2. Preencha:
   ```
   Name: API_K8S
   Description: Porta da API do Kubernetes
   Type: Port(s)
   Port: 6443
   ```
3. **Save**

### Passo 9: Criar Alias - IP do TrueNAS

1. Na aba **IP**, clique em **Add**
2. Preencha:
   ```
   Name: IP_TrueNas
   Description: IP do servidor TrueNAS
   Type: Host(s)
   IP or FQDN: 192.168.1.114
   ```
3. **Save**

### Passo 10: Criar Alias - IP do PC Admin

1. Na aba **IP**, clique em **Add**
2. Preencha:
   ```
   Name: IP_Admin_PC
   Description: IP do PC de administração na WAN
   Type: Host(s)
   IP or FQDN: 192.168.0.10
   ```
3. **Save**

### Passo 11: Aplicar Mudanças
- Clique em **Apply Changes** no topo da página

### ✅ Aliases Criados:
- ✅ LAN_SERVIDORES_NET (192.168.1.0/24)
- ✅ LAN_DESKTOPS_NET (192.168.2.0/24)
- ✅ LAN_K8S_NET (192.168.3.0/24)
- ✅ IP_TrueNas (192.168.1.114)
- ✅ IP_Admin_PC (192.168.0.10)
- ✅ Portas_Web (80, 443)
- ✅ Portas_Admin (22, 3389)
- ✅ Portas_Servicos_LAN (80, 443, 22, 3389)
- ✅ API_K8S (6443)

---

## 🖥️ Regras de Firewall - LAN (Servidores) {#regras-lan}

### Preparação:
1. Navegue: **Firewall → Rules → LAN**
2. **EXCLUA** todas as regras permissivas existentes (exceto Anti-Lockout)
3. Vamos criar as regras na ordem correta

---

### Regra 1: Anti-Lockout (Já existe - NÃO modificar)
```
✅ Já está criada pelo pfSense
Descrição: Regra Anti-Lockout
Ação: Pass
Portas: 443, 80
```
**⚠️ NUNCA exclua esta regra!**

---

### Regra 2: Permitir acesso ao pfSense (DNS e WebGUI)

1. Clique em **Add** ↑ (adicionar no topo, abaixo da Anti-Lockout)
2. Preencha:

**Edit Firewall Rule:**
```
Action: Pass
Disabled: [ ] (desmarcado)
Interface: LAN
Address Family: IPv4
Protocol: TCP/UDP
```

**Source:**
```
Source: LAN net
```

**Destination:**
```
Destination: LAN address
Destination Port Range:
  From: (other) Custom: 53
  To: (other) Custom: 53
```

3. Clique em **Display Advanced** (no final da página)
4. Role até **Extra Options**:
```
Description: LAN → pfSense: DNS
```

5. **Save**

6. Repita o processo para HTTPS/HTTP:

**Segunda regra para WebGUI:**
```
Action: Pass
Interface: LAN
Protocol: TCP
Source: LAN net
Destination: LAN address
Destination Port Range:
  From: HTTPS (443)
  To: HTTPS (443)
Description: LAN → pfSense: WebGUI HTTPS
```

**Save**

---

### Regra 3: Permitir DNS para Internet

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN
Address Family: IPv4
Protocol: TCP/UDP

Source: LAN net
Destination: Any
Destination Port Range:
  From: DNS (53)
  To: DNS (53)

Description: LAN → Internet: Consultas DNS
```
3. **Save**

---

### Regra 4: Permitir HTTP/HTTPS para Internet

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN
Protocol: TCP

Source: LAN net
Destination: Any
Destination Port Range:
  From: (other) Custom: Portas_Web
  To: (other) Custom: Portas_Web

Description: LAN → Internet: Navegação web
```
3. **Save**

---

### Regra 5: Permitir NTP (Sincronização de tempo)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN
Protocol: UDP

Source: LAN net
Destination: Any
Destination Port Range:
  From: NTP (123)
  To: NTP (123)

Description: LAN → Internet: Sincronização NTP
```
3. **Save**

---

### Regra 6: Permitir ICMP (Ping)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN
Address Family: IPv4
Protocol: ICMP
ICMP Type: Any

Source: LAN net
Destination: Any

Description: LAN → Any: ICMP para diagnóstico
```
3. **Save**

---

### Regra 7: Permitir comunicação entre servidores (SSH/RDP)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN
Protocol: TCP

Source: LAN net
Destination: LAN net
Destination Port Range:
  From: (other) Custom: Portas_Admin
  To: (other) Custom: Portas_Admin

Description: LAN → LAN: SSH e RDP entre servidores
```
3. **Save**

---

### Regra 8: Permitir acesso à API do Kubernetes

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN
Protocol: TCP

Source: LAN_SERVIDORES_NET
Destination: Single host or alias: LAN_K8S_NET
Destination Port Range:
  From: (other) Custom: API_K8S
  To: (other) Custom: API_K8S

Description: LAN → K8S: Acesso à API do Kubernetes
```
3. **Save**

---

### Regra 9: BLOQUEAR acesso à rede de clientes

1. **Add** ↑
2. Preencha:
```
Action: Block
Interface: LAN
Protocol: Any

Source: LAN net
Destination: Single host or alias: LAN_DESKTOPS_NET

Log: ✅ (marque "Log packets that are handled by this rule")

Description: LAN → Desktops: BLOQUEAR (segmentação)
```
3. **Save**

---

### Regra 10: BLOQUEAR e LOGAR todo o resto (Opcional)

1. **Add** ↓ (adicionar no final)
2. Preencha:
```
Action: Block
Interface: LAN
Protocol: Any

Source: LAN net
Destination: Any

Log: ✅

Description: LAN → Any: BLOQUEAR resto (default deny)
```
3. **Save**

---

### Aplicar as mudanças:
1. Clique em **Apply Changes** no topo da página
2. Aguarde a mensagem de confirmação verde

---

### ✅ Ordem Final das Regras LAN:

```
1. ✅ Pass  | *      | *   | *       | LAN addr | 443,80      | Anti-Lockout
2. ✅ Pass  | TCP/UDP| *   | LAN net | LAN addr | 53          | LAN → pfSense: DNS
3. ✅ Pass  | TCP    | *   | LAN net | LAN addr | 443         | LAN → pfSense: WebGUI
4. ✅ Pass  | TCP/UDP| *   | LAN net | Any      | 53          | LAN → Internet: DNS
5. ✅ Pass  | TCP    | *   | LAN net | Any      | Portas_Web  | LAN → Internet: Web
6. ✅ Pass  | UDP    | *   | LAN net | Any      | 123         | LAN → Internet: NTP
7. ✅ Pass  | ICMP   | *   | LAN net | Any      | *           | LAN → Any: ICMP
8. ✅ Pass  | TCP    | *   | LAN net | LAN net  | Portas_Admin| LAN → LAN: SSH/RDP
9. ✅ Pass  | TCP    | *   | LAN_SERVIDORES_NET | LAN_K8S_NET | API_K8S    | LAN → K8S: API Access
10.❌ Block | Any    | *   | LAN net | LAN_DESKTOPS_NET | *           | LAN → Desktops: BLOCK
11.❌ Block | Any    | *   | LAN net | Any      | *           | LAN → Any: Default Deny
```

---

## 💻 Regras de Firewall - LAN_DESKTOPS (Clientes) {#regras-desktops}

### Preparação:
1. Navegue: **Firewall → Rules → LAN_DESKTOPS**
2. Exclua todas as regras permissivas (se houver)

---

### Regra 1: Permitir acesso ao pfSense (DNS)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: TCP/UDP

Source: LAN_DESKTOPS net
Destination: LAN_DESKTOPS address
Destination Port Range:
  From: DNS (53)
  To: DNS (53)

Description: Desktops → pfSense: DNS
```
3. **Save**

---

### Regra 2: Permitir acesso ao WebGUI do pfSense

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: TCP

Source: LAN_DESKTOPS net
Destination: LAN_DESKTOPS address
Destination Port Range:
  From: HTTPS (443)
  To: HTTPS (443)

Description: Desktops → pfSense: WebGUI
```
3. **Save**

---

### Regra 3: Permitir DNS para Internet

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: TCP/UDP

Source: LAN_DESKTOPS net
Destination: Any
Destination Port Range:
  From: DNS (53)
  To: DNS (53)

Description: Desktops → Internet: DNS
```
3. **Save**

---

### Regra 4: Permitir HTTP/HTTPS para Internet

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: TCP

Source: LAN_DESKTOPS net
Destination: Any
Destination Port Range:
  From: (other) Custom: Portas_Web
  To: (other) Custom: Portas_Web

Description: Desktops → Internet: Navegação web
```
3. **Save**

---

### Regra 5: Permitir acesso aos servidores (serviços específicos)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: TCP

Source: LAN_DESKTOPS net
Destination: Single host or alias: LAN_SERVIDORES_NET
Destination Port Range:
  From: (other) Custom: Portas_Servicos_LAN
  To: (other) Custom: Portas_Servicos_LAN

Description: Desktops → Servidores: Acesso a serviços
```
3. **Save**

---

### Regra 6: Permitir ICMP (Ping)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: ICMP
ICMP Type: Any

Source: LAN_DESKTOPS net
Destination: Any

Description: Desktops → Any: ICMP para diagnóstico
```
3. **Save**

---

### Regra 7: Permitir NTP

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_DESKTOPS
Protocol: UDP

Source: LAN_DESKTOPS net
Destination: Any
Destination Port Range:
  From: NTP (123)
  To: NTP (123)

Description: Desktops → Internet: NTP
```
3. **Save**

---

### Regra 8: BLOQUEAR e LOGAR todo o resto

1. **Add** ↓ (final)
2. Preencha:
```
Action: Block
Interface: LAN_DESKTOPS
Protocol: Any

Source: LAN_DESKTOPS net
Destination: Any

Log: ✅

Description: Desktops → Any: BLOQUEAR resto (default deny)
```
3. **Save**

---

### Aplicar as mudanças:
- **Apply Changes**

---

### ✅ Ordem Final das Regras LAN_DESKTOPS:

```
1. ✅ Pass  | TCP/UDP| *   | LAN_DESKTOPS_NET | LAN_DESKTOPS addr | 53          | Desktops → pfSense: DNS
2. ✅ Pass  | TCP    | *   | LAN_DESKTOPS_NET | LAN_DESKTOPS addr | 443         | Desktops → pfSense: WebGUI
3. ✅ Pass  | TCP/UDP| *   | LAN_DESKTOPS_NET | Any      | 53          | Desktops → Internet: DNS
4. ✅ Pass  | TCP    | *   | LAN_DESKTOPS_NET | Any      | Portas_Web  | Desktops → Internet: Web
5. ✅ Pass  | TCP    | *   | LAN_DESKTOPS_NET | LAN_SERVIDORES_NET | Portas_Servicos_LAN | Desktops → Servidores
6. ✅ Pass  | ICMP   | *   | LAN_DESKTOPS_NET | Any      | *           | Desktops → Any: ICMP
7. ✅ Pass  | UDP    | *   | LAN_DESKTOPS_NET | Any      | 123         | Desktops → Internet: NTP
8. ❌ Block | Any    | *   | LAN_DESKTOPS_NET | Any      | *           | Desktops → Any: Default
```

---

## Regras de Firewall - WAN {#regras-wan}

Regras aplicadas à interface de entrada da internet.

### ✅ Ordem Final das Regras WAN:

```
1. ✅ Pass  | TCP    | *   | IP_Admin_PC | LAN_K8S_NET | 22           | WAN → K8S: SSH do Admin
2. ✅ Pass  | TCP    | *   | IP_Admin_PC | LAN_SERVIDORES_NET | Portas_Web | WAN → Servidores: HTTP/HTTPS do Admin
3. ✅ Pass  | ICMP   | *   | WAN net     | WAN address | *            | WAN → WAN: ICMP
4. ✅ Pass  | TCP    | *   | WAN net     | WAN address | 443          | WAN → WAN: WebGUI
5. ✅ Pass  | TCP    | *   | Any         | 192.168.3.10| 8443         | WAN → K8S Host: K8S Dashboard (NAT)
6. ✅ Pass  | UDP    | *   | Any         | WAN address | 1194         | WAN → WAN: OpenVPN
```

---

## ☸️ Regras de Firewall - LAN_K8S (Kubernetes) {#regras-k8s}

### Preparação:
1. Navegue: **Firewall → Rules → LAN_K8S**
2. Exclua todas as regras permissivas (se houver)

---

### Regra 1: Permitir acesso ao pfSense (DNS)

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: TCP/UDP

Source: LAN_K8S net
Destination: LAN_K8S address
Destination Port Range:
  From: DNS (53)
  To: DNS (53)

Description: K8S → pfSense: DNS
```
3. **Save**

---

### Regra 2: Permitir comunicação interna no Cluster

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: Any

Source: LAN_K8S net
Destination: LAN_K8S net

Description: K8S → K8S: Comunicação interna
```
3. **Save**

---

### Regra 3: Permitir acesso ao TrueNAS

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: TCP/UDP

Source: LAN_K8S net
Destination: Single host or alias: IP_TrueNas
Destination Port Range:
  From: (other) Custom: Portas_Web
  To: (other) Custom: Portas_Web

Description: K8S → TrueNAS
```
3. **Save**

---

### Regra 4: Permitir retorno de conexão SSH para Admin

1. **Add** ↑
2. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: TCP

Source: LAN_K8S net
Destination: Single host or alias: IP_Admin_PC
Destination Port Range:
  From: SSH (22)
  To: SSH (22)

Description: K8S → Admin PC: Retorno SSH
```
3. **Save**

---

### Regra 5: Permitir acesso à Internet

1. **Add** ↑ (para Web)
2. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: TCP

Source: LAN_K8S net
Destination: Any
Destination Port Range:
  From: (other) Custom: Portas_Web
  To: (other) Custom: Portas_Web

Description: K8S → Internet: Navegação Web
```
3. **Save**

4. **Add** ↑ (para NTP)
5. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: UDP

Source: LAN_K8S net
Destination: Any
Destination Port Range:
  From: NTP (123)
  To: NTP (123)

Description: K8S → Internet: Sincronização NTP
```
6. **Save**

7. **Add** ↑ (para ICMP)
8. Preencha:
```
Action: Pass
Interface: LAN_K8S
Protocol: ICMP

Source: LAN_K8S net
Destination: Any

Description: K8S → Any: ICMP para diagnóstico
```
9. **Save**

---

### Regra 6: BLOQUEAR e LOGAR todo o resto

1. **Add** ↓ (final)
2. Preencha:
```
Action: Block
Interface: LAN_K8S
Protocol: Any

Source: LAN_K8S net
Destination: Any

Log: ✅

Description: K8S → Any: BLOQUEAR resto (default deny)
```
3. **Save**

---

### Aplicar as mudanças:
- **Apply Changes**

---

### ✅ Ordem Final das Regras LAN_K8S:

```
1. ✅ Pass  | TCP/UDP| LAN_K8S net | LAN_K8S addr | 53             | K8S → pfSense: DNS
2. ✅ Pass  | Any    | LAN_K8S net | LAN_K8S net  | *              | K8S → K8S: Comunicação interna
3. ✅ Pass  | TCP/UDP| LAN_K8S net | IP_TrueNas   | Portas_TrueNas | K8S → TrueNAS
4. ✅ Pass  | TCP    | LAN_K8S net | IP_Admin_PC  | 22             | K8S → Admin PC: Retorno SSH
5. ✅ Pass  | TCP    | LAN_K8S net | Any          | Portas_Web     | K8S → Internet: Web
6. ✅ Pass  | UDP    | LAN_K8S net | Any          | 123            | K8S → Internet: NTP
7. ✅ Pass  | ICMP   | LAN_K8S net | Any          | *              | K8S → Any: ICMP
8. ❌ Block | Any    | LAN_K8S net | Any          | *              | K8S → Any: Default Deny
```

---

## 🔄 Regras de NAT (Network Address Translation) {#regras-nat}

### NAT de Saída (Outbound)

O firewall está configurado no modo **Advanced Outbound NAT**, permitindo controle manual sobre como o tráfego das redes internas é traduzido para o IP da WAN.

| Rede de Origem | Interface de Saída | Descrição |
|:---------------|:-------------------|:------------|
| `192.168.3.0/24` | WAN | NAT LAN_K8S para WAN |
| `192.168.1.0/24` | WAN | NAT LAN_SERVIDORES para WAN |
| `192.168.2.0/24` | WAN | Regras automáticas para ISAKMP/VPN |

### NAT de Entrada (Port Forward)

Regras que redirecionam portas da WAN para serviços internos.

| Interface | Porta Externa | IP Interno | Porta Interna | Descrição |
|:----------|:--------------|:-----------|:--------------|:------------|
| WAN | 8443 | 192.168.3.10 | 8443 | NAT → VIP - K8S Dashboard |

---

## 🌐 Configurações de VPN {#regras-vpn}

- **OpenVPN**: Uma regra na interface `WAN` permite a conexão de clientes OpenVPN na porta `1194/UDP`.
- **IPsec**: Regras de NAT geradas automaticamente para o protocolo `ISAKMP` (porta 500). Configuração de um túnel IPsec.
**Em implantação**

---

## ✅ Testes e Validação {#testes}

### Teste 1: VM na LAN (Servidores)

#### Criar VM de teste:
1. No Proxmox, crie uma VM Ubuntu/Debian
2. Conecte à **vmbr1**
3. Configure DHCP ou IP estático:
   ```
   IP: 192.168.1.100
   Gateway: 192.168.1.1
   DNS: 192.168.1.1
   ```

#### Testes de conectividade:

```bash
# 1. Testar gateway
ping -c 4 192.168.1.1
# ✅ Deve funcionar

# 2. Testar DNS
ping -c 4 8.8.8.8
# ✅ Deve funcionar

# 3. Testar navegação
curl -I https://google.com
# ✅ Deve retornar HTTP 200

# 4. Testar acesso à rede de clientes (deve falhar)
ping -c 4 192.168.2.1
# ❌ Deve FALHAR (bloqueado pela regra)

# 5. Testar SSH/RDP entre servidores
# De outro servidor na mesma rede:
ssh usuario@192.168.1.101
# ✅ Deve funcionar

# 6. Testar NTP
ntpdate -q pool.ntp.org
# ✅ Deve funcionar
```

#### Verificar logs:

1. No pfSense: **Status → System Logs → Firewall**
2. Procure por entradas da origem `192.168.1.100`
3. Tentativa de acesso a `192.168.2.x` deve aparecer como **bloqueada** (vermelho)

---

### Teste 2: VM na LAN_DESKTOPS (Clientes)

#### Criar VM de teste:
1. Crie outra VM
2. Conecte à **vmbr2**
3. Configure:
   ```
   IP: 192.168.2.100 (ou DHCP)
   Gateway: 192.168.2.1
   DNS: 192.168.2.1
   ```

#### Testes de conectividade:

```bash
# 1. Testar gateway
ping -c 4 192.168.2.1
# ✅ Deve funcionar

# 2. Testar internet
ping -c 4 8.8.8.8
curl -I https://google.com
# ✅ Deve funcionar

# 3. Testar acesso aos servidores (portas permitidas)
curl http://192.168.1.100
# ✅ Deve funcionar se houver servidor web

# 4. Testar SSH para servidor
ssh usuario@192.168.1.100
# ✅ Deve funcionar (porta 22 permitida)

# 5. Testar porta não permitida (ex: MySQL)
telnet 192.168.1.100 3306
# ❌ Deve FALHAR ou timeout (bloqueado)
```

---

### Teste 3: Acesso ao pfSense pela WAN

De um computador na rede **192.168.0.0/24**:

```bash
# 1. Ping
ping 192.168.0.100
# ✅ Deve funcionar (se habilitou ICMP na WAN)

# 2. Acesso WebGUI
https://192.168.0.100
# ✅ Deve funcionar (regra criada anteriormente)
```

---

### Teste 4: Validar Suricata

#### Teste EICAR (Malware de teste):

Na VM cliente ou servidor:
```bash
# Baixar arquivo de teste
curl http://www.eicar.org/download/eicar.com.txt -o eicar.txt

# Ou via wget
wget http://www.eicar.org/download/eicar.com.txt
```

**Verificar no Suricata:**
1. **Services → Suricata → Alerts**
2. Deve aparecer alerta: `"ET INFO EICAR Test File Download"`
3. Severity: 3 (Informational)

#### Teste de Scan de Portas:

De outra máquina ou da internet:
```bash
# Scan básico
nmap -sS 192.168.0.100

# Scan agressivo
nmap -A -T4 192.168.0.100
```

**Verificar no Suricata:**
1. Deve detectar: `"ET SCAN NMAP"` ou similar
2. Se IPS ativo, pode bloquear o IP automaticamente

---

### Teste 5: Testar NAT

Verificar se o NAT está funcionando:

1. **Firewall → NAT → Outbound**
2. Verifique se está em "Automatic" ou se as regras manuais estão corretas

**Da VM cliente:**
```bash
# Verificar IP público
curl ifconfig.me
# Deve retornar o IP público do seu provedor (192.168.0.100 após NAT)
```

---

### Checklist de Validação:

- [ ] VMs pegam IP via DHCP
- [ ] VMs conseguem pingar o gateway (pfSense)
- [ ] VMs conseguem resolver DNS
- [ ] VMs conseguem acessar internet (HTTP/HTTPS)
- [ ] Servidores NÃO conseguem acessar rede de clientes
- [ ] Clientes conseguem acessar serviços nos servidores
- [ ] Acesso ao pfSense pela WAN funciona
- [ ] Logs mostram bloqueios corretos
- [ ] Suricata está rodando e gerando alertas
- [ ] NAT está funcionando (tráfego sai com IP da WAN)

---

## 📊 Monitoramento e Logs {#monitoramento}

### Dashboard do pfSense

1. Navegue: **Status → Dashboard**
2. Widgets úteis:
   - **System Information**: CPU, memória, uptime
   - **Interfaces**: Status e tráfego das interfaces
   - **Traffic Graphs**: Gráficos de tráfego em tempo real
   - **Firewall Logs**: Últimas entradas do firewall
   - **Gateway Status**: Status do gateway WAN

**Personalizar Dashboard:**
- Clique no ícone **+** para adicionar widgets
- Arraste para reorganizar

---

### Logs do Firewall

#### Visualizar logs em tempo real:

1. **Status → System Logs → Firewall**
2. Clique em **Normal View** ou **Dynamic View** (atualização automática)

**Filtros úteis:**
```
Interface: Selecione WAN, LAN, ou LAN_DESKTOPS
Action: Block ou Pass
Source: IP específico
Destination: IP específico
```

#### Interpretar logs:

**Linha típica:**
```
Oct 1 15:23:45  WAN  block  192.168.1.100:54321  192.168.2.50:80  TCP  PA
```

**Significado:**
- `Oct 1 15:23:45`: Timestamp
- `WAN`: Interface
- `block`: Ação (bloqueou)
- `192.168.1.100:54321`: IP origem:porta
- `192.168.2.50:80`: IP destino:porta
- `TCP`: Protocolo
- `PA`: Flags TCP (P=Push, A=Ack)

---

### Logs do Suricata

#### Visualizar alertas:

1. **Services → Suricata → Alerts**
2. Selecione a interface (dropdown)
3. Clique em **View**

**Filtros:**
- **Priority**: 1 (High), 2 (Medium), 3 (Low)
- **Category**: Malware, Exploit, Scan, etc.

#### Exportar alertas:

1. Na tela de Alerts
2. Clique em **Download** (ícone de download)
3. Formato CSV para análise

---

### Monitoramento de Estados

**Diagnostics → States**

Mostra todas as conexões ativas:
- Útil para troubleshooting
- Ver quem está conectado a quê
- Identificar conexões suspeitas

**Filtros úteis:**
```
Source Address: 192.168.1.100
Destination: 8.8.8.8
Protocol: TCP
```

---

### Monitoramento de Banda

**Status → Traffic Graph**

- Visualize o tráfego em tempo real
- Identifique picos de uso
- Selecione interface específica

**Para histórico:**
- Considere instalar **ntopng** (System → Package Manager)
- Fornece análise detalhada de tráfego

---

### Configurar Syslog Remoto (Opcional)

Para ambientes maiores, envie logs para servidor centralizado:

1. **Status → System Logs → Settings**
2. Em **Remote Logging Options**:
   ```
   Enable Remote Logging: ✅
   Remote log servers: 192.168.1.50:514
   Remote Syslog Contents: Everything
   ```
3. **Save**

---

### Alertas por Email (Opcional)

Configurar notificações:

1. **System → Advanced → Notifications**
2. Configure SMTP:
   ```
   E-Mail server: smtp.gmail.com
   SMTP Port: 587
   Secure SMTP Connection: Enable STARTTLS
   From e-mail address: seuemail@gmail.com
   Notification E-Mail address: destino@email.com
   Username: seuemail@gmail.com
   Password: [senha de app do Gmail]
   ```
3. **Test SMTP Settings**
4. **Save**

**Para Gmail:**
- Use "Senha de app" (não a senha normal)
- Ative autenticação de 2 fatores
- Gere senha de app em: https://myaccount.google.com/apppasswords

---

### Relatórios Periódicos

#### Status Report (semanal):

Crie um script para gerar relatórios:

1. **Diagnostics → Command Prompt**
2. Use comandos para coletar dados:
   ```bash
   # Top 10 IPs com mais tráfego
   pfctl -s state | awk '{print $3}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10
   
   # Conexões ativas
   pfctl -s state | wc -l
   ```

---

## 🔧 Manutenção e Boas Práticas {#manutencao}

### Rotina Diária

- [ ] Verificar Dashboard (status geral)
- [ ] Revisar alertas do Suricata (prioridade 1 e 2)
- [ ] Verificar logs de bloqueio suspeitos

---

### Rotina Semanal

- [ ] Revisar logs de firewall completos
- [ ] Atualizar regras do Suricata
- [ ] Verificar uso de CPU/RAM/Disco
- [ ] Testar backup da configuração
- [ ] Revisar lista de IPs bloqueados (false positives?)

---

### Rotina Mensal

- [ ] Atualizar pfSense (se houver updates)
- [ ] Revisar e ajustar regras de firewall
- [ ] Limpar logs antigos
- [ ] Documentar mudanças realizadas
- [ ] Teste de failover (se tiver HA)
- [ ] Revisar aliases (ainda relevantes?)

---

### Backup da Configuração

#### Backup Manual:

1. **Diagnostics → Backup & Restore**
2. Área: **Config History**
3. Clique em **Backup Configuration**
4. Arquivo XML será baixado
5. **Guarde em local seguro!**

#### Backup Automático:

O pfSense mantém backups automáticos:
- **Config History**: Últimas 30 alterações
- Acesse em: **Diagnostics → Backup & Restore → Config History**

#### Agendar backup remoto:

1. Instale o pacote **AutoConfigBackup**:
   - **System → Package Manager → Available Packages**
   - Procure `AutoConfigBackup`
   - **Install**

2. Configure backup para SFTP/FTP/Cloud

---

### Atualização do pfSense

1. **System → Update**
2. Verifique se há updates disponíveis
3. Leia as notas de release
4. **Confirme backup antes!**
5. Clique em **Confirm** para atualizar

**Recomendações:**
- Atualize em horário de baixo uso
- Teste em ambiente de lab primeiro
- Espere 1-2 semanas após release para estabilizar

---

### Documentação

#### Mantenha documentado:

1. **Diagrama de rede atualizado**
   ```
   WAN: 192.168.0.100/24 → Internet
   LAN: 192.168.1.0/24 → Servidores
   LAN_DESKTOPS: 192.168.2.0/24 → Clientes
   ```

2. **Inventário de regras**
   - Crie planilha com todas as regras
   - Documente o motivo de cada uma
   - Registre data de criação/modificação

3. **Mudanças realizadas**
   - Log de alterações
   - Data, responsável, motivo

4. **Credenciais**
   - Armazene em gerenciador de senhas
   - Não deixe em texto plano

---

### Hardening Adicional

#### 1. Mudar porta do WebGUI:

**System → Advanced → Admin Access**
```
TCP port: 8443 (em vez de 443)
```

#### 2. Restringir acesso administrativo:

**System → Advanced → Admin Access**
```
Anti-lockout: Apenas de IPs específicos
```

#### 3. Ativar autenticação de dois fatores (2FA):

1. Instale: **System → Package Manager → freeradius**
2. Configure TOTP (Google Authenticator)
3. **System → User Manager → Users → admin → Edit**
4. Habilite 2FA

#### 4. Desabilitar serviços não usados:

**System → Advanced → Miscellaneous**
```
Desabilite: UPnP, NAT-PMP se não usar
```

#### 5. Rate Limiting (proteção DoS):

**Firewall → Rules → WAN (ou LAN)**
- Limite conexões por IP
- Use "Advanced Options → Max States"

---

### Troubleshooting Comum

#### Problema: Não consigo acessar a internet

**Checklist:**
1. pfSense pinga 8.8.8.8? (**Diagnostics → Ping**)
2. Gateway WAN está online? (**System → Routing → Gateways**)
3. Regras de firewall permitem? (**Firewall → Rules**)
4. NAT está configurado? (**Firewall → NAT → Outbound**)
5. DNS funciona? (**Diagnostics → DNS Lookup**)

#### Problema: Suricata não inicia

**Soluções:**
1. Aumentar RAM da VM (mínimo 2GB)
2. Reduzir categorias ativas
3. Verificar logs: **Status → System Logs → System**

#### Problema: Muitos falsos positivos

**Soluções:**
1. Desabilitar categorias "policy" e "info"
2. Criar suppressions: **Services → Suricata → Suppress**
3. Ajustar perfil: Detection Performance → "Low"

#### Problema: Alto uso de CPU

**Causas:**
- Muitas regras do Suricata ativas
- Tráfego muito alto
- Hardware insuficiente

**Soluções:**
1. Reduzir categorias do Suricata
2. Desabilitar Suricata nas LANs (manter apenas WAN)
3. Aumentar vCPUs da VM

---

### Recursos de Aprendizado

#### Documentação Oficial:
- **pfSense Documentation**: https://docs.netgate.com/pfsense/
- **pfSense Forum**: https://forum.netgate.com/
- **Suricata Docs**: https://suricata.readthedocs.io/

#### Cursos e Certificações:
- **Netgate pfSense Fundamentals**: Curso oficial gratuito
- **pfSense CE Certified Engineer**: Certificação paga

#### Livros:
- "The pfSense Book" (disponível em docs.netgate.com)
- "Mastering pfSense" (Packt Publishing)

#### Comunidade:
- Reddit: r/pfSense
- YouTube: Lawrence Systems, Netgate

---

### Expansões Futuras

#### 1. Alta Disponibilidade (HA/CARP)
- Configurar dois pfSense em cluster
- Failover automático

#### 2. VPN Site-to-Site
- Conectar filiais
- IPsec ou OpenVPN

#### 3. VPN de Acesso Remoto
- OpenVPN ou WireGuard
- Acesso seguro para home office

#### 4. Captive Portal
- Autenticação de convidados
- Hotspot WiFi

#### 5. Traffic Shaping (QoS)
- Priorizar tráfego crítico
- Limitar banda de usuários/aplicações

#### 6. Multi-WAN
- Balanceamento de carga
- Redundância de links

---

## 📝 Checklist Final de Implementação

### Pré-Requisitos:
- [ ] pfSense instalado e configurado
- [ ] 3 interfaces de rede configuradas (WAN, LAN, LAN_DESKTOPS)
- [ ] Endereçamento IP correto

### Aliases:
- [ ] LAN_SERVIDORES_NET criado
- [ ] LAN_DESKTOPS_NET criado
- [ ] Portas_Web criado
- [ ] Portas_Admin criado
- [ ] Portas_Servicos_LAN criado

### Regras de Firewall - LAN:
- [ ] Regra Anti-Lockout (não modificar)
- [ ] Acesso ao pfSense (DNS, WebGUI)
- [ ] DNS para internet
- [ ] HTTP/HTTPS para internet
- [ ] NTP para internet
- [ ] ICMP permitido
- [ ] SSH/RDP entre servidores
- [ ] Bloqueio para rede de clientes
- [ ] Default deny no final

### Regras de Firewall - LAN_DESKTOPS:
- [ ] Acesso ao pfSense (DNS, WebGUI)
- [ ] DNS para internet
- [ ] HTTP/HTTPS para internet
- [ ] Acesso aos servidores (portas específicas)
- [ ] ICMP permitido
- [ ] NTP para internet
- [ ] Default deny no final

### Regras de Firewall - LAN_K8S:
- [ ] Acesso ao pfSense (DNS)
- [ ] Comunicação interna no Cluster
- [ ] Acesso ao TrueNAS
- [ ] Retorno de conexão SSH para Admin
- [ ] Acesso à Internet (Web, NTP, ICMP)
- [ ] Default deny no final


### Testes:
- [ ] VM na LAN acessa internet
- [ ] VM na LAN NÃO acessa LAN_DESKTOPS
- [ ] VM na LAN_DESKTOPS acessa internet
- [ ] VM na LAN_DESKTOPS acessa serviços nos servidores
- [ ] Acesso ao pfSense pela WAN funciona
- [ ] Suricata detecta ameaças (teste EICAR)
- [ ] Logs registram bloqueios corretamente

### Monitoramento:
- [ ] Dashboard configurado
- [ ] Logs do firewall revisados
- [ ] Alertas do Suricata funcionando
- [ ] Backup da configuração realizado

### Documentação:
- [ ] Diagrama de rede documentado
- [ ] Regras de firewall documentadas
- [ ] Credenciais armazenadas com segurança
- [ ] Log de mudanças iniciado

---

## 🎯 Conclusão


✅ **Firewall pfSense com segmentação de rede**
- Servidores isolados de clientes
- Regras baseadas no princípio do menor privilégio

✅ **Política de segurança robusta**
- Apenas tráfego necessário permitido
- Logs e auditoria configurados

✅ **Base para ambiente de produção**
- Arquitetura escalável
- Boas práticas aplicadas

---

## 📚 Próximos Passos Sugeridos

1. **Familiarize-se com os logs**: Passe alguns dias monitorando
2. **Teste cenários de ataque**: Use Kali Linux para testar sua defesa
3. **Implemente VPN**: Para acesso remoto seguro
4. **Explore ntopng**: Para análise detalhada de tráfego
5. **Configure alertas**: Email ou Telegram para alertas críticos
6. **Documente tudo**: Mantenha documentação atualizada
7. **Pratique recovery**: Simule falhas e pratique restauração

---

## ⚠️ Avisos Importantes

- Este guia é para **ambiente de laboratório/estudo**
- Para **produção**, considere consultoria especializada
- **Sempre faça backup** antes de mudanças críticas
- **Teste em horário de baixo impacto**
- **Mantenha senhas fortes** e únicas
- **Atualize regularmente** pfSense e regras do Suricata

---

## 🆘 Suporte

Se precisar de ajuda:
- **Fórum pfSense**: https://forum.netgate.com/
- **Reddit**: r/pfSense
- **Discord**: Netgate Community

---

**Versão do Guia**: 1.1
**Data**: Outubro 2025  
**Compatível com**: pfSense CE 2.7.x  

| 🏁 Sumário | [WIKI_SUMARIO.md](WIKI_SUMARIO.md) |