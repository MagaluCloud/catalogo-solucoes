# MGC - DBaaS / instâncias / Clusters  AutoDiscovery + GChat + AutoResize (Rundeck Job)

Este repositório (ou job YAML) contém um Job do Rundeck que:

- Faz **autodiscovery** das instâncias / Clusters DBaaS no tenant (via API `x-api-key` + `x-tenant-id`)
- Coleta métricas via endpoints expostos pela instância:
  - `http://<ip>:8080/node/metrics` (disco / filesystem)
  - `http://<ip>:8080/mysql/metrics` ou `http://<ip>:8080/postgres/metrics`
- Calcula:
  - **used/free/total** (GiB)
  - **percentual de uso** (%)
- Envia mensagens para um canal do **Google Chat** (webhook)
- Aplica política de decisão **OK/WARN/CRIT**
- Opcionalmente dispara um **Job de resize** do Rundeck (via API), com **cooldown** para evitar loop.

---

## ✅ Principais features

- **Sem login `mgc`**: usa API direta com `x-api-key` e `x-tenant-id`.
- **Detecção automática de região** caso `REGION` esteja vazio.
- **Detecção robusta do mountpoint**:
  - Prioriza `/mnt/database-data`
  - Se não existir, usa o maior filesystem (ignorando mounts de sistema e tmpfs)
- **Cálculo em `awk`** para evitar problemas com notação científica no Bash.
- **Mensagens multi-linha no GChat**, com `GCHAT >>` no log para auditoria.
- **Auto-resize opcional** (WARN/CRIT) via API do Rundeck.
- **Relatório JSON** persistido em `/var/log/rundeck` (ou diretório configurado).

---

## 📦 Dependências no host do Rundeck

O job roda no nó `localhost` do Rundeck (conforme `strategy: node-first`) e precisa:

- `mgc` (CLI)
- `jq`
- `curl`
- `awk`
- `sed`

Exemplo (Ubuntu/Debian):

```bash
sudo apt-get update
sudo apt-get install -y jq curl gawk sed
# mgc deve estar instalado e no PATH
```

---

## 🔐 Credenciais e segredos

### 1) MGC
O job recebe via options:

- `API_KEY` (secure)
- `TENANT_ID`

> Importante: o script **não imprime a API_KEY**, apenas o tamanho.

### 2) Google Chat Webhook
O job usa a option:

- `RD_WEBHOOK_URL` (obrigatória)

E também suporta (recomendado) guardar no **Key Storage** do Rundeck para manutenção futura.

### 3) Token da API do Rundeck (para auto-resize)
O script lê:

- `RD_TOKEN='${key:keys/project/iaas/rundeck/api_token}'`

Ou seja, você deve criar essa chave no Key Storage para permitir o disparo do resize via API.

---

## 🔑 Como criar as chaves no Key Storage (recomendado)

### Token de API do Rundeck (para disparo do resize)
Crie uma key no projeto `iaas`:

- Path: `keys/project/iaas/rundeck/api_token`
- Tipo: Password (ou texto)

Exemplo via UI do Rundeck:
- **Project Settings → Key Storage → Add Key**

> O token precisa ter permissão para `job:run` e `execution:read` no projeto.

### Webhook do Google Chat (opcional no Key Storage)
Se preferir mover o webhook para Key Storage, use:

- Path: `keys/project/iaas/gchat/webhook_url`

E no script (comentado no topo) você pode trocar para:

```bash
GCHAT_WEBHOOK_URL='${key:keys/project/iaas/gchat/webhook_url}'
```

---

## 🧠 Como funciona a política OK/WARN/CRIT

Você configura thresholds e aumentos:

- `THRESHOLD_WARN` (default 80)
- `THRESHOLD_CRIT` (default 90)
- `PCT_INCREASE_WARN` (default 30)
- `PCT_INCREASE_CRIT` (default 50)

Lógica:
- Se `used_pct >= THRESHOLD_CRIT` → `CRIT`
- Senão se `used_pct >= THRESHOLD_WARN` → `WARN`
- Senão → `OK`

Ações:
- `OK`: manda mensagem “nenhuma ação necessária”.
- `WARN/CRIT`: manda alerta e, se `AUTO_APPLY=1`, dispara job de resize.

---

## ⏳ Cooldown do auto-resize

Para evitar resize repetido em loop, existe `COOLDOWN_MINUTES` (default 360).

O job cria um state em:

```text
/var/lib/rundeck/var/dbaas-resize-cooldown/<INSTANCE_ID>.stamp
```

Se estiver dentro do período, ele:
- Não dispara resize
- Manda mensagem de cooldown no GChat

---

## 🗂 Saída e relatório

Para cada execução:

- O log mostra o sumário por instância:
  - engine, ip, disk used/free/total/percent
  - node_metrics/app_metrics status

- O relatório final é salvo em:

```text
/var/log/rundeck/dbaas-metrics-YYYYMMDD-HHMMSS.json
```

O JSON é um array com objetos por instância (com métricas, decisão e status).

---

## 🧪 Como testar

### 1) Rodar em modo debug
Execute o job com:

- `SHOW_SAMPLE=1`

Isso imprime:
- `mp` detectado
- valores `size/avail`
- `used_pct`

### 2) Validar endpoints manualmente (no host Rundeck)
Pegue o IP privado retornado e teste:

```bash
curl -s http://<IP_PRIVADO>:8080/node/metrics | head
curl -s http://<IP_PRIVADO>:8080/mysql/metrics | head
curl -s http://<IP_PRIVADO>:8080/postgres/metrics | head
```

---

## 🔧 Integração com Job de Resize

Você deve informar:

- `RESIZE_JOB_UUID` (UUID do job de resize existente)
- `RD_URL` (default `http://localhost:4440`)
- `RD_API_VERSION` (default `52`)

### Payload enviado (por padrão)
O script envia as options:

- `API_KEY`
- `REGION`
- `INSTANCE_ID`
- `INSTANCE_NAME`
- `PERCENTUAL`

> Se o seu job de resize usa nomes diferentes, ajuste a função `rundeck_run_resize_job()`.

---

## 🧾 Exemplo de mensagem no GChat (OK)

```text
✅ [OK] DBaaS
🎲 Banco => 'wiladmin' | (postgres)
id=3fe08313-9e96-4a1d-9ffc-f4008d3bdccf
💾 Uso => 0.08% (0.06GiB/73.65GiB) | mountpoint '/mnt/database-data'
⌚️Última checagem: 2026-02-03 21:27:36 UTC.
🚀 Nenhuma ação necessária.
```

---

## ✅ Boas práticas

- Mantenha `AUTO_APPLY=0` em produção até validar thresholds.
- Use `COOLDOWN_MINUTES` alto para evitar resize em cascata.
- Guarde webhook e token no Key Storage.
- Se faltar `used_pct`:
  - rode com `SHOW_SAMPLE=1`
  - valide se o mountpoint detectado aparece no `node/metrics`

---

## 📌 Troubleshooting

### Não envia no GChat
- Verifique `RD_WEBHOOK_URL` preenchido
- Veja no log se aparece:
  - `GCHAT: webhook carregado...`
  - `GCHAT << resposta: {...}`

Teste webhook manual:

```bash
curl -X POST -H 'Content-Type: application/json'   '<WEBHOOK_URL>'   -d '{"text":"Teste de conexão: Webhook funcionando! 🚀"}'
```

### `Result code was 141 (SIGPIPE)`
Esse erro acontece quando há pipe quebrado (ex: `| head`) em pipeline com `set -o pipefail`.
O script atual evita isso (sem `head`/`grep` em pipes críticos).

### Percentual vazio (DISK_USED_PCT="")
Normalmente é mountpoint não batendo com as labels do Prometheus.
Rode com `SHOW_SAMPLE=1` e confirme se existem linhas como:

- `node_filesystem_size_bytes{...,mountpoint="/mnt/database-data",...}`
- `node_filesystem_avail_bytes{...,mountpoint="/mnt/database-data",...}`

---

## 📄 Licença
Uso interno / operacional.
