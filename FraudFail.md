```mermaid
sequenceDiagram
    autonumber
    participant CardOrch as Cards Orchestrator
    participant AIGuard as AI/Sharia Guardrail
    participant ProdDNA as Product DNA
    participant Ledger as Headless Ledger
    participant Customer
    
    Note over CardOrch,Customer: Failure Scenario: AI Guardrail Timeout (Fail-Safe Mode)
    
    rect rgb(255, 245, 245)
        Note over CardOrch,AIGuard: Attempt 1: Normal AI Call
        
        activate CardOrch
        CardOrch->>AIGuard: POST /guardrail/evaluate<br/>Timeout: 100ms
        activate AIGuard
        Note over AIGuard: ML Model Processing...<br/>Fraud Detection Running...<br/>⏱️ Time passing...
        
        Note over AIGuard: ⚠️ Inference takes 560ms<br/>(GPU overloaded)
        
        AIGuard-->>CardOrch: ❌ Timeout Exception (100ms exceeded)
        deactivate AIGuard
        Note right of CardOrch: Time: 100ms<br/>Status: TIMEOUT
    end
    
    rect rgb(255, 250, 240)
        Note over CardOrch,AIGuard: Attempt 2: Retry with Extended Timeout
        
        CardOrch->>AIGuard: POST /guardrail/evaluate<br/>Timeout: 200ms (RETRY)
        activate AIGuard
        Note over AIGuard: ML Model still slow...<br/>⏱️ Still processing...
        
        AIGuard-->>CardOrch: ❌ Timeout Exception (200ms exceeded)
        deactivate AIGuard
        Note right of CardOrch: Time: 300ms total<br/>Status: RETRY_FAILED
    end
    
    rect rgb(255, 255, 240)
        Note over CardOrch,Ledger: Fail-Safe Decision Mode
        
        CardOrch->>CardOrch: Enter Fail-Safe Mode<br/>(No ML available)
        
        CardOrch->>ProdDNA: GET /products/checking/basic-rules
        activate ProdDNA
        ProdDNA-->>CardOrch: {max_failsafe_amount: 10000}
        deactivate ProdDNA
        
        Note over CardOrch: Fail-Safe Logic:<br/>Amount: $50 (5000 cents)<br/>Channel: POS<br/>Location: Home Country (UAE)<br/>Customer Tier: Standard
        
        alt Low-Risk Transaction (Amount < $100 + POS + Home Country)
            Note over CardOrch: ✓ Meets Fail-Safe Criteria
            CardOrch->>Ledger: Post Authorization (FAIL_SAFE_APPROVED)
            activate Ledger
            Note over Ledger: Flag: approved_via_failsafe=true
            Ledger-->>CardOrch: Success
            deactivate Ledger
            CardOrch-->>Customer: APPROVED (via Fail-Safe)
            
        else High-Risk Transaction (Amount > $100 OR Online OR Foreign)
            Note over CardOrch: ✗ Fails Fail-Safe Criteria
            CardOrch-->>Customer: DECLINED (Fail-Safe Rejection)
            Note right of Customer: Customer can retry<br/>or contact support
        end
        
        deactivate CardOrch
    end
    
    rect rgb(240, 255, 255)
        Note over CardOrch,AIGuard: Post-Incident Actions
        
        Note over CardOrch: Log for Review:<br/>- Transaction ID<br/>- Fail-Safe Decision<br/>- Timestamp<br/>- Customer ID
        
        Note over AIGuard: Auto-Scale Trigger:<br/>Add 2 more GPU nodes<br/>if timeout rate > 5%
        
        Note over CardOrch: Nightly Batch Review:<br/>Re-score fail-safe transactions<br/>with full ML model<br/>Flag any fraud misses
    end
