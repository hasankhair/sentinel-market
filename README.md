# 🛡️ SentinelMarket - BNB Builder Hack

**Agents that hire, pay and judge other agents.**

SentinelMarket is an autonomous risk-alert business on BNB Chain. A **Sentinel agent** sells crypto risk alerts to subscribers, hires specialist **worker agents** by on-chain reputation, locks payment in **escrow**, has a **verifier** score the result, and then releases or refunds the money and updates the worker's reputation.

It ships in two parts:

| Part                                     | What it is                                                                                                                                             | Status                                                     |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| **Interactive demo** (`demo/index.html`) | A single-file web app that simulates the whole agent economy, built for live pitches                                                                   | Working, fully simulated                                   |
| **Studio kit** (`studio-kit/`)           | A guide and TypeScript buyer that turn the demo into real agents on BSC testnet using [BNB Agent Studio](https://www.bnbchain.org/en/bnb-agent-studio) | Starter code, needs wiring (see [Known gaps](#known-gaps)) |

> ⚠️ The demo is a **simulation**. Outcomes are random, weighted by each worker's skill, and no real money moves. The Studio kit is the path to real on-chain behaviour. Use **BSC testnet** and throwaway wallets only.

---

## How it works

```
Subscriber ──x402──▶ Sentinel agent ──ERC-8183 job + escrow──▶ Worker agents
                          │                                         │
                          └───────── ERC-8004 identity + reputation ┘
                                      BSC testnet (chain 97)
```

1. A subscriber pays the Sentinel for an alert (rug-pull scan, LP rebalance quote, or whale-wallet digest).
2. The Sentinel picks the best-reputation worker it can afford, or you pick one yourself.
3. The worker's fee is locked in escrow while the worker delivers.
4. A verifier scores the deliverable against a strictness threshold.
5. Pass: funds are released and reputation rises. Fail: funds are refunded and reputation drops.
6. The Sentinel charges subscribers about 40% more than it pays workers, so it funds itself.

### Workers in the demo

| Worker            | Behaviour                                                    |
| ----------------- | ------------------------------------------------------------ |
| **ScanBot** 🔍    | Reliable, highest reputation                                 |
| **QuoteFox** 🦊   | Decent quality, higher price                                 |
| **CheapGuess** 🎲 | Cheap but often fails, so reputation has something to punish |

### Mapping the demo to real protocols

| Demo step                       | Real implementation                                               |
| ------------------------------- | ----------------------------------------------------------------- |
| Subscriber pays                 | x402 / B402 payment face on the Sentinel                          |
| Worker identity and reputation  | ERC-8004 registry                                                 |
| Escrow lock, release and refund | ERC-8183 job (`createJob` → `registerJob` → `fund` → `settle`)    |
| Worker agents                   | Seller agents built with BNB Agent Studio, served over A2A or MCP |
| Self-funding                    | `bag budget enable` (wallet auto-refill)                          |

---

## Demo features

**Core simulation**

- Live pipeline animation: coin trail, per-stage effects, success and failure bursts, progress line
- Treasury, escrow, job counts and success rate with animated counters
- Click a worker to hire it directly, or let the Sentinel pick the best one
- Verifier strictness slider, speed control (1×/2×/4×), auto-pilot, reset

**Pitch tools**

- 🎬 **90-second auto-demo** with narrated captions
- 🎤 **Audience requests**: type a token address or request and a real job runs
- 🚨 **Rug pull caught** and ⛔ **Blacklist bad worker** scenarios
- 🖥 **Presentation mode** (press `P`) for big screens
- 🔊 Sound effects for payment, rejection and pipeline steps (mutable)
- QR and share card for judges

**Data and realism**

- 🏆 Leaderboard, treasury-over-time chart, recent jobs
- 🔎 BscScan-style transaction explorer with click-through detail pages (simulated, chain 97)
- Architecture diagram of the full stack

**Assistant chatbot** (💬, bottom-right)

- Answers questions about the project using live app state
- Can act for you: hire agents, pick tasks, set strictness, run several jobs, start the demo, run scenarios
- Judge Q\&A knowledge (business model, trust, security, real vs simulated)
- Optional 📢 narration mode that comments on each job result

**Design**

- BNB yellow (`#F0B90B`) branding with a switchable light and dark theme
- Responsive layout for phones, keyboard accessible, respects reduced-motion settings

---

## Run the demo

The demo is one self-contained HTML file with no build step.

```bash
git clone https://github.com/<you>/sentinelmarket.git
cd sentinelmarket/demo
python3 -m http.server 8080      # or any static server
# open http://localhost:8080
```

You can also host it on **GitHub Pages**: Settings → Pages → deploy from the `main` branch.

### Notes on hosting

- The **chatbot** uses the Claude artifact runtime and only appears when the page is opened inside claude.ai. Everywhere else it stays hidden and the rest of the app works unchanged.
- The **QR code** loads a small library from cdnjs. If it can't load, the share card shows the link only.
- Update the `SU` constant near the end of `index.html` to your own public URL so the QR and Copy button point to the right place.
- Sound starts after your first click, because browsers block autoplay audio.

---

## Go real with BNB Agent Studio

The `studio-kit/` folder contains everything to replace the simulation with real agents on **BSC testnet**.

### Requirements

- Node.js 22+, pnpm 10, Bun 1.3+ (for deploy)
- Claude Code or Cursor
- A throwaway wallet funded from the [testnet faucet](https://www.bnbchain.org/en/testnet-faucet)

### 1. Install Studio

```bash
npm install --global @bnbagent/studio-cli
bag skills install        # then reload Claude Code / Cursor
```

### 2. Build the three worker agents

In an empty folder per agent, open Claude Code or Cursor and paste one prompt each.

**ScanBot**

```
/bnbagent-studio Create a BNB Chain seller agent named scanbot that sells rug-pull
risk scans for BEP-20 tokens (returns a 0-100 risk score with cited on-chain
evidence). Start on bsc-testnet, use ERC-8183, expose A2A and MCP, default wallet
and LLM, store deliverables on IPFS. Price 0.03 per job.
```

**QuoteFox**

```
/bnbagent-studio Create a seller agent named quotefox that sells PancakeSwap v3 LP
rebalance quotes. bsc-testnet, ERC-8183, A2A + MCP, default wallet/LLM, IPFS. Price 0.04.
```

**CheapGuess**

```
/bnbagent-studio Create a seller agent named cheapguess that sells quick whale-wallet
movement digests. bsc-testnet, ERC-8183, A2A, default wallet/LLM, IPFS. Price 0.02.
```

Implement `runWork` in each `app/agent/src/sellerCore.ts`, then:

```bash
bag doctor && bag dev                      # local test (A2A on :9000)
bag deploy prepare
bag deploy --provider bnb                  # 48h managed testnet trial
bag deploy verify --provider bnb           # confirm ERC-8004 identity
```

Copy each agent's wallet address, public URL and agent id into `studio-kit/sentinel/workers.json`.

### 3. Run the Sentinel (buyer, verifier, treasury)

```bash
cd studio-kit/sentinel
pnpm install
cp .env.example .env.local     # set WALLET_PASSWORD; PRIVATE_KEY on first run only
pnpm start "Rug-pull scan on 0x..."
```

The Sentinel uses [`@bnbagent/sdk`](https://github.com/bnb-chain/bnbagent-sdk) to pick a worker by reputation, create and fund an ERC-8183 job, wait for the deliverable, verify it, then settle or reject and update reputation.

### Security

- Studio manages keys and on-chain funds and is under active development. **Testnet and throwaway wallets only.**
- Never paste wallet passwords into chat or pass them as command-line arguments. Use hidden prompts or `.studio/.env.local`.
- Approve each shell command in your IDE. Do not grant a blanket `bag:*` permission, since the CLI includes deploy and payment commands.
- Studio never auto-settles a buyer's job. Settlement is a deliberate action in the Sentinel script.

---

## Known gaps

These are called out honestly so you know what to verify before relying on the Studio path:

- **Seller transport:** `negotiate`, `notify_funded` and deliverable fetch are isolated in an adapter section of `sentinel.ts`. The exact A2A JSON-RPC shapes come from each agent's card at `/.well-known/agent.json`. Check them against your deployed agent.
- **Reject and dispute:** this call isn't wired. See the "dispute-reject" example in `typescript/examples/client/` of the [BNB Agent SDK](https://github.com/bnb-chain/bnbagent-sdk) and implement `rejectJob()`.
- **Reputation:** the kit keeps it in a local `workers.json`. Swap in the ERC-8004 reputation registry if your SDK version exposes it.
- **Verifier:** `verify()` is a simple heuristic. Replace it with schema checks, evidence-link validation or a second-opinion model.
- **Demo explorer:** transaction hashes, addresses, and function and event names are illustrative, not real chain data.
- The Studio kit has not been run end-to-end against a live deployment.

---

## Repository layout

```
sentinelmarket/
├── README.md
├── demo/
│   └── index.html              # single-file interactive demo
└── studio-kit/
    ├── README.md               # step-by-step Studio setup
    └── sentinel/
        ├── package.json
        ├── sentinel.ts         # buyer, verifier, treasury
        ├── workers.json        # worker registry and reputation
        └── .env.example
```

## Tech

- **Demo:** vanilla HTML, CSS and JavaScript in one file; Web Audio for sound; inline SVG for the diagram and chart
- **On-chain path:** [BNB Agent Studio](https://docs.bnbchain.org/developer-kit/bnbchain-studio/), [BNB Agent SDK](https://docs.bnbchain.org/developer-kit/bnbagent-sdk/) (TypeScript), ERC-8004, ERC-8183, x402, A2A, MCP, BSC testnet

## Resources

- [BNB Agent Studio](https://www.bnbchain.org/en/bnb-agent-studio)
- [Studio quickstart](https://docs.bnbchain.org/developer-kit/bnbchain-studio/quickstart/)
- [BNB Agent SDK quickstart (TypeScript)](https://docs.bnbchain.org/developer-kit/bnbagent-sdk/quickstart-typescript/)
- [BNB Chain testnet faucet](https://www.bnbchain.org/en/testnet-faucet)

## License

Add your license here (for example MIT) before publishing.
