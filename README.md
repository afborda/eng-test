# Prova — logs de infra no Databricks Free

**Formato:** take-home + review de 60 min  
**Esforço:** 6–10 horas  
**Prazo:** 5–7 dias  

Queremos quem **ingira logs de infraestrutura**, modele dados confiáveis e transforme isso em informação útil para investigação. Gráfico vem depois do contrato de dados.

Este ficheiro é o enunciado. Não há outro `.md` neste pack.

## O que é obrigatório / o que é extra

| | Conta para passar | Não reprova se faltar |
|--|-------------------|------------------------|
| **Núcleo** | Medalhão Bronze→Silver→Gold com grain dito; lineage (`rec_id` / ficheiro / quando); relatório **porquê e como**; KPI em Gold; ≥ 3 queries com EXPLAIN e o **porquê** (performance, correção, ou “não precisava reescrever”) | |
| **Recomendado** | DLQ, partição `filter_date`/`source`, 2–3 checks (DNS vazio, count, duplicata) | Nota mais alta, não é hard gate |
| **Extra** | | sklearn ataque/falha; dbt; Great Expectations; OpenLineage |

Não pedimos Kimball, Data Vault nem grafo do Catalog.

## Padrões de modelagem esperados

Não é obrigatório seguir uma metodologia específica. O importante é explicar a escolha e manter um grain claro.

| Padrão | Como aplicar neste teste | Por que usar |
|--------|--------------------------|--------------|
| **Medallion (Bronze, Silver, Gold)** | Bronze preserva o dado recebido; Silver normaliza, deduplica e enriquece; Gold serve os indicadores | Separa rastreabilidade, qualidade e consumo. Permite reprocessar sem perder o original |
| **Fato de eventos** | Uma linha Silver representa uma ocorrência de segurança, com `rec_id`, horário, fonte, severidade, host e cliente | Mantém o detalhe para drill-down e permite novas agregações sem reler arquivos crus |
| **Dimensão de inventário** | Inventário resolve aliases de host para `customer_id` e `asset_id` | Evita repetir regras de cliente em cada fonte e torna os filtros consistentes |
| **Marts Gold agregados** | Postura diária, eventos por hora, top-N e correlação entre fontes | Reduz custo e latência do dashboard; cada tabela tem um objetivo e um grain explícito |
| **Lineage e chave estável** | Propagar `rec_id`, arquivo de origem, `received_at`/`ingestion_ts` e `filter_date` | Permite explicar de onde veio cada número e investigar atrasos ou duplicidades |
| **Carga idempotente** | `MERGE`/upsert por chave natural ou reconstrução determinística das tabelas derivadas | Uma segunda execução não pode alterar a contagem por duplicar registros |

O candidato pode usar outro padrão (por exemplo, tabelas por fonte em Silver) se explicar o trade-off. Evite criar abstrações sem necessidade: o modelo deve responder às perguntas do dashboard.

## Linguagem e Databricks

Ingestão na linguagem que quiser. **Recomendamos Go ou Python.** Rust, Java, shell… também vale.  
Databricks **Free Edition é um diferencial recomendado**, não uma exigência. Uma solução local com DuckDB, Postgres, SQLite ou outra tecnologia equivalente também é válida, desde que seja reproduzível com Docker Compose ou um comando documentado. Quem optar por Databricks deve usar notebooks Python + SQL; não use R nem Scala.

Cadastro: https://login.databricks.com/?dbx_source=docs&intent=CE_SIGN_UP  
Limites: https://docs.databricks.com/aws/en/getting-started/free-edition-limitations

| Pode usar no Databricks Free | Não peça (Free não tem / estoura quota) |
|-----------|------------------------------------------|
| Notebooks serverless **Python + SQL** | R, Scala, cluster personalizado |
| **1** SQL warehouse **2X-Small** | Warehouse maior, all-purpose clássico |
| Unity Catalog + **Volumes** (upload) | Storage AWS/GCP da empresa |
| `COPY INTO` Delta (preferido) | Streaming 24/7, DLT complexo, Jobs como caminho principal |
| Tabelas Delta Bronze / Silver / Gold | Model Serving, GPU |
| Dashboard **SQL** do Databricks | Tableau / Power BI / Looker pago |
| `EXPLAIN` no warehouse | Tuning de cluster que a Free não controla |
| scikit-learn **no notebook** (extra) | Serving do modelo |

Se o warehouse cair por quota, o que está no **Git** (SQL, notebooks, EXPLAIN) ainda tem de dar para avaliar. Um link de workspace, sozinho, não é entrega suficiente.

---

## Cenário e entrega

Time **Raven**. Hosts geram WAF/UFW, sshd/sudo, métricas, blocks e findings. Um coletor na borda escreve ficheiros. O Databricks é o lakehouse.

**Repositório GitHub com:**

1. **Ingest** (linguagem livre) que lê `dados/` e grava um landing.  
2. Pipeline Bronze → Silver → Gold. Databricks Free é recomendado como diferencial; uma alternativa local é aceita.
3. Dashboard + queries **no git**. URL do workspace sozinha não vale.
4. **`RELATORIO.md`** — o que mudou em cada camada, **como** e **porquê**. Sem isto a prova está incompleta.

| Camada | Onde | O que faz |
|--------|------|-----------|
| Ingest | Sua máquina | Lê `dados/` ou S3 → landing + lineage (path, versão) |
| Bronze | Banco/lake local ou Databricks | Carga 1:1, **guarda** `_origin` / ficheiro |
| Silver | Banco/lake local ou Databricks | Tipos, upsert, `filter_date`, dedup — **copia** `rec_id`, path, `received_at` |
| Gold | Banco/lake local ou Databricks | Posture, hourly, top-N, correlação — KPI **volta** ao Silver |
| Serve | Dashboard local ou SQL Warehouse | KPI **só** em Gold; drill-down Silver **com** data |

O ingest **não** precisa escrever Delta. JSONL ou Parquet chega.

---

## Dados

O repositório contém somente a amostra em **`dados/amostra/`** (3 dias, aproximadamente 2 MB). Ela é suficiente para desenvolver e testar o pipeline.

Para a avaliação de volume, use o pack completo: o tar tem aproximadamente 19 MB comprimido e, depois de extraído, deve ter pelo menos 40 MB e alguns milhões de registros. HTTP e S3 são o **mesmo** pack; escolha apenas uma forma de obtê-lo.

### No repo

`dados/amostra/bronze_*.parquet`

### Ou download HTTP

```bash
wget https://data.synthfin.com.br/raven-bronze.tar.gz
sha256sum -c <(echo '630b01e1aa8f0ade43f2979d36ef4040004ae8eefd186d7d12ce351ca9ee04ba  raven-bronze.tar.gz')
mkdir -p raw-data && tar -xzf raven-bronze.tar.gz -C raw-data
```

Se o DNS falhar: https://synthfin.com.br/data/raven-bronze.tar.gz  
Índice: https://data.synthfin.com.br/

### Ou S3 no Databricks Free (boto3)

O compute serverless **não** usa External Location nem `s3a://` com este MinIO. Usa **boto3 HTTP**.

Bucket `databricks`, prefixo `hiring/bronze/`, endpoint `https://s3.synthfin.com.br`.

```python
%pip install boto3

import boto3
from botocore.client import Config
from pathlib import Path

s3 = boto3.client(
    "s3",
    endpoint_url="https://s3.synthfin.com.br",
    aws_access_key_id="AKIADX3MN1EPMDXKPEUZ",
    aws_secret_access_key="7ZUSAzR1G4s4OjbzP7DySnywWx5JXhb8pES6BuuE",
    region_name="us-east-1",
    config=Config(s3={"addressing_style": "path"}),
)

print(s3.list_objects_v2(Bucket="databricks", Prefix="hiring/bronze/"))
obj = s3.get_object(Bucket="databricks", Key="hiring/bronze/bronze_waf.parquet")
Path("/tmp/bronze_waf.parquet").write_bytes(obj["Body"].read())
```

Pack único no S3: `hiring/raven-bronze.tar.gz`.  
**Não** apagues nem escrevas fora de `hiring/`. **Não** uses `s3a://`.

### O que vem no Bronze (cru, anonimizado)

Não há Silver nem Gold. IPs só em `198.18.0.0/15` e `10.0.0.0/8`. Não apagues `rec_id`, `_origin`, `received_at`.

| Ficheiro | Grain | O que o Silver tem de resolver |
|----------|-------|--------------------------------|
| `bronze_waf.parquet` | 1 request/block (~965k) | parse de `event_time`; `src` (`ufw`/`WAF`); join `host`; `filter_date`; dedup |
| `bronze_container.parquet` | 1 evento runtime (~626k) | `process` hashed; **não** é KPI de ataque |
| `bronze_auth.parquet` | 1 evento host (~249k) | flood `backup_missing`; brute curto |
| `bronze_system.parquet` | disco/journal/kernel (~80k) | `disk_critical` vs incidente |
| `bronze_dns.parquet` | 1 query | **0 linhas** — completeza, não gráfico |
| `bronze_cspm.parquet` | 1 finding | coluna `Sev` mista; último estado por finding+host |
| `bronze_inventory.parquet` | 1 asset | `cust`, `aliases` (`id\|ID\|name\|site`) para normalizar `host` |
| `bronze_blocks.parquet` | 1 block | parse de data; join host/IP |
| `bronze_metrics.parquet` | 1 scrape | `ts` misto; join host |
| `bronze_incidents.parquet` | 1 ticket | rótulo fraco; `affected_hosts` sujo |

Timestamps mistos, `src` com casing inconsistente, `host` sujo, duplicatas, `received_at` atrasado. **Não** vem `filter_date` nem `customer_id` nos eventos — isso é Silver (join com inventory).

---

## Barra do ingest

**Núcleo:** lê `dados/` (ou o mesmo pack via HTTP/S3); segunda execução do mesmo dia **não duplica**; lineage em cada linha (path, hora, versão).

**Recomendado:** partição `filter_date`/`source`; DLQ (linha inválida não mata o batch); teste de parser (WAF + auth + uma linha podre); não carregar ~1,9M linhas na RAM se puder iterar.

README do teu repo: um comando (`python -m ingest`, `go run`, `docker compose`, …).

---

## Execução e diferencial Databricks Free

Quem usar Databricks deve criar um schema no Unity Catalog (por exemplo, `raven.bronze`, `raven.silver` e `raven.gold`). Quem usar outra tecnologia deve entregar o equivalente no banco escolhido.

Toda tabela que o warehouse lê tem `filter_date DATE NOT NULL`.  
Toda query de dashboard começa com `WHERE filter_date BETWEEN :start AND :end`.

Gold sugerido: `gold_posture_daily` `(customer_id, filter_date)`, `gold_events_hourly`, `gold_topn_daily`, uma `gold_corr_*`.

Usar o Databricks Free Edition, com notebooks Python/SQL, Delta Lake, SQL Warehouse e um dashboard SQL, é um diferencial porque demonstra familiaridade com um ambiente real de lakehouse. A entrega continua precisando ser revisável pelo Git e não pode depender apenas de uma URL ou de credenciais privadas.

No git: notebooks `01_bronze` / `02_silver` / `03_gold`; `sql/` dos widgets; ≥ 3 queries com EXPLAIN; prints ou HTML do dashboard; `RELATORIO.md`.

### Dashboard

- Posture: crit/high **abertos**, clientes, assets expostos, freshness (`max(ingestion_ts)`)
- Tendência dia/hora com severity/action
- Top-N (IP, user, asset ou customer)
- Drill-down ordenado, data limitada, `LIMIT`
- Uma correlação (auth + blocks, ou CSPM exposto + WAF)
- Filtros: data, customer, severity

`COUNT` de WAF blocked como “ataques hoje” não passa.  
Gráfico de DNS com feed vazio não passa.

---

## Lineage (obrigatório, mínimo)

Conseguir responder: *este número no dashboard veio de que linha, de que ficheiro, de quando?*

Não pedimos o grafo do Unity Catalog. Pedimos colunas que atravessam as camadas.

| Camada | Manter |
|--------|--------|
| Ingest / Bronze | path do parquet (`_origin` ou `_metadata.file_path`), `rec_id` |
| Silver | os mesmos + `filter_date` + `ingestion_ts` |
| Gold | `filter_date` + `customer_id`; drill-down usa `rec_id` no Silver |

No relatório, um exemplo concreto: *“o KPI X no dia D vem destas linhas Silver / deste `bronze_auth.parquet`.”*

---

## Relatório (`RELATORIO.md`) — copia e preenche

Máximo ~6 páginas. Sem o “porquê”, não conta.

```md
## Bronze
- O que chegou cru:
- O que não tratei de propósito:
- Como o ingest grava o landing:

## Silver
- Ajustes (parse, filter_date, join, canónicos, dedup, late):
- Porquê cada um:
- Grain (1 linha = ):

## Agregações
| Agregação | Grain | Porquê |
|-----------|-------|--------|
|           |       |        |
Porquê não agregar no Bronze a cada clique:

## Gold
- Posture / hourly / top-N / correlação — como e porquê:
- O que ficou de fora:

## Lineage
- Colunas Bronze → Silver → Gold:
- Exemplo: número do dashboard → linhas Silver → ficheiro Bronze:

## Queries (≥ 3)
SQL inicial → EXPLAIN → rewrite ou “não precisava” → porquê
(performance / correção / o Gold já chega)

| # | Objetivo | O que mudei (ou não) | Porquê |
|---|----------|----------------------|--------|
| 1 |          |                      |        |
| 2 |          |                      |        |
| 3 |          |                      |        |

## Medalhão
- Porquê estas três camadas neste problema:
- O que falharia em produção se saltasse uma:
```

---

## Extra (não reprova)

Não substitui o núcleo. sklearn no notebook: padrão de **ataque** (burst futuro) **ou** **falha** de host a partir de métricas. Split **por tempo**. Features em *t* só usam dados ≤ *t*. Baseline vs modelo. Sem API paga, sem LLM, sem Model Serving.

## Restrições

- Só Databricks **Free Edition** (tua conta). Sem keys AWS da empresa.  
- Sem BI proprietário.  
- Não tentes reverter IPs sintéticos.
