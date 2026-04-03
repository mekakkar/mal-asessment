```mermaid
sequenceDiagram
    autonumber
    participant Customer
    participant Merchant as Merchant POS
    participant Scheme as Visa/Mastercard
    participant Processor as Issuer Processor
    participant JIT as JIT Auth Gateway
    participant CardOrch as Cards Orchestrator
    participant AcctReg as Account Registry
    participant CtrlDB as Card Control DB
    participant BalAPI as Balance API
    participant AIGuard as AI/Sharia Guardrail
    participant CoreOrch as Core Orchestrator
    participant ProdDNA as Product DNA
    participant Ledger as Headless Ledger
    participant EventBus as Event Backbone
    
    Note over Customer,EventBus: Authorization Flow (Target: <200ms)
    
    Customer->>Merchant: Swipe/Tap Card ($50 at Starbucks)
    Merchant->>Scheme: Authorization Request (ISO 8583)
    Scheme->>Processor: Route to Issuer (MAL Bank)
    
    rect rgb(240, 248, 255)
        Note over Processor,Ledger: MAL Platform Boundary (0ms start)
        Processor->>JIT: POST /webhooks/card/authorize<br/>{card_token, amount, merchant}
        activate JIT
        
        JIT->>CardOrch: AUTHORIZATION_REQUESTED event
        activate CardOrch
        Note right of CardOrch: Time: 5ms
        
        CardOrch->>AcctReg: GET /accounts/lookup?card_token=xyz
        activate AcctReg
        AcctReg-->>CardOrch: {customer_id, funding_account_id, status}
        deactivate AcctReg
        Note right of CardOrch: Time: 15ms
        
        CardOrch->>CtrlDB: HGETALL card:controls (Redis)
        activate CtrlDB
        CtrlDB-->>CardOrch: {geo_block, channel_limits, spending_limits}
        deactivate CtrlDB
        Note right of CardOrch: Time: 20ms
        
        CardOrch->>BalAPI: GET daily_spend (Redis)
        activate BalAPI
        BalAPI-->>CardOrch: {total_spend_today: $150}
        deactivate BalAPI
        Note right of CardOrch: Check: $150 + $50 < $1000 ✓<br/>Time: 25ms
        
        CardOrch->>AIGuard: POST /guardrail/evaluate<br/>{transaction, customer}
        activate AIGuard
        Note over AIGuard: Parallel Processing:<br/>1. Fraud ML Model<br/>2. Sharia Compliance<br/>3. Sanctions Screening
        AIGuard-->>CardOrch: {decision: APPROVE, fraud_score: 0.05,<br/>sharia_compliant: true, guardrail_token}
        deactivate AIGuard
        Note right of CardOrch: Time: 60ms (ML inference)
        
        CardOrch->>CoreOrch: POST /core/hold<br/>{account_id, amount, guardrail_token}
        activate CoreOrch
        Note right of CoreOrch: Time: 65ms
        
        CoreOrch->>ProdDNA: GET /products/checking/rules
        activate ProdDNA
        ProdDNA-->>CoreOrch: {overdraft: false, fees: {...}}
        deactivate ProdDNA
        Note right of CoreOrch: Time: 70ms
        
        CoreOrch->>Ledger: Post PENDING Authorization<br/>Idempotency: auth_jit_abc123
        activate Ledger
        Note over Ledger: Double-Entry:<br/>DR Customer Checking -$50<br/>CR Processor Liability +$50
        Ledger-->>CoreOrch: {transaction_id, new_balance: $450, status: POSTED}
        deactivate Ledger
        Note right of CoreOrch: Time: 120ms
        
        CoreOrch-->>CardOrch: Hold placed successfully
        deactivate CoreOrch
        Note right of CardOrch: Time: 125ms
        
        CardOrch-->>JIT: {decision: APPROVE, auth_code: AUTH_123456}
        deactivate CardOrch
        Note right of JIT: Time: 130ms
        
        JIT-->>Processor: HTTP 200 OK<br/>{approved: true, auth_code}
        deactivate JIT
        Note right of Processor: Time: 140ms
    end
    
    Processor->>Scheme: APPROVED
    Scheme->>Merchant: APPROVED
    Merchant->>Customer: Payment Successful ✓
    
    Note over Customer,Merchant: Total Authorization Time: 150ms ✅
    
    rect rgb(255, 250, 240)
        Note over Ledger,EventBus: Async Post-Authorization (Non-Blocking)
        Ledger->>EventBus: Publish: card.authorization.approved
        activate EventBus
        Note over EventBus: Topic: card.authorization.approved<br/>Event ID: evt_auth_001
        
        par Parallel Event Consumers
            EventBus->>BalAPI: Update balance cache
            Note right of BalAPI: Update daily spend: $200
            
            EventBus->>EventBus: Push Notification Service
            Note right of EventBus: Send: "Card used at Starbucks $50"
            
            EventBus->>EventBus: Data Platform (Snowflake)
            Note right of EventBus: Enrich + Store for analytics
            
            EventBus->>EventBus: CRM Sync (Salesforce)
            Note right of EventBus: Update customer timeline
        end
        deactivate EventBus
        Note over EventBus: All consumers process in 1-2 seconds
    end
