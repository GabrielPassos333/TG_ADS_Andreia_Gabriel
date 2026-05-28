# RBMC · PPP Monitor Dashboard

> Dashboard de monitoramento para visualização de dados processados através do **n8n** e armazenados no **Supabase**.

## 📋 Sobre

Dashboard desenvolvido como parte do **Trabalho de Conclusão de Curso (TCC)** do curso de **Análise e Desenvolvimento de Sistemas** da **FATEC Indaiatuba**.

Este projeto oferece uma interface visual para:
- ✅ Monitorar dados processados pela automação n8n
- 📊 Visualizar estatísticas de processamento em tempo real
- 📈 Acompanhar taxas de sucesso, erros e pendências
- 🗂️ Controlar registros geodésicos do IBGE-PPP
- 🔍 Filtrar dados por estação e ano

---

## 🚀 Quick Start

### 1. Clonar o Repositório

```bash
git clone <seu-repositorio>
cd Dashboard
```

### 2. Configurar Credenciais do Supabase

Copie o arquivo de template e preencha com suas credenciais:

```bash
cp config.example.js config.js
```

Edite `config.js` com sua URL e chave anon do Supabase:

```javascript
const CONFIG = {
  url: 'https://seu-projeto.supabase.co',
  key: 'sua-anon-key-aqui'
};
```

> **Encontre suas credenciais em:** [Supabase Console](https://app.supabase.com) → Seu projeto → Settings → API

### 3. Abrir o Dashboard

Simplesmente abra `index.html` em seu navegador:

---

## 📊 Funcionalidades

### Métricas em Tempo Real
- **Total de Registros**: Quantidade total de dias processados
- **Concluídos**: Registros com todas as etapas OK
- **Com Erro**: Registros que falharam em alguma etapa
- **Pendentes**: Registros aguardando processamento

### Análise de Desempenho
- Tempo médio por etapa de processamento
- Taxa de sucesso por etapa (Download, PPP, Drive)
- Precisão média (σ Lat, σ Lon, σ H)

### Filtros e Controles
- Filtrar por **estação geodésica**
- Filtrar por **ano**
- Atualizar dados manualmente
- Visualizar últimas atualizações

### Tabela Detalhada
Informações completas de cada processamento:
- Dia processado e estação
- Status de cada etapa (Download, PPP, Drive)
- Coordenadas (Lat/Lon)
- Precisão vertical e horizontal
- Coordenadas UTM
- Timestamp de atualização

---

## 🏗️ Estrutura do Projeto

```
Dashboard/
├── index.html           # Interface principal do dashboard
├── styles.css           # Estilos e design responsivo
├── config.js            # Credenciais (NÃO fazer commit!)
├── .gitignore          # Ignora config.js
└── README.md           # Este arquivo
```

---

## 🔌 Integração com n8n

Este dashboard lê dados de uma automação **n8n** que:

1. **Baixa arquivos ZIP** de dados GNSS
2. **Envia para processamento** no IBGE-PPP
3. **Salva resultados** no Supabase

A estrutura esperada no Supabase:

### Tabela: `registros_ibge`
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
```

### Tabela: `resultados_ppp`
```sql
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
```

---

## 🔐 Segurança

⚠️ **IMPORTANTE**: Suas credenciais do Supabase são **sensíveis**!

- ✅ `config.js` está no `.gitignore` (nunca será commitado)
- ✅ Repositório público contém apenas `config.example.js`
- ✅ Cada usuário deve criar seu próprio `config.js`
- ❌ NUNCA faça commit de credenciais reais

---

## 🛠️ Desenvolvimento

### Requisitos
- Navegador moderno (Chrome, Firefox, Safari, Edge)
- Conexão com internet (API do Supabase)
- Conta Supabase com credenciais válidas

### Modificações Locais

Para desenvolver localmente, você pode abrir diretamente `index.html` no navegador. 


### Estrutura do Código

O `index.html` contém todo o código JavaScript inline com:
- `CONFIG`: Configurações do Supabase
- `headers()`: Prepara requisições autenticadas
- `fetchAll()`: Busca dados da API REST do Supabase
- `renderMetrics()`: Renderiza cards de métricas
- `renderEtapas()`: Calcula tempos médios por etapa
- `renderBars()`: Gera gráficos de barras
- `renderTabela()`: Popula tabela com registros
- `loadAll()`: Função principal que orquestra tudo

---

## 📱 Design

Dashboard desenvolvido com:
- **Design System**: Cores e tipografia customizadas
- **Tipografia**: IBM Plex Sans + IBM Plex Mono
- **Responsive**: Funciona em desktop, tablet e mobile
- **Tema Escuro**: Interface dark mode para reduzir fadiga ocular
- **Performance**: CSS puro, sem frameworks

---

## 📈 Métricas Monitoradas

| Métrica | Descrição |
|---------|-----------|
| Taxa de Sucesso | % de registros OK por etapa |
| Tempo Médio | Tempo médio de processamento por etapa |
| Precisão (σ) | Desvio padrão vertical e horizontal |
| Eficiência | Razão entre OK e Total |

---

## 🐛 Troubleshooting

### Banner de erro "Chave de API não configurada"
- Verifique se `config.js` existe
- Confirme que as credenciais estão corretas
- Teste acesso ao Supabase manualmente

### Tabela não carrega dados
- Verificar se o Supabase está online
- Confirmar que as tabelas existem
- Validar permissões RLS no Supabase
- Abrir console do navegador (F12) para ver erros




---

## 📜 Licença

Este projeto é fornecido como trabalho acadêmico. Use livremente para fins educacionais.

---

## 🔗 Links Úteis

- [Supabase Documentation](https://supabase.com/docs)
- [n8n Workflows](https://n8n.io/)
- [IBGE PPP](https://www.ibge.gov.br/pt/inicio.html)
- [FATEC Indaiatuba](https://www.fatec.sp.gov.br/)

---

**Última atualização:** Maio 2026  
