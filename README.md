# Junos Auto-Failover: RPM & Event-Options 🚀

![Juniper Networks](https://img.shields.io/badge/Vendor-Juniper_Networks-006752?style=for-the-badge&logo=junipernetworks)
![Junos OS](https://img.shields.io/badge/OS-Junos_12.1%2B-blue?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)
![Automation](https://img.shields.io/badge/Automation-Event--Options-orange?style=for-the-badge)

Este repositório contém uma solução de automação nativa para **Junos OS** que implementa o failover automático de rotas estáticas baseado na saúde do link, monitorado via **RPM (Real-time Performance Monitoring)**.

## 📋 Cenário de Aplicação
Em ambientes de ISP ou Datacenters, nem sempre o próximo salto (Next-Hop) está diretamente conectado, o que impede que a interface "caia" fisicamente em caso de falha no transporte. Esta configuração garante que, se o tráfego ICMP parar de responder através do link principal, a rota seja removida automaticamente.

## 🛠️ Detalhamento dos Componentes

### 1. Roteamento Estático
Utilizamos o parâmetro `qualified-next-hop` para permitir que o mesmo prefixo tenha múltiplos próximos saltos com métricas (preferências) diferentes.
- **Primário:** Preferência 5.
- **Backup:** Preferência 20.

### 2. Monitoramento (RPM)
O Probe ICMP verifica a disponibilidade do Gateway:
| Parâmetro | Valor | Descrição |
| :--- | :--- | :--- |
| `probe-count` | 3 | Quantidade de pacotes por teste. |
| `probe-interval` | 2s | Intervalo entre os pacotes. |
| `successive-loss` | 3 | Quantidade de falhas consecutivas para declarar o teste como "FAILED". |

### 3. Política de Eventos (Event-Options)
A inteligência da rede que vincula o monitoramento à ação:
- **FAIL_CLIENTE:** Disparado pelo evento `rpm_test_failed`. Executa o `deactivate` na rota principal, forçando o tráfego para o backup.
- **RESTORE_CLIENTE:** Disparado pelo evento `rpm_test_completed`. Executa o `activate` para restaurar a prioridade do link principal quando o serviço estabiliza.

## 🚀 Como Implementar

```junos
# Definição das Rotas
set routing-options static route 1.1.1.1/32 qualified-next-hop 2.2.2.2 preference 5
set routing-options static route 1.1.1.1/32 qualified-next-hop 3.3.3.3 preference 20

# Configuração do Probe
set services rpm probe MONITOR_CLIENTE test ICMP_CHECK target address 2.2.2.2
set services rpm probe MONITOR_CLIENTE test ICMP_CHECK probe-type icmp-ping
set services rpm probe MONITOR_CLIENTE test ICMP_CHECK probe-count 3
set services rpm probe MONITOR_CLIENTE test ICMP_CHECK probe-interval 2
set services rpm probe MONITOR_CLIENTE test ICMP_CHECK thresholds successive-loss 3

# Automação de Failover
set event-options policy FAIL_CLIENTE events rpm_test_failed
set event-options policy FAIL_CLIENTE attributes-match rpm_test_failed.test-name matches ICMP_CHECK
set event-options policy FAIL_CLIENTE then execute-commands commands "deactivate routing-options static route 1.1.1.1/32 qualified-next-hop 2.2.2.2"
set event-options policy FAIL_CLIENTE then execute-commands commands "commit comment 'FAILOVER_CLIENTE_RPM'"

# Automação de Restauração
set event-options policy RESTORE_CLIENTE events rpm_test_completed
set event-options policy RESTORE_CLIENTE attributes-match rpm_test_completed.test-name matches ICMP_CHECK
set event-options policy RESTORE_CLIENTE then execute-commands commands "activate routing-options static route 1.1.1.1/32 qualified-next-hop 2.2.2.2"
set event-options policy RESTORE_CLIENTE then execute-commands commands "commit comment 'RESTORE_CLIENTE_RPM'"
