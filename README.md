# Sistema de Previsão de Ações com LSTM

Um sistema completo para coletar dados de ações, treinar modelos de previsão usando LSTM (Long Short-Term Memory) e servir previsões através de uma API REST.

## 📋 Funcionalidades

- **Coleta de Dados**: Download automático de dados históricos de ações usando Yahoo Finance
- **Treinamento de Modelos**: Modelos LSTM personalizados para cada ticker
- **API REST**: Endpoints para obter previsões e gerenciar modelos
- **Armazenamento Eficiente**: Dados particionados em formato Parquet
- **Previsões Multi-dia**: Previsão de preços para os próximos 1-30 dias

## 🏗️ Arquitetura

```mermaid
    sequenceDiagram
        participant Admin as Administrador
        participant Bulk as stock_bulk_loader.py
        participant YF as Yahoo Finance API
        participant Storage as Dados Parquet
        participant Script as stock_script.py
        participant Models as Modelos (.keras/.pkl)
        participant Scheduler as Cron/Scheduler
        participant Daily as stock_daily_loader.py
        participant API as stock_api.py
        participant Client as Cliente/Usuario
        
        Note over Admin, Client: FASE 1: CONFIGURAÇÃO INICIAL (Primeira Execução)
        
        Admin->>Bulk: python stock_bulk_loader.py
        activate Bulk
        Bulk->>YF: Download 3 anos histórico<br/>(AAPL, MSFT, NVDA, etc.)
        YF-->>Bulk: Dados históricos (OHLCV)
        Bulk->>Storage: Salva dados particionados<br/>(year/month/day)
        deactivate Bulk
        
        Admin->>Script: python stock_script.py
        activate Script
        Script->>Storage: Carrega dados parquet
        Storage-->>Script: Dados históricos
        
        loop Para cada ticker (AAPL, MSFT, etc.)
            Script->>Script: Preparar dados (60 sequências)
            Script->>Script: Construir modelo LSTM
            Script->>Script: Treinar modelo (100 epochs)
            Script->>Script: Avaliar métricas (RMSE, MAE)
            Script->>Models: Salvar modelo treinado<br/>({ticker}_model.keras + data.pkl)
        end
        deactivate Script
        
        Note over Admin, Client: FASE 2: OPERAÇÃO CONTÍNUA
        
        Admin->>API: python stock_api.py
        activate API
        Note right of API: API fica sempre ativa<br/>aguardando requisições
        
        Admin->>Scheduler: Configurar crontab/scheduler
        activate Scheduler
        
        loop Atualização Diária (Seg-Sex, 18h)
            Scheduler->>Daily: Executar daily_loader
            activate Daily
            Daily->>YF: Download dados do dia
            YF-->>Daily: Dados mais recentes
            Daily->>Storage: Atualizar partições
            deactivate Daily
        end
        
        loop Re-treinamento Semanal (Sáb, 2h)
            Scheduler->>Script: Re-executar treinamento
            activate Script
            Script->>Storage: Carregar dados atualizados
            Script->>Script: Re-treinar modelos
            Script->>Models: Atualizar modelos salvos
            deactivate Script
        end
        
        Note over Admin, Client: FASE 3: CONSUMO DA API
        
        Client->>API: GET /health
        API-->>Client: Status + modelos carregados
        
        Client->>API: GET /predict/AAPL?days=5
        activate API
        
        alt Modelo já carregado
            API->>API: Usar modelo em cache
        else Modelo não carregado
            API->>Models: Carregar modelo AAPL
            Models-->>API: Modelo + metadados
            API->>API: Cache modelo na memória
        end
        
        API->>API: Gerar previsões (5 dias)
        API->>API: Calcular datas úteis
        API-->>Client: JSON com previsões
        deactivate API
        
        Client->>API: GET /models
        API-->>Client: Lista de modelos disponíveis
        
        Note over Admin, Client: CICLO CONTÍNUO
        Note right of Scheduler: - Daily: Novos dados<br/>- Weekly: Re-treinamento<br/>- API: Sempre disponível<br/>- Cache: Modelos em memória
```

## 🚀 Instalação

### Dependências

```bash
pip install requirements.txt
```

### Tickers Suportados (Padrão)

- AAPL (Apple)
- MSFT (Microsoft) 
- NVDA (NVIDIA)
- AMZN (Amazon)
- META (Meta/Facebook)
- BRK-B (Berkshire Hathaway)
- GOOGL/GOOG (Google/Alphabet)
- AVGO (Broadcom)
- TSLA (Tesla)

## 📊 Coleta de Dados

### Carregamento Inicial (3 anos de histórico)

```bash
python stock_bulk_loader.py
```

### Atualização Diária

```bash
python stock_daily_loader.py
```

Os dados são salvos em formato Parquet com particionamento por ano/mês/dia para otimizar consultas.

## 🤖 Treinamento de Modelos

### Treinamento Interativo

```bash
python stock_script.py
```

O script oferece opções interativas para:
- Selecionar tickers específicos ou treinar todos
- Configurar número de epochs
- Visualizar métricas de performance

### Parâmetros do Modelo

- **Arquitetura**: 3 camadas LSTM (50 neurônios cada) + Dropout (0.2)
- **Sequência**: 60 dias de histórico para previsão
- **Otimizador**: Adam (learning_rate=0.001)
- **Função de Perda**: Mean Squared Error

### Métricas de Avaliação

- **MSE** (Mean Squared Error)
- **MAE** (Mean Absolute Error) 
- **RMSE** (Root Mean Squared Error)

## 🌐 API REST

### Iniciar o Servidor

```bash
python stock_api.py
```

### Endpoints Disponíveis

#### 1. Health Check
```
GET /health
```
**Resposta:**
```json
{
  "status": "healthy",
  "timestamp": "2025-05-27 10:30:00",
  "loaded_models": ["AAPL", "MSFT"]
}
```

#### 2. Previsão de Ações
```
GET /predict/<ticker>?days=5
```

**Parâmetros:**
- `ticker`: Símbolo da ação (ex: AAPL)
- `days`: Número de dias para prever (1-30, padrão: 5)

**Exemplo:**
```bash
curl http://localhost:5000/predict/AAPL?days=7
```

**Resposta:**
```json
{
  "ticker": "AAPL",
  "predictions": [
    {
      "date": "2025-05-28",
      "predicted_close": 182.45
    },
    {
      "date": "2025-05-29", 
      "predicted_close": 184.20
    }
  ],
  "prediction_date": "2025-05-27 10:30:00",
  "model_info": {
    "sequence_length": 60,
    "trained_ticker": "AAPL"
  }
}
```

#### 3. Listar Modelos
```
GET /models
```

**Resposta:**
```json
{
  "available_models": [
    {
      "ticker": "AAPL",
      "model_file": "aapl_model.keras", 
      "loaded": true
    }
  ],
  "total_models": 1
}
```

## 📁 Estrutura de Arquivos

### Modelos Salvos
Cada modelo gera dois arquivos:
- `{ticker}_model.keras`: Modelo neural Keras
- `{ticker}_data.pkl`: Scaler e metadados (joblib)

### Dados Particionados
```
dados/
├── year=2022/
│   ├── month=1/
│   │   ├── day=1/
│   │   └── day=2/
│   └── month=2/
├── year=2023/
└── year=2024/
```

## 🔧 Configuração Avançada

### Personalizar Tickers

Edite as listas nos arquivos `stock_bulk_loader.py` e `stock_daily_loader.py`:

```python
tickers = 'AAPL MSFT NVDA AMZN META GOOG TSLA'  # Seus tickers
```

### Ajustar Parâmetros do Modelo

No arquivo `stock_predictor.py`, classe `StockPredictor`:

```python
def __init__(self, sequence_length=60):  # Janela de histórico
    
def build_model(self):
    # Ajustar camadas LSTM, dropout, etc.
```

### Configurar Diretórios

No arquivo `stock_api.py`:

```python
prediction_api = StockPredictionAPI(models_dir="meus_modelos")
```

## ⚠️ Tratamento de Erros

### Erros Comuns

1. **Modelo não encontrado (404)**
   ```json
   {
     "error": "Modelo não encontrado para o ticker XYZ",
     "message": "Você precisa treinar um modelo para este ticker primeiro"
   }
   ```

2. **Dias inválidos (400)**
   ```json
   {
     "error": "Número de dias deve estar entre 1 e 30"
   }
   ```

3. **Dados insuficientes**
   - Mínimo necessário: 65+ registros por ticker (sequence_length + 5)

## 📈 Exemplo de Uso Completo

```bash
# 1. Coletar dados históricos
python stock_bulk_loader.py

# 2. Treinar modelos
python stock_script.py

# 3. Iniciar API
python stock_api.py

# 4. Fazer previsões
curl http://localhost:5000/predict/AAPL?days=5

# 5. Atualização diária (crontab)
python stock_daily_loader.py
```

## 🔄 Automação e Monitoramento

### Crontab para Atualizações

```bash
# Atualização diária às 18h (após fechamento do mercado)
0 18 * * 1-5 cd /path/to/project && python stock_daily_loader.py

# Re-treinamento semanal
0 2 * * 6 cd /path/to/project && python stock_script.py
```

### Logs e Monitoramento

A API utiliza logging Python padrão. Para logs estruturados:

```python
import logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
```

## ⚡ Performance e Otimizações

- **Cache de Modelos**: Modelos são carregados uma vez e mantidos em memória
- **Dados Particionados**: Consultas rápidas por data com Parquet
- **Validação de Entrada**: Filtros de dias úteis para previsões
- **Gestão de Memória**: Carregamento lazy dos modelos

## 📝 Notas Importantes

- **Mercados**: Previsões consideram apenas dias úteis (seg-sex)
- **Timezone**: Todas as datas em UTC por padrão
- **Accurácia**: Modelos são indicativos, não garantem performance futura
- **Re-treinamento**: Recomenda-se re-treinar periodicamente com dados atualizados

## 🤝 Contribuições

1. Fork o projeto
2. Crie uma branch para sua feature
3. Commit suas mudanças
4. Push para a branch
5. Abra um Pull Request

## 📄 Licença

Este projeto está sob licença MIT. Veja o arquivo LICENSE para detalhes.

## ⚠️ Disclaimer

Este software é apenas para fins educacionais e de pesquisa. Não constitui aconselhamento financeiro. Sempre consulte profissionais qualificados antes de tomar decisões de investimento.