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