# Technical Design Document (TDD): Web3 Early Project & Memecoin Scout Bot

## 1. System Overview & Objectives
The objective is to architect a high-speed, real-time alerting system that identifies Web3 projects (both memecoins and utility) at the exact moment of their inception (e.g., contract deployment or first liquidity provision), effectively beating the Twitter timeline. Furthermore, the system must accurately filter projects to exclusively surface those originating from developers in Turkey, Switzerland, Italy, France, Germany, or Belgium.

## 2. System Architecture
The architecture relies on a decoupled, event-driven microservices model to ensure speed and fault tolerance. 

### 2.1 Component Diagram
1. **On-Chain Event Listener (The Vanguard)**: Subscribes to mempool and block events on key chains.
2. **Off-Chain Discovery Engine**: Crawls GitHub and Twitter for early signals.
3. **Data Enrichment Pipeline**: Correlates contract addresses with social and web footprints.
4. **Scoring & Classification Node**: Applies region heuristics and Meme vs. Utility logic.
5. **Database & Caching Layer**: Stores historical data and manages rate-limits.
6. **Alerting Microservice**: Pushes final, curated data to the user via Telegram/Discord.

## 3. Detailed Component Design

### 3.1 The Vanguard: On-Chain Event Listener
To beat the timeline, we cannot rely on manual curation. We must listen to the blockchain directly.
- **Networks**: Solana (Primary for Memecoins), Base, Ethereum, Arbitrum (Primary for Utility).
- **Tech Stack**: Rust or Go for maximum execution speed, connecting via WebSockets (WSS).
- **Triggers**:
  - **EVM**: Listen to `ContractCreation` events. Filter for standard ERC20 interfaces. Listen to Uniswap V2/V3 `PoolCreated` events.
  - **Solana**: Listen to Raydium/Orca `InitializeInstruction` events via Helius RPC.

### 3.2 Data Enrichment & Region Filtering (The Core Challenge)
Once a contract is found, the Enrichment Pipeline immediately searches for metadata (Website, Twitter, Telegram, GitHub) hidden in the contract code (verified source) or via automated search queries.

**Region Detection Heuristics (Turkey, Switzerland, Italy, France, Germany, Belgium):**
1. **Source Code Analysis**: Scrape verified contract code from Etherscan/Basescan. Look for non-English comments (e.g., French, German, Italian, Turkish).
2. **GitHub Metadata**: If a GitHub repo is linked or found via code similarity, hit the GitHub API to extract the user's `Location` field.
3. **Twitter API**: Scrape the linked Twitter profile's location and timezone data.
4. **WHOIS Data**: Query domain registration details for the project's website to find registrant country (often masked, but useful when public).
5. **Timezone Clustering (Fallback)**: Analyze the exact timestamp of the contract deployment and initial GitHub commits. If the activity consistently aligns with CET (Central European Time) or TRT (Turkey Time) working/waking hours, increase the region probability score.

### 3.3 Classification Engine: Memecoin vs. Utility
The system uses a two-pronged approach to categorize projects:

**A. On-Chain Footprint Analysis:**
- **Memecoin Indicators**: No complex logic, 100% of supply sent to Liquidity Pool, ownership renounced immediately, standard OpenZeppelin/SPL template, liquidity locked or burned.
- **Utility Indicators**: Presence of proxy patterns (upgradeable contracts), complex access control, timelocks, multiple interacting contracts (e.g., a token contract + a staking contract deployed simultaneously).

**B. Off-Chain NLP / LLM Analysis:**
- We will pipe the scraped website text and Twitter bio through a lightweight LLM (e.g., OpenAI `gpt-4o-mini`).
- **Prompt Logic**: Instruct the LLM to classify as `MEME` (driven by hype, animal themes, community focus) or `UTILITY` (DeFi, Infrastructure, Tooling, DAO).

### 3.4 Alerting & Bot Interface
- **Framework**: Node.js `telegraf` for Telegram or `discord.js` for Discord.
- **Data Delivery**: The bot will push real-time alerts containing:
  - Project Name / Ticker
  - Contract Address & Direct links to DEX Screeners (DexScreener, DEXTools)
  - Classification (Meme / Utility)
  - Region Match Confidence Score (e.g., "85% - Linked GitHub user is based in Berlin, Germany")
  - Social Links (Twitter, Website)
- **User Commands**:
  - `/subscribe [region] [type]` (e.g., `/subscribe turkey meme`)

## 4. Technology Stack & Infrastructure
- **Backend Languages**: Go (for on-chain listener), TypeScript/Node.js (for enrichment, APIs, and Bot).
- **Databases**: 
  - **PostgreSQL**: For persistent storage of contracts, user subscriptions, and analytical data.
  - **Redis**: For fast deduplication (ensuring we don't alert the same contract twice) and rate limiting.
- **External APIs**: 
  - **RPC Nodes**: QuickNode, Alchemy, Helius.
  - **Socials**: Twitter API v2, GitHub REST API, WHOIS lookup APIs.
  - **AI**: OpenAI API for text classification.
- **Deployment**: AWS ECS or Google Cloud Run for scalable, containerized deployments.

## 5. Verification & Testing Plan
1. **Historical Backtesting**: Run the region heuristics against known past projects from the target regions to calibrate the detection accuracy.
2. **Dry-Run Mode**: Deploy the listener without the alerting bot connected, dumping results into a private Slack/Discord channel to manually verify if the bot is actually catching projects "before the TL" and correctly filtering regions.
3. **Unit Testing**: Strict tests on the Smart Contract analysis logic to ensure we don't misclassify a complex utility token as a memecoin.
