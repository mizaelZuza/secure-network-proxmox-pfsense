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

## 📝 Checklist Final de Implementação

### Suricata (IDS/IPS):
- [ ] Suricata instalado
- [ ] Regras baixadas (ETOpen, Snort Community, etc.)
- [ ] Interface WAN configurada
- [ ] Categorias selecionadas
- [ ] Modo IPS ativado
- [ ] Suricata rodando

---
## 🎯 Conclusão

✅ **IDS/IPS com Suricata**
- Detecção de ameaças em tempo real
- Bloqueio automático de ataques


| 🏁 Sumário | [WIKI_SUMARIO.md](WIKI_SUMARIO.md) |