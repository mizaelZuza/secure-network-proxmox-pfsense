# Projeto: Implementação de Firewall pfSense Seguro com IDS/IPS no Proxmox

## 🎯 Objetivo

Este projeto tem como objetivo implementar uma infraestrutura de rede segura utilizando o firewall pfSense virtualizado no Proxmox. A configuração visa proteger uma rede local segmentada em múltiplas sub-redes (servidores, clientes e Kubernetes) através de políticas de segurança bem definidas e um sistema de detecção e prevenção de intrusão (IDS/IPS) com Suricata.

## 🛡️ Arquitetura da Rede

A topologia da rede implementada é a seguinte:

```
Internet (Provedor)
    ↓
192.168.0.0/24 (vmbr0 - WAN)
    ↓
[pfSense Firewall]
    ├─→ 192.168.1.0/24 (vmbr1 - LAN Servidores)
    ├─→ 192.168.2.0/24 (vmbr2 - LAN Clientes)
    └─→ 192.168.3.0/24 (vmbr3 - LAN Kubernetes)
```

### Endereçamento IP

| Interface    | Descrição   | Bridge | IP do pfSense | Rede             | IPv6  | Função     |
| :----------- | :---------- | :----- | :------------- | :--------------- | :---- | :--------- |
| WAN          | WAN         | vmbr0  | 192.168.0.100  | 192.168.0.0/24   | -     | Internet   |
| LAN          | LAN         | vmbr1  | 192.168.1.1    | 192.168.1.0/24   | Track | Servidores |
| OPT1         | LAN_DESKTOPS| vmbr2  | 192.168.2.1    | 192.168.2.0/24   | -     | Clientes   |
| OPT2         | LAN_K8S     | vmbr3  | 192.168.3.1    | 192.168.3.0/24   | -     | Kubernetes |

## 🔒 Políticas de Segurança

Os seguintes princípios de segurança são aplicados:

-   **Menor Privilégio**: Liberar apenas o acesso necessário para cada componente da rede.
-   **Segmentação**: Isolar servidores, clientes e o cluster Kubernetes para limitar o impacto de possíveis ataques.
-   **Defesa em Profundidade**: Implementar múltiplas camadas de segurança para aumentar a proteção.
-   **Log e Auditoria**: Registrar tentativas de acesso e eventos relevantes para análise e auditoria.

### Matriz de Acesso

| Origem → Destino | Internet | pfSense | Servidores (LAN) | Clientes (Desktops) | Kubernetes (LAN_K8S) |
| :--------------- | :------- | :------ | :--------------- | :------------------ | :------------------- |
| **Servidores**   | HTTP/HTTPS, DNS, NTP, ICMP | DNS, WebGUI | SSH, RDP | ❌ Bloqueado | API K8S |
| **Clientes**     | HTTP/HTTPS, DNS, ICMP | DNS, WebGUI | HTTP/HTTPS, SSH, RDP | Entre si OK | ❌ Bloqueado |
| **Kubernetes**   | HTTP/HTTPS, DNS, NTP, ICMP | DNS | TrueNAS | ❌ Bloqueado | Interno OK |
| **Internet**     | - | Apenas admin (192.168.0.0/24) | ❌ Bloqueado | ❌ Bloqueado | Apenas NAT/Port Forward |

## 🔥 Firewall pfSense e IDS/IPS Suricata

O pfSense é configurado com regras de firewall detalhadas para controlar o tráfego entre as redes (LAN, LAN_DESKTOPS, LAN_K8S) e a Internet (WAN), seguindo rigorosamente o princípio do "default deny" (bloquear tudo que não é explicitamente permitido). As regras são organizadas por interface e visam garantir a segmentação e o menor privilégio.

Adicionalmente, o **Suricata** é implementado como um sistema de detecção e prevenção de intrusão (IDS/IPS). Ele monitora o tráfego em tempo real, utilizando um conjunto de regras (ETOpen, Snort GPLv2 Community, etc.) para detectar e, opcionalmente, bloquear automaticamente ameaças como malware, exploits, varreduras de porta e ataques DoS. A configuração do Suricata é otimizada para balancear desempenho e eficácia na detecção.

### Aliases

Aliases são utilizados para facilitar a gestão e manutenção das regras do firewall, agrupando IPs, redes e portas. Os seguintes aliases são criados:

-   `LAN_SERVIDORES_NET` (192.168.1.0/24)
-   `LAN_DESKTOPS_NET` (192.168.2.0/24)
-   `LAN_K8S_NET` (192.168.3.0/24)
-   `IP_TrueNas` (IP do servidor TrueNAS, e.g., 192.168.1.114)
-   `IP_Admin_PC` (IP do PC de administração na WAN, e.g., 192.168.0.10)
-   `Portas_Web` (80, 443)
-   `Portas_Admin` (22, 3389)
-   `Portas_Servicos_LAN` (80, 443, 22, 3389)
-   `API_K8S` (6443)

## ✅ Testes e Validação

O projeto inclui um guia detalhado para testar e validar a configuração do firewall e do IDS/IPS. Isso envolve a criação de VMs de teste em cada segmento de rede e a execução de comandos para verificar a conectividade, o bloqueio de tráfego indesejado, o funcionamento do DNS, NTP, acesso web, SSH/RDP, e a detecção de ameaças pelo Suricata (e.g., teste EICAR, scan de portas).

## 📊 Monitoramento e Logs

Para garantir a segurança contínua e a visibilidade da rede, são configuradas diversas ferramentas de monitoramento e log:

-   **Dashboard do pfSense**: Visão geral do sistema, interfaces e tráfego.
-   **Logs do Firewall**: Visualização em tempo real e filtragem de eventos de bloqueio e permissão.
-   **Logs do Suricata**: Monitoramento de alertas de intrusão, com detalhes sobre a ameaça, origem e destino.
-   **Monitoramento de Estados**: Visualização de conexões ativas para troubleshooting.
-   **Monitoramento de Banda**: Gráficos de tráfego em tempo real.
-   **Syslog Remoto (Opcional)**: Envio de logs para um servidor centralizado.
-   **Alertas por Email (Opcional)**: Notificações para eventos críticos.

## 🔧 Manutenção e Boas Práticas

Uma rotina de manutenção é essencial para a segurança do ambiente, incluindo:

-   **Rotinas Diárias/Semanais/Mensais**: Verificação de dashboards, alertas, logs, uso de recursos, backups e atualizações.
-   **Backup da Configuração**: Procedimentos para backup manual e automático da configuração do pfSense.
-   **Atualização do pfSense**: Guia para manter o sistema atualizado com as últimas correções de segurança.
-   **Documentação**: Manutenção de diagramas de rede, inventário de regras, log de mudanças e credenciais.

## 🔒 Hardening Adicional

Para aumentar ainda mais a segurança do pfSense, são sugeridas práticas como:

-   Mudar a porta do WebGUI.
-   Restringir o acesso administrativo a IPs específicos.
-   Ativar autenticação de dois fatores (2FA).
-   Desabilitar serviços não utilizados.
-   Configurar Rate Limiting para proteção contra DoS.

## 📚 Próximos Passos Sugeridos

Após a implementação básica, são sugeridas expansões e melhorias como:

-   Alta Disponibilidade (HA/CARP).
-   VPN Site-to-Site e de Acesso Remoto.
-   Captive Portal.
-   Traffic Shaping (QoS).
-   Multi-WAN.

## 📖 Documentação Detalhada

Para instruções passo a passo e informações aprofundadas sobre cada aspecto da configuração, consulte os seguintes guias:

-   **Guia Completo: Implementação de Firewall pfSense Seguro + IDS/IPS**: [pfsense-security-guide.md](pfsense-security-guide.md)
-   **Implementação de IDS/IPS com Suricata**: [instalacao-opicional.md](instalacao-opicional.md)


| 🏁 Sumário | [WIKI_SUMARIO.md](WIKI_SUMARIO.md) |
