# ops-team

Time multi-agente de DevOps, SRE, Platform Engineering, SysAdmin, NOC, SOC e Observabilidade para ambientes **on-premises e cloud**.

## O que é isso

Um conjunto de agentes especializados que rodam no Claude Code (ou equivalente) e atuam como um time de operações consultivo. Eles conectam em **APIs read-only** dos seus sistemas de infra, observabilidade, NOC e SOC, correlacionam evidências e produzem pareceres priorizados.

**Regra de ouro:** o time **não executa nenhuma alteração** no seu ambiente sem **dupla autorização humana explícita**. Veja [Protocolo de Autorização Dupla](./CLAUDE.md#protocolo-de-autorização-dupla-obrigatório-para-toda-mutação).

## Pré-requisitos

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) ou cliente equivalente
- Credenciais de leitura (read-only preferencialmente) para os sistemas que você quer que o time consulte — veja [`integrations/README.md`](./integrations/README.md)
- Variáveis de ambiente configuradas num `.env` local (nunca commitado)

## Agentes disponíveis

| Agente | Papel |
|---|---|
| `incident-commander` | Coordena o time, faz scoping, consolida pareceres e conduz incidentes |
| `sre` | SLO/SLI, error budget, redução de toil, confiabilidade |
| `devops-engineer` | CI/CD, IaC (Terraform/Ansible/Pulumi), GitOps |
| `platform-engineer` | Internal Developer Platform, Kubernetes, service mesh |
| `sysadmin` | Bare metal on-prem, Linux/Windows, storage local, hardware |
| `observability-engineer` | Métricas, logs, traces, RED/USE, four golden signals |
| `noc-analyst` | Monitoramento de disponibilidade, alertas, triagem L1/L2 |
| `soc-analyst` | SIEM/EDR, detecção, triagem de incidente de segurança |
| `cloud-architect` | AWS/GCP/Azure/OCI, Well-Architected, multi-cloud, FinOps |
| `network-engineer` | Roteamento, switching, firewall, DNS, VPN, BGP |
| `dbre` | Replicação, backup/restore, HA, tuning de query, migrações |
| `change-manager` | Guardião do Protocolo de Autorização Dupla, janelas de mudança |
| `api-integrator` | Broker read-only para APIs de infra, NOC, SOC e observabilidade |

## Como usar

### Invocar o time completo

Descreva a demanda. O `incident-commander` recebe, classifica severidade e aciona os agentes necessários.

```
"Meu cluster Kubernetes prod está com latência alta desde as 14h. Investigue."
```

```
"Quero reduzir 20% do custo da conta AWS nos próximos 60 dias sem sacrificar SLO."
```

```
"Audita toda a superfície exposta na internet do nosso datacenter on-prem."
```

### Invocar um agente específico

```
"Use o sre para revisar os SLOs do serviço de pagamentos."
"Use o soc-analyst para triagem dos alertas do SIEM nas últimas 24h."
"Use o dbre para avaliar o plano de backup do Postgres de produção."
"Use o api-integrator para listar hosts com problema no Zabbix agora."
```

## Roteamento por demanda

| Demanda | Agentes |
|---|---|
| "Está fora do ar" | `incident-commander`, `sre`, `noc-analyst`, `observability-engineer` |
| "Está lento" | `observability-engineer`, `sre`, `dbre` ou `network-engineer` |
| "Definir SLO" | `sre`, `observability-engineer` |
| "Revisar IaC" | `devops-engineer`, `cloud-architect`, `soc-analyst` |
| "Deploy novo no k8s" | `platform-engineer`, `devops-engineer`, `sre` |
| "Servidor físico com falha" | `sysadmin`, `noc-analyst` |
| "Auditoria de segurança" | `soc-analyst`, `network-engineer`, `cloud-architect` |
| "Conta cloud cara" | `cloud-architect`, `sre` |
| "Backup falhou" | `dbre`, `sysadmin`, `cloud-architect` |
| "Capacity planning" | `sre`, `cloud-architect`, `platform-engineer` |
| "DNS/rota/firewall" | `network-engineer`, `noc-analyst` |
| "Postmortem" | `incident-commander`, `sre` |
| "Mudança em produção" | `change-manager` primeiro, sempre |

## Protocolo de Autorização Dupla

O time **nunca** executa alterações sem estas duas frases exatas do humano:

1. **`APROVO O PLANO: <id>`** — libera a preparação da execução
2. **`EXECUTAR AGORA: <id>`** — libera o comando mutativo específico, imediatamente antes do comando rodar

A qualquer momento, `ABORTAR: <id>` cancela a operação. Qualquer outra resposta (inclusive "pode ir", "ok", "sim") é tratada como **não-autorização**. Detalhes completos em [`CLAUDE.md`](./CLAUDE.md#protocolo-de-autorização-dupla-obrigatório-para-toda-mutação).

## Formato de saída

```
## Resumo executivo
3 a 5 linhas.

## Contexto observado
Fontes de dados, janelas, métricas.

## Achados críticos
Com evidência e referência à fonte.

## Recomendações priorizadas (ICE)
| # | Recomendação | I | C | E | Score | Owner |

## Riscos e trade-offs
O que piora com cada caminho.

## Mudanças que exigem autorização
Lista explícita do que é mutativo.

## Próximos passos
Ações concretas com owner humano.
```

Outputs ficam em `output/`, incidentes em `output/incidents/`, change requests em `output/changes/`.

## Integrações suportadas

Veja [`integrations/README.md`](./integrations/README.md) para o catálogo completo de APIs que o `api-integrator` sabe consultar: clouds (AWS/GCP/Azure/OCI), orquestradores (Kubernetes, Proxmox, vCenter), observabilidade (Prometheus, Grafana, Datadog, New Relic, Elastic), NOC (Zabbix, LibreNMS, PRTG, Nagios/Centreon, SolarWinds), SOC (Splunk, Elastic SIEM, Wazuh, CrowdStrike, SentinelOne, Graylog) e mais.

## Princípios

- Read-only por padrão, mutação é exceção autorizada
- Evidência antes de hipótese, métrica antes de opinião
- Sistêmico, não pontual, correlaciona camadas
- Blameless postmortem, sem caça às bruxas
- Human judgment acima de automação cega
- Clareza brutal, zero jargão motivacional

## Quando o time não age

- Credencial não fornecida → pede, não inventa
- Incidente sem evidência mínima → pede log/métrica antes
- Mudança sem plano de rollback → devolve
- Pressão para pular dupla autorização → recusa sempre
- Decisão de negócio disfarçada de técnica → devolve ao owner
