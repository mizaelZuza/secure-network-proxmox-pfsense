# Guia Completo: Implementação de Firewall pfSense Seguro + IDS/IPS

## 📋 Sumário
1. [Arquitetura da Rede](#arquitetura)
2. [Políticas de Segurança](#politicas)
3. [Criação de Aliases](#aliases)
4. [Regras de Firewall - LAN (Servidores)](#regras-lan)
5. [Regras de Firewall - LAN_DESKTOPS (Clientes)](#regras-desktops)
6. [Implementação de IDS/IPS com Suricata](#suricata)
7. [Testes e Validação](#testes)
8. [Monitoramento e Logs](#monitoramento)
9. [Manutenção e Boas Práticas](#manutencao)

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
```

### Endereçamento IP:
| Interface | Bridge | IP do pfSense | Rede | Gateway | Função |
|-----------|--------|---------------|------|---------|--------|
| WAN | vmbr0 | 192.168.0.100 | 192.168.0.0/24 | 192.168.0.1 | Internet |
| LAN | vmbr1 | 192.168.1.1 | 192.168.1.0/24 | - | Servidores |
| LAN_DESKTOPS | vmbr2 | 192.168.2.1 | 192.168.2.0/24 | - | Clientes |

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
   Name: Rede_Servidores
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
   Name: Rede_Clientes
   Description: Rede dos clientes/desktops (vmbr2)
   Type: Network(s)
   Network: 192.168.2.0/24
   ```
3. **Save**

### Passo 4: Criar Alias - Portas Web

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

### Passo 5: Criar Alias - Portas de Administração

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

### Passo 6: Criar Alias - Serviços Permitidos

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

### Passo 7: Aplicar Mudanças
- Clique em **Apply Changes** no topo da página

### ✅ Aliases Criados:
- ✅ Rede_Servidores (192.168.1.0/24)
- ✅ Rede_Clientes (192.168.2.0/24)
- ✅ Portas_Web (80, 443)
- ✅ Portas_Admin (22, 3389)
- ✅ Portas_Servicos_LAN (80, 443, 22, 3389)

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
  From: HTTP (80)
  To: HTTPS (443)

Description: LAN → Internet: Navegação web
```
3. **Save**

**Alternativa usando Alias:**
```
Destination Port Range:
  From: (other) Custom: Portas_Web
  To: (other) Custom: Portas_Web
```

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

### Regra 8: BLOQUEAR acesso à rede de clientes

1. **Add** ↑
2. Preencha:
```
Action: Block
Interface: LAN
Protocol: Any

Source: LAN net
Destination: Single host or alias: Rede_Clientes

Log: ✅ (marque "Log packets that are handled by this rule")

Description: LAN → Desktops: BLOQUEAR (segmentação)
```
3. **Save**

---

### Regra 9: BLOQUEAR e LOGAR todo o resto (Opcional)

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
5. ✅ Pass  | TCP    | *   | LAN net | Any      | 80,443      | LAN → Internet: Web
6. ✅ Pass  | UDP    | *   | LAN net | Any      | 123         | LAN → Internet: NTP
7. ✅ Pass  | ICMP   | *   | LAN net | Any      | *           | LAN → Any: ICMP
8. ✅ Pass  | TCP    | *   | LAN net | LAN net  | 22,3389     | LAN → LAN: SSH/RDP
9. ❌ Block | Any    | *   | LAN net | Rede_Cli | *           | LAN → Desktops: BLOCK
10.❌ Block | Any    | *   | LAN net | Any      | *           | LAN → Any: Default Deny
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
  From: HTTP (80)
  To: HTTPS (443)

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
Destination: Single host or alias: Rede_Servidores
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
1. ✅ Pass  | TCP/UDP| *   | Desktops | pfSense  | 53          | Desktops → pfSense: DNS
2. ✅ Pass  | TCP    | *   | Desktops | pfSense  | 443         | Desktops → pfSense: WebGUI
3. ✅ Pass  | TCP/UDP| *   | Desktops | Any      | 53          | Desktops → Internet: DNS
4. ✅ Pass  | TCP    | *   | Desktops | Any      | 80,443      | Desktops → Internet: Web
5. ✅ Pass  | TCP    | *   | Desktops | Servs    | 80,443,22.. | Desktops → Servidores
6. ✅ Pass  | ICMP   | *   | Desktops | Any      | *           | Desktops → Any: ICMP
7. ✅ Pass  | UDP    | *   | Desktops | Any      | 123         | Desktops → Internet: NTP
8. ❌ Block | Any    | *   | Desktops | Any      | *           | Desktops → Any: Default
```

---

## 🔍 Implementação de IDS/IPS com Suricata {#suricata}

### O que é Suricata?

**Suricata** é um sistema de **detecção e prevenção de intrusão** (IDS/IPS) open-source que:
- Analisa o tráfego em tempo real
- Detecta ataques e malware
- Pode bloquear automaticamente ameaças (modo IPS)
- Usa assinaturas (rules) constantemente atualizadas

### IDS vs IPS:
- **IDS (Intrusion Detection System)**: Apenas **alerta** sobre ameaças
- **IPS (Intrusion Prevention System)**: **Bloqueia** ameaças automaticamente

---

### Parte 1: Instalação do Suricata

#### Passo 1: Acessar o Package Manager

1. Login no pfSense
2. Navegue: **System → Package Manager**
3. Clique na aba **Available Packages**

#### Passo 2: Instalar Suricata

1. Use o campo de busca: digite `suricata`
2. Localize o pacote **suricata**
3. Clique em **Install**
4. Confirme clicando em **Confirm**
5. Aguarde a instalação (pode levar alguns minutos)
6. Quando terminar, aparecerá: "Package suricata installed successfully"

---

### Parte 2: Configuração Inicial do Suricata

#### Passo 1: Acessar Suricata

1. Navegue: **Services → Suricata**
2. Clique na aba **Global Settings**

#### Passo 2: Configurações Globais

```
Enable Suricata: ✅ (marque)
Enable Live Rule Swap on Update: ✅
Remove Blocked Hosts Interval: 1 hour
Keep Suricata Settings After Deinstall: ✅
Hide Deprecated Rules Categories: ✅
```

**Save** (no final da página)

---

### Parte 3: Baixar Regras (Rules)

#### Passo 1: Acessar Updates

1. Navegue: **Services → Suricata**
2. Clique na aba **Updates**

#### Passo 2: Selecionar fontes de regras

**Opções gratuitas recomendadas:**

```
ETOpen Emerging Threats rules: ✅
  - Regras gratuitas, atualizadas regularmente
  - Cobertura ampla de ameaças

Snort GPLv2 Community rules: ✅
  - Regras da comunidade Snort
  - Complementa as ETOpen

Abuse.ch SSL Blacklist: ✅
  - Lista de certificados SSL maliciosos

Abuse.ch Feodo Tracker: ✅
  - Lista de servidores C&C de botnets
```

**⚠️ Nota sobre Snort e ET Pro:**
- Requerem registro (gratuito ou pago)
- Para laboratório, as opções acima são suficientes

#### Passo 3: Atualizar regras

1. Role até o final da página
2. Clique em **Update** (botão com ícone de download)
3. Aguarde o download (pode levar vários minutos na primeira vez)
4. Status mudará para "Success" quando concluir

---

### Parte 4: Configurar Interface WAN

#### Passo 1: Adicionar Interface

1. Navegue: **Services → Suricata**
2. Aba **Interfaces**
3. Clique em **Add**

#### Passo 2: Configurar Interface WAN

**Enable:**
```
Enable: ✅
Interface: WAN
Description: Suricata on WAN
```

**Alert Settings:**
```
Block Offenders: ✅ (modo IPS - bloqueia ameaças)
Kill States: ✅ (remove conexões ativas de IPs bloqueados)
IPS Mode: Legacy Mode
```

**Choose the networks Suricata should inspect:**
```
WAN and Local Networks
```

**Scroll para baixo e clique em Save**

---

### Parte 5: Configurar Categorias de Regras

#### Passo 1: Acessar WAN Categories

1. Em **Services → Suricata → Interfaces**
2. Clique no ícone de **categorias** (ícone de lista) na linha da WAN

#### Passo 2: Selecionar categorias (Recomendação para laboratório)

**Emerging Threats (ET) - Categorias recomendadas:**

```
Categoria                          | Ativar | Descrição
-----------------------------------|--------|---------------------------
✅ emerging-attack_response         | Sim    | Respostas a ataques
✅ emerging-malware                 | Sim    | Malware conhecido
✅ emerging-exploit                 | Sim    | Exploits de vulnerabilidades
✅ emerging-scan                    | Sim    | Varreduras de porta
✅ emerging-dos                     | Sim    | Ataques DoS/DDoS
✅ emerging-phishing                | Sim    | Tentativas de phishing
✅ emerging-botcc                   | Sim    | Command & Control de botnets
✅ emerging-trojan                  | Sim    | Trojans
✅ emerging-worm                    | Sim    | Worms
❌ emerging-policy                  | Não    | Gera muitos falsos positivos
❌ emerging-info                    | Não    | Informativo apenas
```

**⚠️ Cuidado:**
- Não ative TODAS as categorias
- Muitas regras = alto uso de CPU e falsos positivos
- Comece conservador e vá ajustando

#### Passo 3: Aplicar seleção

1. Role até o final
2. **Save** (botão no final)

---

### Parte 6: Configurar Interface LAN (Opcional)

Para monitorar tráfego interno suspeito:

1. **Services → Suricata → Interfaces**
2. **Add**
3. Configure:
```
Enable: ✅
Interface: LAN
Description: Suricata on LAN
Block Offenders: ✅
IPS Mode: Legacy Mode
Networks: LAN and Local Networks
```
4. **Save**
5. Configure categorias similares à WAN (mas pode ser mais restritivo)

---

### Parte 7: Iniciar o Suricata

#### Passo 1: Iniciar nas interfaces

1. **Services → Suricata → Interfaces**
2. Clique no **ícone de play** (▶️) na coluna da WAN
3. Status mudará para "Running" (ícone verde)
4. Repita para LAN se configurou

#### Passo 2: Verificar se está funcionando

1. **Services → Suricata → Interfaces**
2. Verifique:
   - Status: **Running** (bolinha verde)
   - Coluna "Alerts": começará a incrementar conforme detectar ameaças

---

### Parte 8: Monitorar Alertas

#### Visualizar Alertas:

1. **Services → Suricata → Alerts**
2. Selecione a interface (WAN ou LAN) no dropdown
3. Clique em **View**

**Colunas importantes:**
- **Timestamp**: Quando ocorreu
- **SID**: ID da regra que disparou
- **Severity**: Gravidade (1=crítico, 2=alto, 3=médio)
- **Source/Dest**: IPs origem e destino
- **Message**: Descrição da ameaça

#### Bloquear IPs manualmente:

- Clique no ícone **+** ao lado de um alerta
- O IP será bloqueado automaticamente

---

### Parte 9: Ajustes Finos e Supressões

#### Falsos Positivos:

Se uma regra estiver gerando muitos alertas falsos:

1. **Services → Suricata → Suppress**
2. Clique na interface (WAN ou LAN)
3. **Add**
4. Configure:
```
Type: Suppress
SID: [número da regra]
Track By: Source ou Destination
IP Address: [IP específico ou rede]
Description: [motivo da supressão]
```
5. **Save**

---

### Parte 10: Configurações Avançadas (Opcional)

#### Ajustar Performance:

1. **Services → Suricata → Interface WAN (editar)**
2. Aba **Detection Performance Settings**

**Para ambiente de laboratório/baixo tráfego:**
```
Search Optimize: Low
IDS Profile: Community
```

**Para ambientes com mais tráfego:**
```
Search Optimize: Medium
IDS Profile: Balanced
```

---

### Manutenção do Suricata

#### Atualizar Regras Regularmente:

**Manual:**
1. **Services → Suricata → Updates**
2. **Update** (recomendado semanalmente)

**Automático:**
1. **Services → Suricata → Global Settings**
2. Configure:
```
Update Interval: 12 hours (ou 1 day)
Update Start Time: 00:00
```

#### Limpar Logs Antigos:

1. **Services → Suricata → Logs View**
2. Selecione a interface
3. Clique em **Clear** para limpar logs antigos

---

### Testes do Suricata

#### Teste 1: EICAR Test File (Malware de teste)

De uma VM, baixe o arquivo de teste EICAR:

```bash
curl http://www.eicar.org/download/eicar.com.txt
```

**Esperado:** Suricata deve detectar e alertar (ou bloquear se IPS ativo)

#### Teste 2: Scan de Portas

De fora da rede (ou de uma VM), faça um scan:

```bash
nmap -sS 192.168.0.100
```

**Esperado:** Suricata deve detectar o scan e alertar

#### Teste 3: Verificar Logs

1. **Services → Suricata → Alerts**
2. Deve aparecer alertas dos testes acima

---

### Resolução de Problemas

#### Suricata não inicia:

**Causa comum:** Memória insuficiente

**Solução:**
1. Aumente RAM da VM pfSense para pelo menos 2GB
2. Desabilite categorias de regras menos importantes
3. **Services → Suricata → Interface (editar) → Detection Performance**: escolha "Low"

#### Alto uso de CPU:

**Soluções:**
1. Reduza número de categorias ativas
2. Ajuste "Detection Performance" para "Low"
3. Considere usar apenas modo IDS (não IPS) nas LANs

#### Muitos falsos positivos:

**Soluções:**
1. Revise as categorias ativas
2. Desative categorias "policy" e "info"
3. Use suppressions para regras específicas
4. Ajuste "IDS Profile" para "Balanced"

---

### Recursos Adicionais sobre Suricata

**Documentação Oficial:**
- https://docs.netgate.com/pfsense/en/latest/packages/suricata/

**Emerging Threats Rules:**
- https://rules.emergingthreats.net/

**Suricata Documentation:**
- https://suricata.readthedocs.io/

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
- [ ] Rede_Servidores criado
- [ ] Rede_Clientes criado
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

### Suricata (IDS/IPS):
- [ ] Suricata instalado
- [ ] Regras baixadas (ETOpen, Snort Community, etc.)
- [ ] Interface WAN configurada
- [ ] Categorias selecionadas
- [ ] Modo IPS ativado
- [ ] Suricata rodando

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

Parabéns! Você implementou:

✅ **Firewall pfSense com segmentação de rede**
- Servidores isolados de clientes
- Regras baseadas no princípio do menor privilégio

✅ **IDS/IPS com Suricata**
- Detecção de ameaças em tempo real
- Bloqueio automático de ataques

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

**Versão do Guia**: 1.0  
**Data**: Outubro 2025  
**Compatível com**: pfSense CE 2.7.x  

