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
├── config.example.js    # Template de configuração
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
- dia_corrido (text)
- estacao (text)
- ano (integer)
- status_download (enum: ok, erro, pendente)
- status_ppp (enum: ok, erro, pendente)
- status_drive (enum: ok, erro, pendente)
- url_download (text)
- erro_msg (text, opcional)
- criado_em (timestamp)
- atualizado_em (timestamp)
- iniciado_download_em (timestamp, opcional)
- concluido_download_em (timestamp, opcional)
- enviado_ppp_em (timestamp, opcional)
- concluido_ppp_em (timestamp, opcional)
- concluido_drive_em (timestamp, opcional)
```

### Tabela: `resultados_ppp`
```sql
- url_download (text) - chave estrangeira
- marco (text)
- lat (text)
- lon (text)
- sigma_lat (numeric)
- sigma_lon (numeric)
- sigma_hgeo (numeric)
- utmn (text)
- utme (text)
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

Para desenvolver localmente, você pode servir o arquivo com um servidor web simples:

```bash
# Python 3
python -m http.server 8000

# Node.js
npx http-server

# php
php -S localhost:8000
```

Então acesse: `http://localhost:8000`

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

### Tempos não aparecem
- Adicione colunas de timestamp nas tabelas do Supabase conforme estrutura acima

---

## 📞 Contato & Suporte

**Desenvolvido por:** [Seu Nome]  
**Instituição:** FATEC Indaiatuba  
**Curso:** Análise e Desenvolvimento de Sistemas  
**Ano:** 2024-2026  

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
**Status:** ✅ Em produção
