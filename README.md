# Projeto: Implementação de Firewall pfSense Seguro com IDS/IPS no Proxmox

## 🎯 Objetivo

Este projeto tem como objetivo implementar uma infraestrutura de rede segura utilizando o firewall pfSense virtualizado no Proxmox. A configuração visa proteger uma rede local segmentada em duas sub-redes (servidores e clientes) através de políticas de segurança bem definidas e um sistema de detecção e prevenção de intrusão (IDS/IPS) com Suricata.

## 🛡️ Arquitetura da Rede

A topologia da rede implementada é a seguinte:

```
Internet (Provedor)
    ↓
192.168.0.0/24 (vmbr0 - WAN)
    ↓
[pfSense Firewall]
    ├─→ 192.168.1.0/24 (vmbr1 - LAN Servidores)
    └─→ 192.168.2.0/24 (vmbr2 - LAN Clientes)
```

### Endereçamento IP

| Interface    | Bridge | IP do pfSense | Rede             | Gateway        | Função     |
| :----------- | :----- | :------------- | :--------------- | :------------- | :--------- |
| WAN          | vmbr0  | 192.168.0.100  | 192.168.0.0/24   | 192.168.0.1  | Internet   |
| LAN          | vmbr1  | 192.168.1.1    | 192.168.1.0/24   | -              | Servidores |
| LAN_DESKTOPS | vmbr2  | 192.168.2.1    | 192.168.2.0/24   | -              | Clientes   |

## 🔒 Políticas de Segurança

Os seguintes princípios de segurança são aplicados:

-   **Menor Privilégio**: Liberar apenas o acesso necessário para cada componente da rede.
-   **Segmentação**: Isolar servidores de clientes para limitar o impacto de possíveis ataques.
-   **Defesa em Profundidade**: Implementar múltiplas camadas de segurança para aumentar a proteção.
-   **Log e Auditoria**: Registrar tentativas de acesso e eventos relevantes para análise e auditoria.

### Matriz de Acesso

| Origem → Destino       | Internet                      | pfSense                 | Servidores (LAN)          | Clientes (Desktops)   |
| :--------------------- | :---------------------------- | :---------------------- | :------------------------ | :-------------------- |
| **Servidores**         | HTTP/HTTPS, DNS, NTP, ICMP   | DNS, WebGUI             | SSH, RDP                  | ❌ Bloqueado          |
| **Clientes**           | HTTP/HTTPS, DNS, ICMP         | DNS, WebGUI             | HTTP/HTTPS, SSH, RDP      | Entre si OK           |
| **Internet**           | -                             | Apenas admin (192.168.0.0/24) | ❌ Bloqueado            | ❌ Bloqueado          |

## 🔥 Firewall pfSense e IDS/IPS Suricata

O pfSense é configurado com regras de firewall para controlar o tráfego entre as redes e a Internet, seguindo a matriz de acesso definida. Adicionalmente, o Suricata é implementado como um sistema de detecção e prevenção de intrusão (IDS/IPS) para monitorar o tráfego em tempo real, detectar ameaças e, opcionalmente, bloquear automaticamente o tráfego malicioso.

### Aliases

Aliases são utilizados para facilitar a gestão e manutenção das regras do firewall. Os seguintes aliases são criados:

-   Rede\_Servidores (192.168.1.0/24)
-   Rede\_Clientes (192.168.2.0/24)
-   Portas\_Web (80, 443)
-   Portas\_Admin (22, 3389)
-   Portas\_Servicos\_LAN (80, 443, 22, 3389)


Este README fornece uma visão geral do projeto e suas configurações de segurança. Para detalhes específicos de configuração, consulte a documentação detalhada do pfSense e Suricata.