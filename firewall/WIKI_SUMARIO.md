# 🧭 Sumário Wiki — Projeto de Firewall Seguro com pfSense + IDS/IPS (Suricata)

## 📘 Visão Geral do Projeto

> **Título:** Implementação de Firewall pfSense Seguro com IDS/IPS no Proxmox  
> **Autor:** Mizael Zuza  
> **Versão:** 1.0  
> **Data:** 2025-10-19  

📄 **Documentos principais:**
- [README.md](README.md) → visão geral e objetivos do projeto  
- [pfsense-security-guide.md](pfsense-security-guide.md) → documentação técnica de segmentação e regras  
- [instalacao-opicional.md](instalacao-opicional.md) → guia detalhado de implementação do Suricata  

---

## 🏗️ Estrutura Geral do Projeto

```
📂 Projeto-Firewall-Seguro
 ├── README.md                         → Apresentação e visão geral
 ├── pfsense-security-guide.md         → Guia completo de configuração do pfSense
 ├── instalacao-opicional.md           → Implementação detalhada do Suricata IDS/IPS
 ├── diagramas/
 │   ├── arquitetura-rede.png          → Topologia geral da rede (pfSense + Proxmox)
 │   └── fluxo-seguranca.png           → Fluxo lógico de inspeção e segurança
 ├── backups/
 │   └── pfsense-config.xml            → Backup de configuração exportado
 └── docs/
     └── referencias.md                → Links úteis, documentação e fontes externas
```

---

## 📑 Índice de Conteúdo

### 1️⃣ [README.md — Introdução e Arquitetura](README.md)

#### 🧱 Estrutura
- 🎯 **Objetivo**
- 🛡️ **Arquitetura da Rede**
- 🔒 **Políticas de Segurança**
- 📊 **Matriz de Acesso**
- 🔥 **Firewall pfSense e IDS/IPS Suricata**
- ✅ **Testes e Validação**
- 📈 **Monitoramento e Logs**
- 🔧 **Manutenção e Boas Práticas**
- 🔐 **Hardening Adicional**
- 🚀 **Próximos Passos**
- 📖 **Documentação Detalhada**

📎 **Links de referência:**
- [Documentação pfSense](https://docs.netgate.com/pfsense/en/latest/)
- [Documentação Suricata](https://suricata.readthedocs.io/)
- [Emerging Threats Rules](https://rules.emergingthreats.net/)

---

### 2️⃣ [pfsense-security-guide.md — Configuração do Firewall e Segmentação](pfsense-security-guide.md)

#### 🔧 Conteúdo Técnico
- 🌐 **Configuração de Interfaces no Proxmox**
- 🧩 **Instalação e Setup inicial do pfSense**
- 🗺️ **Criação de Bridges e VLANs**
- 🔒 **Definição de Aliases e Regras de Firewall**
- 📦 **Configuração de NAT e Port Forward**
- 🧠 **Políticas de Segmentação e Isolamento**
- 📡 **Validação de Conectividade**
- 🛠️ **Troubleshooting e Boas Práticas**
- 📋 **Checklist Final de Segurança**

📎 **Integrações:**
- Link direto para IDS/IPS: [→ Implementação de IDS/IPS com Suricata](instalacao-opicional.md#suricata)
- Link para retorno ao topo da wiki: [↑ Voltar ao Sumário](#-sumário-wiki--projeto-de-firewall-seguro-com-pfsense--idsips-suricata)

---

### 3️⃣ [instalacao-opicional.md — Implementação do IDS/IPS Suricata](instalacao-opicional.md)

#### 🔍 Conteúdo
- ⚙️ **Instalação do Suricata**
- 🔧 **Configurações Iniciais**
- 📥 **Download de Regras (ETOpen, Snort GPLv2)**
- 🌐 **Configuração em Interfaces WAN/LAN**
- 🧾 **Seleção de Categorias e Ajustes**
- ▶️ **Inicialização e Testes (EICAR, nmap, etc.)**
- 🧠 **Supressões e Falsos Positivos**
- ⚙️ **Ajustes de Performance**
- 🧰 **Troubleshooting e Manutenção**
- 📊 **Monitoramento e Logs**
- 🧾 **Checklist Final de Implementação**

📎 **Links diretos dentro do arquivo:**
- [Parte 1: Instalação do Suricata](instalacao-opicional.md#parte-1-instalação-do-suricata)
- [Parte 5: Categorias de Regras](instalacao-opicional.md#parte-5-configurar-categorias-de-regras)
- [Parte 9: Ajustes Finos e Supressões](instalacao-opicional.md#parte-9-ajustes-finos-e-supressões)
- [Testes do Suricata](instalacao-opicional.md#testes-do-suricata)
- [Resolução de Problemas](instalacao-opicional.md#resolução-de-problemas)

---

## 🧰 Recursos Complementares

### 📘 Documentação Oficial
| Componente | Link Oficial |
|-------------|--------------|
| **pfSense** | [https://docs.netgate.com/pfsense](https://docs.netgate.com/pfsense) |
| **Suricata** | [https://suricata.readthedocs.io](https://suricata.readthedocs.io) |
| **Emerging Threats Rules** | [https://rules.emergingthreats.net](https://rules.emergingthreats.net) |

### 🧩 Ferramentas de Apoio
- **Proxmox VE Docs:** https://pve.proxmox.com/pve-docs/
- **TrueNAS Docs:** https://www.truenas.com/docs/
- **Nmap (teste de IDS):** https://nmap.org/

---

## 📋 Checklist Geral do Projeto

| Etapa | Status | Descrição |
|:------|:--------:|:----------|
| Configuração de bridges no Proxmox | ✅ | vmbr0–vmbr3 configuradas |
| Instalação e setup do pfSense | ✅ | Rede e regras aplicadas |
| Segmentação de rede (Servidores, Clientes, K8s) | ✅ | Isolamento funcional |
| Implementação de IDS/IPS com Suricata | ✅ | Modo IPS ativo na WAN |
| Testes de validação (EICAR, Nmap, etc.) | ✅ | Alertas e bloqueios confirmados |
| Logs e monitoramento configurados | ✅ | Dashboard e alertas ativos |
| Backup e manutenção agendada | ✅ | Configuração exportada e rotina definida |

---

## 🧱 Próximas Expansões

| Tópico | Descrição | Status |
|--------|------------|--------|
| 🔁 **Alta Disponibilidade (CARP)** | Redundância com failover automático | ⏳ Planejado |
| 🔐 **VPN Site-to-Site / Cliente** | Túnel seguro entre redes remotas | ⏳ Planejado |
| 🚦 **QoS e Traffic Shaping** | Controle de banda e priorização | ⏳ Planejado |
| 🌍 **pfBlockerNG + GeoIP** | Bloqueio por país e DNSBL | ⏳ Planejado |
| 🧠 **Hardening Avançado** | Tuning de kernel e Fail2ban | ⏳ Planejado |

---

## 🔗 Navegação Rápida

| Seção | Link |
|-------|------|
| 🏁 Início | [README.md](README.md) |
| 🧱 Firewall e Segmentação | [pfsense-security-guide.md](pfsense-security-guide.md) |
| 🔍 IDS/IPS Suricata | [instalacao-opicional.md](instalacao-opicional.md) |
| 📚 Recursos e Referências | [docs/referencias.md](docs/referencias.md) |


