# SinaApp Agentless Behavioral Fraud Detection

Behavioural banking fraud detection by [SINAINSIGHT](https://www.sinainsight.pro), built on the [SinaApp](https://www.sinainsight.pro/sinaapp/) stateful synthetic monitoring engine.

## How It Works

Traditional fraud detection (CEP / Complex Event Processing) evaluates each transaction individually, or within a sliding window of minutes to hours. SinaApp takes a fundamentally different approach:

1. **Scan every account** on a regular cycle (5 min for suspect accounts, 60 min otherwise)
2. **Remember previous cycles** using sina_mem, an encrypted persistent key-value store
3. **Detect behavioural patterns** that span days or weeks — accumulations of weak signals invisible to transaction-by-transaction analysis

### Example: The Slow Drain

A fraudster executes 70 individually normal transfers over 3 weeks, each below every threshold. No single transaction triggers an alert. But SinaApp remembers the account balance at each cycle and detects that 95% of the account has been drained — a pattern that CEP cannot see.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│  Scheduler   │────▶│   Agents     │────▶│   GoPhantom   │
│  (dispatch)  │     │  (execute)   │     │  (pure-Go     │
│              │     │              │     │   browser)    │
└──────┬───────┘     └──────┬───────┘     └───────────────┘
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│   MongoDB    │     │    Kafka     │
│  (accounts,  │     │  (results,   │
│   sina_mem)  │     │   metrics)   │
└──────────────┘     └──────────────┘
```

- **Scheduler**: Dispatches accounts to agents based on priority and cycle time
- **Agents**: Execute fraud detection scenarios against each account via GoPhantom
- **GoPhantom**: Pure-Go browser engine (no Chromium) — goja JS + goquery DOM + net/http
- **sina_mem**: Encrypted cross-cycle memory — each scenario remembers past results
- **Kafka**: Distributes results and aggregates metrics

All components are **agentless** — nothing is installed on the monitored banking systems.

## Proof of Concept Results

A live POC runs continuously at [fraud-poc.sinainsight.eu](https://fraud-poc.sinainsight.eu/):

| Metric | Value |
|--------|-------|
| Simulated accounts | 100,000 |
| Fraud scenarios per account | 7 |
| Total executions per cycle | 700,000 |
| Cycle time | < 60 minutes |
| True positives | 998 / 1,000 |
| False positives | 0 |
| Detection rate | 99.8% |
| Precision | 100% |
| Infrastructure | 4× VPS 6 vCPU, 11 GB RAM |

### Fraud Scenarios

The 7 detection scenarios cover distinct fraud patterns:

| Scenario | Pattern Detected |
|----------|-----------------|
| Account Drain | Slow balance depletion over many cycles |
| Amount Anomaly | Transactions outside historical distribution |
| Velocity Burst | Unusual transaction frequency |
| Geo Hop | Impossible geographic transitions |
| Round Trip | Circular money flows between accounts |
| Structuring | Amounts split to stay below reporting thresholds |
| Dormant Wake | Sudden activity on long-dormant accounts |

Each scenario uses sina_mem to maintain per-account state across cycles.

## Throughput

| Configuration | Accounts | Scenarios/min | Infrastructure |
|---------------|----------|---------------|----------------|
| Remote target | 100,000 | 17,000+ | 4× VPS 6 vCPU |
| Local target | — | 40,000+ | 1× VPS 6 vCPU |
| Fast path | — | 70,000–81,000 | 1× VPS 6 vCPU |

The bottleneck is GoPhantom's execution engine, not the network. See [gophantom-benchmarks](https://github.com/sinainsight/gophantom-benchmarks) for detailed measurements.

## Key Differentiators

| Feature | SinaApp | Traditional CEP |
|---------|---------|----------------|
| Memory span | Days to weeks (sina_mem) | Minutes to hours (sliding window) |
| Analysis unit | Per-account lifecycle | Per-transaction |
| Weak signal detection | Yes (accumulation) | No (threshold-based) |
| Deployment | On-premise only | Varies |
| Data exfiltration | None — all data stays in-house | Varies |
| Agents on monitored systems | None | Often required |

## Deployment

SinaApp is deployed **on-premise only**, inside the customer's information system. No data leaves the customer's perimeter. There is no SaaS model.

- **PAN and IBAN anonymisation** at the source
- **NIS2, DORA, GDPR, PCI DSS** compliant by design
- Complements existing CEP engines and fraud tools

## Links

- [Live POC dashboard](https://fraud-poc.sinainsight.eu/) — Continuous fraud detection demo
- [GoPhantom benchmarks](https://github.com/sinainsight/gophantom-benchmarks) — Throughput measurements
- [lib_sina_crypt](https://github.com/sinainsight/lib-sina-crypt) — Encryption library (sina_mem storage)
- [SINAINSIGHT](https://www.sinainsight.pro) — Company website
- [llms.txt](https://www.sinainsight.pro/llms-full.txt) — Full technical reference for AI agents

## License

Documentation: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
SinaApp is proprietary software by SINAINSIGHT SAS.
