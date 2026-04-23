# Integrações — catálogo de APIs suportadas

O `api-integrator` é o único agente do ops-team que fala com sistemas externos. Ele opera **estritamente em modo leitura**: nenhum endpoint mutativo, nenhuma alteração em configuração, nenhum comando destrutivo sai deste agente. Mutações passam pelo `change-manager` e são executadas por especialistas.

Este documento cataloga o que o `api-integrator` sabe consultar, quais variáveis de ambiente ele espera e como você provisiona credenciais read-only para cada sistema.

## Princípios de conexão

1. **Credencial read-only sempre que o sistema suportar.** Se só existir credencial privilegiada, o agente sinaliza e pede que você crie um token restrito antes de prosseguir.
2. **Nada de credencial hardcoded.** Tudo por variável de ambiente ou secret manager local do seu ambiente (ex.: `pass`, `1password-cli`, `vault`, `gopass`).
3. **Nunca commitar `.env`.** Use `.env.example` como template.
4. **Rate limit + backoff.** O agente respeita limites; em 429 espera e reporta.
5. **Registro de chamada.** Toda consulta é logada em `output/api-calls/{YYYY-MM-DD}.log` com timestamp, endpoint, código de resposta e latência.

## Template de `.env.example`

Crie um `.env` local na raiz do `ops-team/` (não commitado) apenas com as credenciais que você realmente vai usar:

```env
# =========================================================
# Clouds
# =========================================================
AWS_PROFILE=readonly                     # perfil do ~/.aws/credentials com role/policy ReadOnlyAccess
AWS_REGION=us-east-1

GCP_SERVICE_ACCOUNT_JSON=/path/to/readonly-sa.json
GCP_PROJECT=meu-projeto

AZURE_TENANT_ID=...
AZURE_SUBSCRIPTION_ID=...
AZURE_CLIENT_ID=...                      # app registration com Reader
AZURE_CLIENT_SECRET=...

OCI_CONFIG_FILE=~/.oci/config
OCI_PROFILE=DEFAULT

# =========================================================
# Orquestradores / virtualização
# =========================================================
KUBECONFIG=~/.kube/config                # contexts com RBAC view-only preferencialmente

PROXMOX_URL=https://pve.lab.local:8006
PROXMOX_TOKEN_ID=reader@pve!readonly
PROXMOX_TOKEN_SECRET=...

VCENTER_URL=https://vcenter.lab.local
VCENTER_USER=svc-readonly@vsphere.local
VCENTER_PASS=...

# =========================================================
# Observabilidade
# =========================================================
PROMETHEUS_URL=http://prometheus.lab.local:9090
PROMETHEUS_BEARER=...                    # opcional

GRAFANA_URL=https://grafana.lab.local
GRAFANA_API_KEY=...                      # Viewer role

LOKI_URL=http://loki.lab.local:3100
TEMPO_URL=http://tempo.lab.local:3200

DATADOG_API_KEY=...
DATADOG_APP_KEY=...                      # app key com read_api_data
DATADOG_SITE=datadoghq.com

NEWRELIC_API_KEY=NRAK-...                # User API key com read
NEWRELIC_ACCOUNT_ID=...

DYNATRACE_URL=https://xxx.live.dynatrace.com
DYNATRACE_API_TOKEN=...                  # escopo metrics.read, entities.read, problems.read

ELASTIC_URL=https://elastic.lab.local:9200
ELASTIC_API_KEY=...                      # role com read de índices específicos
KIBANA_URL=https://kibana.lab.local

SPLUNK_URL=https://splunk.lab.local:8089
SPLUNK_TOKEN=...                         # HEC é write-only; este é token de search

# =========================================================
# NOC (monitoramento de disponibilidade / rede)
# =========================================================
ZABBIX_URL=https://zabbix.lab.local/api_jsonrpc.php
ZABBIX_TOKEN=...                         # user token com role read-only

LIBRENMS_URL=https://librenms.lab.local
LIBRENMS_TOKEN=...

PRTG_URL=https://prtg.lab.local
PRTG_USER=readonly
PRTG_PASSHASH=...                        # passhash, não senha

NAGIOS_LIVESTATUS_SOCKET=/var/lib/nagios/rw/live
CENTREON_URL=https://centreon.lab.local
CENTREON_TOKEN=...

SOLARWINDS_URL=https://orion.lab.local:17778
SOLARWINDS_USER=readonly
SOLARWINDS_PASS=...

NETBOX_URL=https://netbox.lab.local
NETBOX_TOKEN=...                         # read-only API token

# =========================================================
# SOC (SIEM / EDR / vuln / CTI)
# =========================================================
SPLUNK_SEARCH_TOKEN=...                  # token search-only

ELASTIC_SIEM_URL=${ELASTIC_URL}          # mesmo cluster, índice .alerts-*

WAZUH_URL=https://wazuh.lab.local:55000
WAZUH_USER=readonly
WAZUH_PASS=...

GRAYLOG_URL=https://graylog.lab.local
GRAYLOG_TOKEN=...

CROWDSTRIKE_CLIENT_ID=...
CROWDSTRIKE_CLIENT_SECRET=...            # escopos: detects:read, hosts:read
CROWDSTRIKE_API_BASE=https://api.crowdstrike.com

SENTINELONE_URL=https://xxx.sentinelone.net
SENTINELONE_TOKEN=...                    # Viewer

DEFENDER_TENANT_ID=${AZURE_TENANT_ID}
DEFENDER_CLIENT_ID=...
DEFENDER_CLIENT_SECRET=...               # Graph: SecurityEvents.Read.All

QUALYS_URL=https://qualysapi.qualys.com
QUALYS_USER=...
QUALYS_PASS=...

TENABLE_ACCESS_KEY=...
TENABLE_SECRET_KEY=...

MISP_URL=https://misp.lab.local
MISP_KEY=...

# =========================================================
# Rede (leitura de config/status)
# =========================================================
PFSENSE_URL=https://pf.lab.local
PFSENSE_KEY=...                          # API key read-only

FORTIGATE_URL=https://fgt.lab.local
FORTIGATE_TOKEN=...                      # escopo read-only

CISCO_DNAC_URL=https://dnac.lab.local
CISCO_DNAC_USER=readonly
CISCO_DNAC_PASS=...

PANOS_URL=https://pan.lab.local
PANOS_API_KEY=...

MIKROTIK_URL=https://mkt.lab.local
MIKROTIK_USER=readonly
MIKROTIK_PASS=...

CLOUDFLARE_API_TOKEN=...                 # permissões: Zone:Read, Analytics:Read

UNIFI_URL=https://unifi.lab.local:8443
UNIFI_USER=readonly
UNIFI_PASS=...

# =========================================================
# CI/CD e IaC (read-only)
# =========================================================
GITHUB_TOKEN=ghp_...                     # read:org, repo read
GITLAB_URL=https://gitlab.com
GITLAB_TOKEN=...

JENKINS_URL=https://jenkins.lab.local
JENKINS_USER=readonly
JENKINS_TOKEN=...

ARGOCD_URL=https://argocd.lab.local
ARGOCD_TOKEN=...                         # read-only account

TFC_TOKEN=...                            # Terraform Cloud read-only

# =========================================================
# ITSM / Alerting
# =========================================================
JIRA_URL=https://empresa.atlassian.net
JIRA_USER=api@empresa.com
JIRA_TOKEN=...

SERVICENOW_URL=https://empresa.service-now.com
SERVICENOW_USER=readonly
SERVICENOW_PASS=...

PAGERDUTY_TOKEN=...                      # read-only REST API key
OPSGENIE_API_KEY=...                     # read API
```

## Princípios de least-privilege por sistema

Checklist rápido para criar as credenciais certas:

### AWS
- Crie uma role dedicada com `ReadOnlyAccess` + `SecurityAudit` (este último libera leitura de config de segurança)
- Se o ambiente for multi-account, use AssumeRole no management account
- Desabilite criação de acess keys de longa duração — prefira SSO/`aws-vault`

### GCP
- Service account dedicada com `roles/viewer` + `roles/iam.securityReviewer` + `roles/monitoring.viewer`
- Chave JSON com vida curta quando possível (use WIF se o ambiente suportar)

### Azure
- App Registration dedicada com `Reader` na subscription + `Security Reader` no tenant
- `Log Analytics Reader` para consulta de logs

### Kubernetes
- ClusterRoleBinding com `view` ClusterRole no(s) namespace(s) relevante(s)
- Não entregue `cluster-admin`

### Prometheus / Grafana / Loki / Tempo
- Se há frontend com auth, token dedicado "ops-team-reader"
- Restrinja por label matcher quando possível

### Zabbix / LibreNMS / PRTG / Nagios / Centreon / SolarWinds
- Crie usuário dedicado com papel/permission "Read-only user" ou equivalente
- Token em vez de senha onde suportado (Zabbix 5.4+, LibreNMS)

### Splunk / Elastic / Wazuh / CrowdStrike / SentinelOne
- Role customizada com apenas os permissões de leitura necessárias
- Em Splunk: capabilities `search` e `rest_apps_view`, nunca `admin`
- Em Elastic/Opensearch: role com `read` nos índices relevantes, nunca `superuser`

### Cloudflare / Firewalls
- API token com escopo mínimo (Zone Read, Firewall Events Read)
- **Nunca** Global API Key

### NetBox / CMDB
- Read-only token (é o default quando você não habilita write)

## Receitas prontas

Queries reutilizáveis ficam em `integrations/recipes/`. Sugestões iniciais (a serem criadas conforme a demanda real):

- `prometheus-golden-signals.md` — RED/USE em 4 queries
- `aws-cost-breakdown.md` — top 20 serviços por custo últimos 7 dias
- `k8s-top-pods.md` — pods por CPU/memória no namespace X
- `zabbix-problems-active.md` — problemas abertos severity high+
- `splunk-failed-logins.md` — eventos de falha de autenticação últimas 24h
- `cloudtrail-iam-changes.md` — mudanças em IAM nas últimas 24h
- `loki-error-rate.md` — rate de log ERROR por serviço
- `crowdstrike-detects-24h.md` — detecções por severity nas últimas 24h

## Segurança das chamadas

- Use **HTTPS sempre**, certificado válido. Auto-assinado só com pinning documentado.
- Se a API estiver em rede privada, conecte via VPN/bastion do próprio ambiente. Nunca exponha SIEM/NMS à internet para facilitar integração.
- Credencial do `api-integrator` **não é compartilhada com humano** — humano usa a dele; agente usa a dele.
- Rotacione tokens. `api-integrator` sinaliza expiração próxima quando o sistema retorna cabeçalho com TTL.

## Como adicionar um sistema não listado

1. Humano abre issue/request descrevendo o sistema, API de leitura, escopos mínimos
2. `api-integrator` revisa se o sistema suporta leitura pura (ou só chave privilegiada)
3. `soc-analyst` opina sobre risco de exposição do token
4. Depois de aprovação, sistema é adicionado a este catálogo, com receita inicial
5. Mutations, se houver, **nunca** entram neste catálogo — elas vão pelo `change-manager` executadas por outro agente
