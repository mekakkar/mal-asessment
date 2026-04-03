```mermaid
sequenceDiagram
    autonumber
    participant Scheme as Visa Network
    participant Processor as Issuer Processor
    participant SchemeGW as Scheme Gateway
    participant RepayEngine as Repayment Engine
    participant CoreOrch as Core Orchestrator
    participant Ledger as Headless Ledger
    participant EventBus as Event Backbone
    participant BillingEng as Billing Engine
    participant ReconSvc as Reconciliation Service
    participant DataPlatform as Data Platform
    
    Note over Scheme,DataPlatform: Settlement Flow (T+1 Day After Authorization)
    Note over Scheme: Batch Cutoff: 11:00 PM UTC
    
    rect rgb(255, 240, 245)
        Note over Scheme,Processor: External Settlement Process
        Scheme->>Scheme: Generate Daily Settlement File<br/>Date: 2026-04-03<br/>Format: ISO 20022 XML
        Note right of Scheme: File Contains:<br/>- Auth Codes<br/>- Settlement Amounts<br/>- Interchange Fees<br/>- Scheme Fees
        
        Scheme->>Processor: SFTP Transfer<br/>/settlements/visa_2026-04-03.xml
        Note right of Processor: File Received: 11:15 PM
    end
    
    rect rgb(240, 255, 240)
        Note over Processor,Ledger: MAL Settlement Processing
        Processor->>SchemeGW: POST /webhooks/settlement/batch<br/>{settlement_id, date, total_txns, file_url}
        activate SchemeGW
        
        SchemeGW->>RepayEngine: Process Settlement File
        activate RepayEngine
        Note right of RepayEngine: Download from Processor<br/>Parse XML/CSV<br/>Match to Authorizations
        
        loop For Each Transaction in Batch
            RepayEngine->>CoreOrch: POST /core/settle-authorization<br/>{auth_id, amount, fees}
            activate CoreOrch
            
            CoreOrch->>Ledger: Convert PENDING → FINAL Settlement
            activate Ledger
            Note over Ledger: Multi-Entry Transaction:<br/>1. Finalize Customer DR (FINAL)<br/>2. Clear Processor Liability<br/>3. Record Interchange Fee (DR)<br/>4. Record Scheme Fee (DR)<br/>5. Net Payable to Visa (CR)
            
            Note over Ledger: Example for $50 Transaction:<br/>DR Customer Checking -$50.00 (FINAL)<br/>DR Processor Liability -$50.00<br/>DR Interchange Expense -$1.25<br/>DR Scheme Fees -$0.10<br/>CR Visa Payable +$51.35
            
            Ledger-->>CoreOrch: {settlement_id, status: POSTED}
            deactivate Ledger
            
            CoreOrch-->>RepayEngine: Settlement Complete
            deactivate CoreOrch
        end
        
        RepayEngine-->>SchemeGW: Batch Processing Complete<br/>{processed: 1247, failed: 0}
        deactivate RepayEngine
        deactivate SchemeGW
    end
    
    rect rgb(255, 250, 230)
        Note over Ledger,DataPlatform: Post-Settlement Events & Reconciliation
        
        Ledger->>EventBus: Publish: card.settlement.completed
        activate EventBus
        Note over EventBus: Topic: card.settlement.completed<br/>Payload: {settlement_id, amount, fees}
        
        par Parallel Consumers
            EventBus->>BillingEng: Update Customer Statement
            activate BillingEng
            Note right of BillingEng: Convert PENDING → POSTED<br/>Include in April Cycle
            BillingEng-->>BillingEng: Statement Updated
            deactivate BillingEng
            
            EventBus->>DataPlatform: Enrich Transaction
            activate DataPlatform
            Note right of DataPlatform: Add Final Fee Data<br/>Update Revenue Dashboards<br/>Train ML Models
            DataPlatform-->>DataPlatform: Analytics Updated
            deactivate DataPlatform
        end
        deactivate EventBus
    end
    
    rect rgb(240, 240, 255)
        Note over ReconSvc,DataPlatform: Daily Reconciliation (2:00 AM)
        
        activate ReconSvc
        ReconSvc->>Ledger: Query: SUM(settlements) WHERE date='2026-04-03'
        Ledger-->>ReconSvc: Ledger Total: $456,789.00
        
        ReconSvc->>SchemeGW: Get Settlement File Total
        SchemeGW-->>ReconSvc: Visa File Total: $456,789.00
        
        alt Totals Match
            ReconSvc->>ReconSvc: Status: MATCHED ✓
            Note right of ReconSvc: Log Success<br/>Update Dashboard
        else Totals Mismatch
            ReconSvc->>ReconSvc: Status: MISMATCH ⚠️
            Note right of ReconSvc: Alert Finance Team<br/>Create Investigation Ticket<br/>Variance: Calculate Difference
            ReconSvc->>ReconSvc: Generate Variance Report
        end
        deactivate ReconSvc
    end
