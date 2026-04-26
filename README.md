# ops-team

Time multi-agente de DevOps, SRE, Platform Engineering, SysAdmin, NOC, SOC e Observabilidade para ambientes **on-premises e cloud**.

**Regra de ouro:** o time **não executa nenhuma alteração** sem [dupla autorização humana explícita](#protocolo-de-autorização-dupla). Ele observa, mede, recomenda — e só age com sua ordem direta.

---

## Início rápido

**Pré-requisitos:**
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) instalado e configurado
- Credenciais de leitura para os sistemas que quer consultar (veja `.env.example`)

```sh
cp .env.example .env
# Preencha apenas os sistemas que você quer que o time consulte
```

Descreva o problema em linguagem natural — o `incident-commander` faz o scoping automaticamente:

```
"Meu cluster Kubernetes prod está com latência alta desde as 14h. Investigue."
```

```
"Quero reduzir 20% do custo da conta AWS sem sacrificar SLO."
```

```
"Audita a superfície exposta na internet do nosso datacenter on-prem."
```

Ou invoque um agente diretamente:

```
"Use o sre para revisar os SLOs do serviço de pagamentos."
"Use o soc-analyst para triagem dos alertas do SIEM nas últimas 24h."
"Use o dbre para avaliar o plano de backup do Postgres de produção."
"Use o api-integrator para listar hosts com problema no Zabbix agora."
```

---

## Agentes

13 agentes especializados em `agents/`:

| Agente | Papel | Modelo |
|---|---|---|
| [`incident-commander`](agents/incident-commander.md) | Coordena o time, classifica severidade (SEV1–4), conduz incidentes em tempo real, consolida pareceres | Opus |
| [`sre`](agents/sre.md) | SLO/SLI/error budget, redução de toil, postmortem blameless, capacity planning, DORA metrics | Opus |
| [`devops-engineer`](agents/devops-engineer.md) | CI/CD, IaC (Terraform/OpenTofu/Ansible/Pulumi), GitOps, supply chain (SLSA), drift detection | Sonnet |
| [`platform-engineer`](agents/platform-engineer.md) | Internal Developer Platform, Kubernetes em profundidade, service mesh, golden paths, developer experience | Sonnet |
| [`sysadmin`](agents/sysadmin.md) | Bare metal on-prem, Linux/Windows Server, storage (RAID/LVM/ZFS), hardware, firmware, datacenter | Sonnet |
| [`observability-engineer`](agents/observability-engineer.md) | Instrumentação (OpenTelemetry), dashboards, alertas acionáveis, RED/USE/Four Golden Signals | Sonnet |
| [`noc-analyst`](agents/noc-analyst.md) | Triagem de alertas de NMS (Zabbix/LibreNMS/PRTG/Nagios), correlação, escalação L1/L2 | Sonnet |
| [`soc-analyst`](agents/soc-analyst.md) | SIEM/EDR (Splunk/Wazuh/CrowdStrike), triagem, threat hunting (MITRE ATT&CK), IR defensivo | Sonnet |
| [`cloud-architect`](agents/cloud-architect.md) | AWS/GCP/Azure/OCI, Well-Architected (6 pilares), multi-cloud, FinOps, landing zone | Opus |
| [`network-engineer`](agents/network-engineer.md) | Roteamento BGP/OSPF, switching, firewall, DNS, VPN, captura de pacote L1–L7 | Sonnet |
| [`dbre`](agents/dbre.md) | Replicação, backup/restore/PITR, HA, query tuning, migração de schema, Postgres/MySQL/Mongo/Redis | Sonnet |
| [`change-manager`](agents/change-manager.md) | Guardião do Protocolo de Autorização Dupla, Change Requests, CAB virtual, janelas de mudança | Opus |
| [`api-integrator`](agents/api-integrator.md) | Broker read-only para clouds, orquestradores, observabilidade, NOC, SOC, CI/CD e ITSM | Sonnet |

---

## Roteamento por demanda

| Demanda | Agentes acionados |
|---|---|
| "Está fora do ar" | `incident-commander` → `sre`, `noc-analyst`, `observability-engineer` |
| "Está lento, mas no ar" | `observability-engineer`, `sre`, `dbre` (se banco) ou `network-engineer` (se rede) |
| "Definir SLO/SLI para o serviço X" | `sre`, `observability-engineer` |
| "Revisar Terraform/Ansible/pipeline" | `devops-engineer`, `cloud-architect`, `soc-analyst` |
| "Deploy novo no Kubernetes" | `platform-engineer`, `devops-engineer`, `sre` |
| "Servidor físico com falha" | `sysadmin`, `noc-analyst`, `observability-engineer` |
| "Auditoria de segurança da infra" | `soc-analyst`, `network-engineer`, `cloud-architect`, `sysadmin` |
| "Conta cloud cara demais" | `cloud-architect`, `sre` |
| "Backup falhou" | `dbre` (banco), `sysadmin` (filesystem), `cloud-architect` (snapshot cloud) |
| "Capacity planning Q2" | `sre`, `cloud-architect`, `platform-engineer` |
| "DNS/rota/firewall com problema" | `network-engineer`, `noc-analyst` |
| "Postmortem do incidente X" | `incident-commander`, `sre`, `soc-analyst` (se segurança) |
| "Mudança em produção" | `change-manager` **primeiro**, sempre — depois o especialista |
| "Coletar dado de Prometheus/Zabbix/Splunk" | `api-integrator` |

Demanda ambígua? O `incident-commander` faz o scoping antes de acionar o time.

---

## Protocolo de Autorização Dupla

Toda mutação passa por dois gates obrigatórios, nesta ordem:

```
1.  APROVO O PLANO: CR-{id}     ← aprova o Change Request; libera preparação
2.  EXECUTAR AGORA: CR-{id}     ← autoriza o comando literal; libera execução
```

`ABORTAR: CR-{id}` cancela imediatamente em qualquer etapa.

**Qualquer outra resposta** ("pode ir", "ok", "sim", emoji) é tratada como **não-autorização** — o protocolo repete o pedido. Detalhes completos e categorias de risco em [`CLAUDE.md`](./CLAUDE.md#protocolo-de-autorização-dupla-obrigatório-para-toda-mutação).

---

## Formato de saída

Todo parecer segue esta estrutura:

```
## Resumo executivo
3 a 5 linhas, o essencial.

## Contexto observado
Fontes de dados, janelas de tempo, métricas consultadas.

## Achados críticos
Lista numerada com evidência técnica e referência à fonte. Sem palpite.

## Recomendações priorizadas (ICE)
| # | Recomendação | Impacto | Confiança | Esforço | Score | Owner |
Score = (Impacto × Confiança) / Esforço — ordenado do maior para o menor.

## Riscos e trade-offs
O que piora em cada caminho.

## Mudanças que exigem autorização
Lista explícita do que é mutativo, com risco e janela sugerida.

## Próximos passos
Ações concretas com owner humano e prazo.
```

Arquivos gerados automaticamente em:

| Tipo | Caminho |
|---|---|
| Parecer operacional | `output/{ambiente}-{YYYY-MM-DD}.md` |
| Incidente ativo | `output/incidents/{YYYY-MM-DD}-{slug}.md` |
| Change Request | `output/changes/CR-{id}.md` |
| Audit trail de API | `output/api-calls/{YYYY-MM-DD}.log` |
| Registro de padrões | `output/changes/patterns.md` |

---

## Integrações suportadas

O `api-integrator` conecta (read-only) em:

| Categoria | Sistemas |
|---|---|
| **Cloud** | AWS, GCP, Azure, OCI |
| **Orquestradores** | Kubernetes, Proxmox, VMware vCenter, Nutanix, OpenStack |
| **Observabilidade** | Prometheus, Grafana, Loki, Tempo/Jaeger, Datadog, New Relic, Dynatrace, Elastic/OpenSearch, InfluxDB |
| **NOC** | Zabbix, LibreNMS, PRTG, Nagios/Centreon/Icinga, SolarWinds, NetBox |
| **SOC** | Splunk, Elastic SIEM, Wazuh, Graylog, CrowdStrike, SentinelOne, Microsoft Defender, Qualys, Tenable, MISP |
| **Rede** | pfSense/OPNsense, FortiGate, Cisco DNA-C, Palo Alto PAN-OS, MikroTik, Unifi, Cloudflare |
| **CI/CD** | GitHub, GitLab, Bitbucket, Jenkins, ArgoCD, Flux, Terraform Cloud |
| **ITSM** | Jira, ServiceNow, PagerDuty, Opsgenie, Grafana OnCall |

Configure credenciais em `.env` (use `.env.example` como base). Catálogo completo e política de least-privilege em [`integrations/README.md`](./integrations/README.md).

---

## Princípios

| # | Princípio | Na prática |
|---|---|---|
| 1 | **Read-only por padrão** | Mutação é exceção autorizada, nunca regra |
| 2 | **Evidência antes de hipótese** | Sem dado real, não há diagnóstico |
| 3 | **Pensamento sistêmico** | Uma falha raramente é pontual; rede, compute, app e pessoas formam o mesmo sistema |
| 4 | **Blameless postmortem** | Foco em falha de sistema, não em culpa individual |
| 5 | **Toil é dívida operacional** | Trabalho repetitivo é candidato a automação — após autorização |
| 6 | **Clareza brutal** | Sem jargão motivacional; sem "talvez" quando o log pode responder |
| 7 | **Human judgment acima de automação** | O time propõe, nunca impõe |

---

## Quando o time não age

- **Credencial não fornecida** → pede explicitamente ao humano, não tenta adivinhar
- **Incidente sem evidência mínima** → solicita log ou métrica antes de especular
- **Mudança sem plano de rollback** → devolve ao requisitante
- **Pressão para pular a dupla autorização** → recusa educadamente, sempre
- **Decisão de negócio disfarçada de técnica** → devolve ao owner de negócio
