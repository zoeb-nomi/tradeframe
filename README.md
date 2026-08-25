# TradeFrame

A personal trading operating system: automated reflexes, AI judgment, human (or algorithmic) execution — with risk management as the first-class module. Two books, one risk standard: a **US equities book** (live; fully automated analysis, human one-tap execution) and an **India equities book** (in build; algorithmic execution via broker API).

**📖 Interactive system guide:** https://zoeb-nomi.github.io/tradeframe/ *(sanitized public edition — operational identifiers deliberately absent)*

## System flow

```mermaid
flowchart TB
    PRM["PRM — Portfolio Risk Manager\n(parent · observe + tripwires + cost ledger → real returns)"]

    subgraph USB["US book — live"]
        FRMUS["FRM-US\nhard rails · breachable bands · scored judgment"]
        WATCH["Arm-and-watch\nnightly proposals → intraday triggers"]
        HUMAN["Human one-tap confirm\n(broker has no API)"]
        FRMUS --> WATCH --> HUMAN
    end

    subgraph INB["India book — in build"]
        FRMIN["FRM-IN\nsame standard · R-based sizing · circuit breakers"]
        ALGO["Validated algo execution\n(broker order API · kill switch · dry-run first)"]
        FRMIN --> ALGO
    end

    JUDG["Judgment layer\nAI briefs · research · monthly counterfactual audit"]
    REFLEX["Reflex layer\nalways-on workflows: polling · thresholds · alerts · digests"]
    MEM["Memory\nledger · state tables · versioned specs"]

    JUDG -- "proposes only, never executes" --> FRMUS
    JUDG -- "proposes only, never executes" --> FRMIN
    REFLEX --> FRMUS
    REFLEX --> FRMIN
    PRM --- FRMUS
    PRM --- FRMIN
    USB --> MEM
    INB --> MEM
    MEM --> JUDG
```

**Design rules that hold everywhere:** the AI proposes, a deterministic layer validates against hard rails, and only then does a human (US) or validated code (India) execute — an LLM hallucinating a quantity is structurally incapable of causing a trade. Each book has its own kill switch; an automated halt can stop *new* risk but never touches exits or protective stops. Every decision is scored against a do-nothing baseline, and returns are reported net of **all** costs.
