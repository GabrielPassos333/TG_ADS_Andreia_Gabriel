# RBMC-PPP · Automação de Processamento Geodésico

> Trabalho de Graduação — FATEC Indaiatuba  
> Alunos: **Gabriel** e **Andréia**

---

## Sobre o projeto

Este projeto automatiza o ciclo completo de processamento de dados geodésicos da **Rede Brasileira de Monitoramento Contínuo dos Sistemas GNSS (RBMC)** do IBGE, utilizando a plataforma de automação **n8n**.

O sistema realiza, de forma automática e resiliente:

1. Download dos arquivos RINEX (formato ZIP) do FTP do IBGE
2. Descompactação e filtragem do arquivo de observação (`.d`)
3. Envio para processamento no serviço **IBGE-PPP** (Posicionamento por Ponto Preciso)
4. Salvamento dos resultados geodésicos em banco de dados
5. Upload do arquivo `.sum` para o Google Drive
6. Notificação de conclusão via Telegram
7. Geração de insights sobre os dados processados via Inteligência Artificial

---

## Arquitetura do sistema

```
IBGE FTP ──────────────────────────────────────────────────┐
                                                            │
IBGE-PPP ◄──────── n8n (orquestrador) ────────────────────►│
                        │                                   │
                        ├──► Supabase (Postgres) — primário │
                        ├──► MySQL local — backup           │
                        ├──► Google Drive — arquivos .sum   │
                        ├──► Telegram — notificações        │
                        └──► Dashboard HTML — visualização  │
                                                            │
IA 1 (SQL) ──► Postgres ──► IA 2 (Insights) ──────────────┘
```

O workflow é dividido em duas fases:

**Fase 1 — Preparar ambiente** (roda uma vez por conjunto de dados)
Gera os links de download e popula o banco de dados com todos os registros em status `pendente`.

**Fase 2 — Executar / Retomar** (pode rodar múltiplas vezes com segurança)
Busca no banco todos os registros com status `pendente` ou `erro` e processa cada um em loop. Em caso de falha em qualquer etapa, o erro é registrado no banco e o próximo item é processado — o fluxo nunca trava.

---

## Funcionalidade de IA

O projeto conta com um fluxo auxiliar de análise inteligente, onde duas IAs trabalham em conjunto:

- **IA 1** recebe a pergunta do usuário em linguagem natural e gera automaticamente a query SQL correspondente, consultando apenas as tabelas permitidas (`registros_ibge` e `resultados_ppp`) e nunca executando operações de escrita
- **IA 2** recebe o resultado bruto do banco de dados e transforma em uma resposta clara em português, destacando anomalias, métricas e conclusões

Exemplos de perguntas que o sistema responde:
- *"Quais dias tiveram erro no processamento PPP?"*
- *"Qual a média de tempo de execução por estação?"*
- *"Quais resultados tiveram sigma da altura acima de 0,005 m?"*
- *"Qual a variação de coordenadas da estação VICO ao longo do ano?"*

---

## Pré-requisitos

- [n8n](https://n8n.io/) — self-hosted ou cloud
- Conta no [Supabase](https://supabase.com/) — plano gratuito suficiente
- MySQL rodando localmente
- Conta Google com acesso ao Google Drive
- Bot do Telegram criado via [@BotFather](https://t.me/BotFather)
- API key de um modelo de IA (Google Gemini, Anthropic Claude, ou outro compatível com o n8n)

---

## Banco de dados

Execute os scripts abaixo **uma única vez** em cada banco antes de importar o workflow.

### Postgres — Supabase

```sql
CREATE TABLE IF NOT EXISTS registros_ibge (
  id              SERIAL          PRIMARY KEY,
  estacao         TEXT            NOT NULL,
  ano             TEXT            NOT NULL,
  dia_corrido     TEXT            NOT NULL,
  url_download    TEXT            UNIQUE NOT NULL,
  status_download TEXT            NOT NULL DEFAULT 'pendente',
  status_ppp      TEXT            NOT NULL DEFAULT 'pendente',
  status_drive    TEXT            NOT NULL DEFAULT 'pendente',
  erro_msg        TEXT,
  arquivo_ppp     TEXT,
  criado_em       TIMESTAMP       NOT NULL DEFAULT NOW(),
  atualizado_em   TIMESTAMP       NOT NULL DEFAULT NOW(),
  inicio_execucao TIMESTAMP,
  fim_execucao    TIMESTAMP,
  tempo_execucao  NUMERIC(10,3)
);

CREATE TABLE IF NOT EXISTS resultados_ppp (
  id              SERIAL          PRIMARY KEY,
  registro_id     INT             UNIQUE REFERENCES registros_ibge(id),
  url_download    TEXT            UNIQUE NOT NULL,
  link_zip        TEXT,
  status_ppp      TEXT,
  processo_em     TIMESTAMP,
  rinex           TEXT,
  marco           TEXT,
  inicio          TIMESTAMP,
  fim             TIMESTAMP,
  modo            TEXT,
  antena          TEXT,
  altant          NUMERIC(8,3),
  intervalo       NUMERIC(6,2),
  frequencia      TEXT,
  observacao      TEXT,
  orbita          TEXT,
  datum           TEXT,
  lat             TEXT,
  lon             TEXT,
  hgeo            NUMERIC(10,4),
  hnor            NUMERIC(10,4),
  sigma_lat       NUMERIC(8,5),
  sigma_lon       NUMERIC(8,5),
  sigma_hgeo      NUMERIC(8,5),
  utmn            NUMERIC(12,3),
  utme            NUMERIC(12,3),
  mc              INT,
  modelo_geoidal  TEXT,
  fator_correcao  NUMERIC(8,4),
  incerteza       NUMERIC(8,4),
  criado_em       TIMESTAMP       NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_registros_estacao   ON registros_ibge (estacao);
CREATE INDEX IF NOT EXISTS idx_registros_ano       ON registros_ibge (ano);
CREATE INDEX IF NOT EXISTS idx_registros_status    ON registros_ibge (status_download, status_ppp, status_drive);
CREATE INDEX IF NOT EXISTS idx_resultados_url      ON resultados_ppp (url_download);
CREATE INDEX IF NOT EXISTS idx_resultados_marco    ON resultados_ppp (marco);
CREATE INDEX IF NOT EXISTS idx_resultados_processo ON resultados_ppp (processo_em);
```

### MySQL — servidor local

```sql
CREATE TABLE IF NOT EXISTS registros_ibge (
  id              INT AUTO_INCREMENT  PRIMARY KEY,
  estacao         VARCHAR(50)         NOT NULL,
  ano             VARCHAR(10)         NOT NULL,
  dia_corrido     VARCHAR(10)         NOT NULL,
  url_download    VARCHAR(500)        UNIQUE NOT NULL,
  status_download VARCHAR(20)         NOT NULL DEFAULT 'pendente',
  status_ppp      VARCHAR(20)         NOT NULL DEFAULT 'pendente',
  status_drive    VARCHAR(20)         NOT NULL DEFAULT 'pendente',
  erro_msg        TEXT,
  arquivo_ppp     TEXT,
  criado_em       DATETIME            NOT NULL DEFAULT NOW(),
  atualizado_em   DATETIME            NOT NULL DEFAULT NOW(),
  inicio_execucao DATETIME,
  fim_execucao    DATETIME,
  tempo_execucao  DECIMAL(10,3)
);

CREATE TABLE IF NOT EXISTS resultados_ppp (
  id              INT AUTO_INCREMENT  PRIMARY KEY,
  registro_id     INT                 UNIQUE,
  url_download    VARCHAR(500)        UNIQUE NOT NULL,
  link_zip        TEXT,
  status_ppp      VARCHAR(20),
  processo_em     DATETIME,
  rinex           VARCHAR(100),
  marco           VARCHAR(50),
  inicio          DATETIME,
  fim             DATETIME,
  modo            VARCHAR(50),
  antena          VARCHAR(100),
  altant          DECIMAL(8,3),
  intervalo       DECIMAL(6,2),
  frequencia      VARCHAR(20),
  observacao      VARCHAR(100),
  orbita          VARCHAR(50),
  datum           VARCHAR(100),
  lat             VARCHAR(50),
  lon             VARCHAR(50),
  hgeo            DECIMAL(10,4),
  hnor            DECIMAL(10,4),
  sigma_lat       DECIMAL(8,5),
  sigma_lon       DECIMAL(8,5),
  sigma_hgeo      DECIMAL(8,5),
  utmn            DECIMAL(12,3),
  utme            DECIMAL(12,3),
  mc              INT,
  modelo_geoidal  VARCHAR(100),
  fator_correcao  DECIMAL(8,4),
  incerteza       DECIMAL(8,4),
  criado_em       DATETIME            NOT NULL DEFAULT NOW(),
  FOREIGN KEY (registro_id) REFERENCES registros_ibge(id)
);
```

---

## Configuração das credenciais

Após importar o arquivo `.json` no n8n, localize e configure as seguintes credenciais. Todos os nós que precisam de ajuste estão identificados abaixo:

### Postgres (Supabase)

Nos nós com nome **"Postgres account"** ou **"Supabase Postgres"**:

| Campo | Valor |
|---|---|
| Host | `aws-0-us-east-1.pooler.supabase.com` |
| Port | `6543` |
| Database | `postgres` |
| User | `postgres.SEU_PROJECT_ID` |
| Password | sua senha do Supabase |
| SSL | obrigatório |

A URL do projeto e a **anon key** (para o dashboard HTML) ficam em: Supabase → Settings → API.

### MySQL local

Nos nós com nome **"MySQL local"**:

| Campo | Valor |
|---|---|
| Host | `localhost` ou IP do servidor |
| Port | `3306` |
| Database | nome do seu banco |
| User | seu usuário MySQL |
| Password | sua senha MySQL |

### Google Drive

Nos nós **"Upload Google Drive"**:
1. Crie uma credencial OAuth2 do Google Drive no n8n
2. Atualize o `folderId` para o ID da pasta onde os arquivos `.sum` serão salvos — o ID aparece na URL do Drive ao abrir a pasta

### Telegram

No nó **"Telegram"**:
1. Crie um bot via [@BotFather](https://t.me/BotFather) e copie o token
2. Configure a credencial Telegram no n8n com esse token
3. Atualize o `chatId` para o ID do chat onde as notificações serão enviadas

### IBGE-PPP

No nó **"IBGE-PPP"**, atualize o campo `email` para o seu endereço de e-mail — o IBGE usa esse campo para identificar o solicitante:

```
email: seu@email.com
```

### IA (Chat Model)

No nó **AI Agent**, conecte o modelo de sua preferência no slot **Chat Model**:
- Google Gemini → credencial com API key do Google AI Studio
- Anthropic Claude → credencial com API key da Anthropic
- OpenAI → credencial com API key da OpenAI

---

## Como usar

### Fase 1 — Preparar ambiente

1. Abra o workflow no n8n
2. No nó **INSERIR DADOS**, configure:
   - `estacao` — código da estação (ex: `vico`)
   - `ano` — ano desejado (ex: `2023`)
   - `primeiro_dia` — dia juliano inicial (ex: `121`)
   - `ultimo_dia` — dia juliano final (ex: `365`)
3. Execute o trigger **"When clicking Execute Workflow"**

Isso popula o banco com todos os registros em status `pendente`. Esta fase só precisa ser executada uma vez por conjunto de dados.

### Fase 2 — Processar

1. Execute o trigger **"Executar / Retomar workflow"**
2. O sistema busca automaticamente todos os registros pendentes ou com erro e processa cada um
3. Ao final, uma mensagem é enviada no Telegram com o resumo da execução
4. Em caso de falha parcial, basta executar novamente — apenas os registros não concluídos serão reprocessados

### Fluxo de IA

1. Execute o trigger do workflow de IA
2. Preencha o campo `pergunta` com sua dúvida em português
3. A IA 1 gera o SQL, o banco executa, a IA 2 interpreta e responde

### Dashboard

Abra o arquivo `dashboard_cavalo.html` em qualquer navegador. Antes da primeira abertura, edite o arquivo e substitua `COLE_SUA_ANON_KEY_AQUI` pela **anon key** do seu projeto Supabase.

---

## Estrutura do repositório

```
├── Cavalo_v2c.json          # Workflow principal (n8n)
├── dashboard_cavalo.html    # Dashboard de monitoramento
├── schema_rbmc.sql          # CREATE TABLE (Postgres)
└── README.md
```

---

## Observações técnicas

- O workflow usa **dual-write**: cada operação é gravada primeiro no Postgres (Supabase) e depois replicada no MySQL local. Uma falha no MySQL não interrompe o fluxo principal
- O controle de estado é feito pelo Postgres — o MySQL é estritamente backup
- Os campos `lat` e `lon` são mantidos como `TEXT` porque o IBGE-PPP retorna no formato graus minutos segundos (`-20 45 41,4020`). Uma conversão para decimal pode ser feita em pós-processamento
- O tempo de execução por registro é calculado como `fim_execucao - inicio_execucao` em segundos e salvo na coluna `tempo_execucao`

---

## Licença

Projeto acadêmico — FATEC Indaiatuba · 2025
