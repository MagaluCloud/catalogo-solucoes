# Rundeck Jobs – Monitoring

Este diretório contém os **Jobs do Rundeck** usados no projeto `monitoring` (observabilidade) para automação na **Magalu Cloud (MGC)**.

Gerado em **2026-02-13** a partir dos YAMLs enviados.

---

## Conteúdo

- Jobs de ciclo de vida das VMs `app-*` (restaura snapshot, adiciona/remover targets)
- Load Balancer (criação e atualização de backends/targets)
- Atualização de targets do Prometheus via `file_sd_configs`
- Instalação de componentes (Prometheus/Node Exporter e Elasticsearch)
- Testes de performance (grupo `performance`)

---

## Pré-requisitos

### Na máquina onde o Job roda (tipicamente o servidor do Rundeck)
- `bash`
- `mgc` CLI
- `jq`
- Permissão de `sudo` (para jobs que escrevem em `/etc/prometheus` e reiniciam serviços)
- Prometheus instalado quando o job atualiza `/etc/prometheus/hosts_app.yml`

### Para jobs com SSH
- Conectividade de rede até as VMs alvo (IP privado)
- Chave SSH válida com permissão no usuário alvo (ex.: `ubuntu`) e `sudo` (se necessário)

### Key Storage (recomendado)
A maioria dos jobs já aponta para:
- `keys/project/monitoring/MGCAPIKEY`

---

## Importando os jobs

### Via Rundeck UI
1. Jobs → Upload Definition
2. Selecione o arquivo `.yaml`
3. Confirme e salve

### Via Rundeck CLI (`rd`)
Exemplo:
```bash
rd jobs load -p monitoring -f ./monitoring/jobs/<arquivo>.yaml
```

---

## Catálogo rápido

| Grupo | Job | Arquivo | Options | ID |
|---|---|---|---:|---|
| `performance` | perf-run-on-node | `perf-run-on-node.yaml` | 13 | `903d392e-398e-46e1-922d-17cc26e2ee75` |
| `root` | Adiciona VM ao  Pool do Load Balancer | `adiciona-vm-ao-pool-do-load-balancer.yaml` | 1 | `cupdate-lb-targets` |
| `root` | Apaga as instancias de aplicação  | `apaga-as-instancias-de-aplica-o.yaml` | 1 | `delete-targets` |
| `root` | Atualiza nods do grafana  | `atualiza-nods-do-grafana.yaml` | 1 | `atualiza-nodes-grafana` |
| `root` | Criar Load Balancer - CPEN5-SE | `criar-load-balancer-cpen5-se.yaml` | 1 | `create-lb-cpen5-se` |
| `root` | Instala prometheus  | `instala-prometheus.yaml` | 2 | `a3b5cfa6-fc27-4006-bc2e-5a2b5fed381c` |
| `root` | Install Elasticsearch | `install-elasticsearch.yaml` | 0 | `8e0c9550-e357-47f1-b0a5-a04ccf43334f` |
| `root` | Remove-instances-and-update-grafana | `remove-instances-and-update-grafana.yaml` | 1 | `remove-instances-and-update-grafana` |
| `root` | Restaura snapshot  | `restaura-snapshot.yaml` | 1 | `restaura-targets` |
| `root` | create node and add in load balancer and monitoring stack | `create-node-and-add-in-load-balancer-and-monitoring-stack.yaml` | 1 | `add-nodes-in-app-pool` |


---

## Notas importantes (antes de rodar em produção)

- Alguns jobs fazem ações destrutivas (**delete de VMs**) e/ou **replace completo** de targets em Load Balancer.
- Alguns scripts possuem valores fixos (ex.: `LB_NAME`, `MACHINE_TYPE_ID`) que devem ser ajustados ao seu ambiente.
- Foi identificado pelo menos um job com **API Key hardcoded** dentro do script. Remova isso imediatamente e use somente Key Storage/options.

---

## Jobs (detalhamento)

## perf-run-on-node

**Arquivo:** `perf-run-on-node.yaml`  
**ID/UUID:** `903d392e-398e-46e1-922d-17cc26e2ee75`  
**Grupo:** `performance`

Executa testes (cpu|mem|disk|net|all) em UMA VM cujo nome você informar. Marca janela no Prometheus via textfile collector.

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `NODE_NAME` | ✅ | — | `` | Nome exato do nó/VM onde o job deve rodar |
| `MODE` | ✅ | — | `all` | Tipo de teste |
| `DURATION` | ✅ | — | `60` | Duração (s) |
| `CPU_WORKERS` | — | — | `` | Workers de CPU (padrão: nproc) |
| `MEM_PCT` | — | — | `80` | % de memória para stress-ng (ex.: 80) |
| `FIO_FILE` | — | — | `/tmp/fio_testfile` | Arquivo/dispositivo para fio (ex.: /tmp/fio_testfile ou /dev/nvme0n1 -> cuidado!) |
| `FIO_SIZE` | — | — | `2G` | Tamanho do arquivo (ex.: 2G) |
| `FIO_IODEPTH` | — | — | `32` | Profundidade de fila |
| `FIO_BS` | — | — | `4k` | Block size |
| `FIO_RW` | — | — | `randrw` | Padrão de I/O |
| `NET_SERVER` | — | — | `` | Servidor iperf3 (IP/host) quando MODE=net ou no all |
| `NET_PARALLEL` | — | — | `4` | Conexões paralelas do iperf3 (-P) |
| `SETUP_IPERF_SERVER` | — | — | `false` | Se true, inicia iperf3 -s neste nó antes do teste |

**Notas e cuidados**

- ℹ️ Usa `stress-ng`, `fio`, `iperf3` conforme o modo escolhido e escreve marcações em `/var/lib/node_exporter/textfile` (textfile collector).

### O que o job executa (alto nível)

- Executa testes de performance (CPU/Mem/Disk/Net) na VM alvo e grava marcador no textfile collector do node_exporter.

## Adiciona VM ao  Pool do Load Balancer

**Arquivo:** `adiciona-vm-ao-pool-do-load-balancer.yaml`  
**ID/UUID:** `cupdate-lb-targets`  
**Grupo:** `root`

Varre o ambiente procurando por VMs que tenhham APP no nome e adiciona ao load balancer

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ⚠️ O script contém **API Key hardcoded**. Troque para usar a option do Rundeck (`$RD_OPTION_MGCAPIKEY`) e/ou Key Storage.
- ℹ️ `LB_NAME` está fixo como `teste-lb-will-cpen5-se`.
- ⚠️ Este job faz **replace** de targets no backend do Load Balancer (substitui o pool).

### O que o job executa (alto nível)

- Busca VMs `app-*`, coleta NICs e faz `replace` dos targets do backend do Load Balancer.

## Apaga as instancias de aplicação 

**Arquivo:** `apaga-as-instancias-de-aplica-o.yaml`  
**ID/UUID:** `delete-targets`  
**Grupo:** `root`

Restaura snapshot

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ⚠️ Este job **deleta instâncias** (`mgc virtual-machine instances delete --no-confirm`). Use com cuidado.

### O que o job executa (alto nível)

- Deleta instâncias cujo nome bate com `^app-[0-9]+$`.

## Atualiza nods do grafana 

**Arquivo:** `atualiza-nods-do-grafana.yaml`  
**ID/UUID:** `atualiza-nodes-grafana`  
**Grupo:** `root`

Varre o ambiente procurando por VMs que tenhham APP no nome e adiciona ao load balancer

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `RD_OPTION_MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ⚠️ Há **inconsistência no nome da variável**: o YAML define `RD_OPTION_MGCAPIKEY`, mas o script usa `RD_OPTION_RD_OPTION_MGCAPIKEY`.
- ℹ️ Atualiza targets do Prometheus em `/etc/prometheus/hosts_app.yml` e reinicia o serviço.
- ℹ️ Requer permissão de `sudo` para reiniciar o Prometheus na máquina onde o job roda.

### O que o job executa (alto nível)

- Executa um script Bash inline conforme definido no YAML.

## create node and add in load balancer and monitoring stack

**Arquivo:** `create-node-and-add-in-load-balancer-and-monitoring-stack.yaml`  
**ID/UUID:** `add-nodes-in-app-pool`  
**Grupo:** `root`

O que faz esse Job: Procura por um snapshot que tenha app no nome e o restaura. Coleta o IP privado dessa máquina Adiciona essa MV no pool do Load Balancer - LB Atualiza o Prometheus para que ele consiga enxhergar essa MV

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ℹ️ `LB_NAME` está fixo como `teste-lb-will-cpen5-se`.
- ℹ️ `MACHINE_TYPE_ID` está fixo: `0122fb85-0fd0-47d1-bf25-f24ca6f11493`.
- ℹ️ Atualiza targets do Prometheus em `/etc/prometheus/hosts_app.yml` e reinicia o serviço.
- ⚠️ Este job faz **replace** de targets no backend do Load Balancer (substitui o pool).
- ℹ️ Requer permissão de `sudo` para reiniciar o Prometheus na máquina onde o job roda.

### O que o job executa (alto nível)

- Restaura uma nova VM a partir de um snapshot (busca por `app-` no nome do snapshot).
- Busca VMs `app-*`, coleta NICs e faz `replace` dos targets do backend do Load Balancer.
- Coleta IPs IPv4 de portas relacionadas a `app-` e gera arquivo `file_sd` para Prometheus.

## Criar Load Balancer - CPEN5-SE

**Arquivo:** `criar-load-balancer-cpen5-se.yaml`  
**ID/UUID:** `create-lb-cpen5-se`  
**Grupo:** `root`

Cria um Load Balancer na MGC com base nas instâncias atuais

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ℹ️ `LB_NAME` está fixo como `teste-lb-will-cpen5-se`.

### O que o job executa (alto nível)

- Cria um Network Load Balancer e configura backend, health-check e listener.

## Instala prometheus 

**Arquivo:** `instala-prometheus.yaml`  
**ID/UUID:** `a3b5cfa6-fc27-4006-bc2e-5a2b5fed381c`  
**Grupo:** `root`

Esse script e instala o Prometheus em cada VM do ambiente (além de coletar os IPs como já faz), você pode fazer isso de forma paralela ou sequencial, usando ssh com chave previamente autorizada ou outro mecanismo de acesso (Rundeck plugin, Ansible, etc).

Abaixo está a versão adaptada do seu script, que assume:
	1.	Você tem uma chave SSH configurada com acesso root ou sudo às VMs.
	2.	O nome da chave SSH é passado por uma variável ou está no padrão ~/.ssh/id_rsa.
	3.	O IP privado será usado para conectar (então deve haver rota, ou VPN/SG permitindo).
	4.	A instalação do Prometheus será feita via apt (Ubuntu), com systemctl para iniciar o serviço.

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `API_KEY` | — | — | `` |  |
| `SSH_KEY_FILE` | — | — | `` |  |

**Notas e cuidados**

- ⚠️ O path `manaded-services` parece typo (provável `managed_services`). Ajuste para o caminho real da sua chave.

### O que o job executa (alto nível)

- Conecta via SSH nas VMs e instala/configura Prometheus e/ou Node Exporter de forma idempotente.

## Install Elasticsearch

**Arquivo:** `install-elasticsearch.yaml`  
**ID/UUID:** `8e0c9550-e357-47f1-b0a5-a04ccf43334f`  
**Grupo:** `root`

_Sem descrição no YAML._

### Opções

_Sem opções configuradas no YAML._

**Notas e cuidados**

- ℹ️ Este job instala pacotes via `apt` (Ubuntu/Debian) e requer `sudo`.

### O que o job executa (alto nível)

- Instala e inicia o Elasticsearch via repositório oficial (apt).

## Remove-instances-and-update-grafana

**Arquivo:** `remove-instances-and-update-grafana.yaml`  
**ID/UUID:** `remove-instances-and-update-grafana`  
**Grupo:** `root`

Varre o ambiente procurando por VMs que tenhham APP no nome, deleta e atualiza os nós do Grafana

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ℹ️ Atualiza targets do Prometheus em `/etc/prometheus/hosts_app.yml` e reinicia o serviço.
- ⚠️ Este job **deleta instâncias** (`mgc virtual-machine instances delete --no-confirm`). Use com cuidado.
- ℹ️ Requer permissão de `sudo` para reiniciar o Prometheus na máquina onde o job roda.

### O que o job executa (alto nível)

- Coleta IPs IPv4 de portas relacionadas a `app-` e gera arquivo `file_sd` para Prometheus.
- Deleta instâncias cujo nome bate com `^app-[0-9]+$`.

## Restaura snapshot 

**Arquivo:** `restaura-snapshot.yaml`  
**ID/UUID:** `restaura-targets`  
**Grupo:** `root`

Restaura snapshot

### Opções

| Opção | Obrigatória | Secure | Default | Descrição |
|---|---:|---:|---|---|
| `MGCAPIKEY` | ✅ | ✅ | `` | Chave de API da MGC |

**Notas e cuidados**

- ℹ️ `MACHINE_TYPE_ID` está fixo: `0122fb85-0fd0-47d1-bf25-f24ca6f11493`.

### O que o job executa (alto nível)

- Restaura uma nova VM a partir de um snapshot (busca por `app-` no nome do snapshot).


---

## Checklist rápido de hardening

Antes de compartilhar/rodar:

- [ ] Remover qualquer API Key hardcoded dos scripts
- [ ] Padronizar options: `MGCAPIKEY` (required + secure + storagePath)
- [ ] Corrigir typos de variáveis (`RD_OPTION_RD_OPTION_*`) e paths (`manaded-services`)
- [ ] Tornar configuráveis: `LB_NAME`, `MACHINE_TYPE_ID`, portas (80/9100), paths de targets
- [ ] Validar que `prometheus.yml` usa `file_sd_configs` apontando para `/etc/prometheus/hosts_app.yml`
- [ ] Adicionar logs de “diff” quando o job fizer replace de targets (para auditoria)

