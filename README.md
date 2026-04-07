# Crypto Data Pipeline — Kubernetes on AWS

A production-style containerized data pipeline that tracks live cryptocurrency prices for 6 assets every 30 minutes, persists data to AWS DynamoDB, computes risk metrics, and publishes an evolving 4-panel analytics dashboard to a public S3 website — all orchestrated by Kubernetes running on EC2.

**Live Dashboard:** `http://iss-tracking-data.s3-website-us-east-1.amazonaws.com/plot.png`

<!-- Before AWS teardown: replace the line above with the note below and add a final screenshot -->
<!-- > ⚠️ **Note:** AWS infrastructure has been shut down — the link above is no longer active. Final dashboard snapshot below. -->
<!-- ![Final Dashboard Snapshot](screenshots/final-dashboard.png) -->

---

## Architecture

```
CoinGecko API ──► Python App (Docker) ──► DynamoDB (crypto-tracking)
                        │                        │
                   K3S CronJob              fetch history
                  (every 30 min)                 │
                        │                  compute metrics
                   EC2 t3.large                  │
                  (us-east-1)             generate dashboard
                        │                        │
                   IAM Role ──────────────► S3 Static Website
                                               plot.png
                                               data.csv
```

---

## Tracked Assets

| Coin | Symbol | Category |
|------|--------|----------|
| Bitcoin | BTC | Blue chip |
| Ethereum | ETH | Blue chip |
| XRP | XRP | Blue chip |
| Solana | SOL | Blue chip |
| Bittensor | TAO | AI / emerging |
| BlockDAG | BDAG | Emerging |

---

## Dashboard Panels

The live dashboard updates every 30 minutes with 4 panels:

- **Return Since Tracking Started** — normalized % return per coin from first data point
- **Fastest Growing** — total return bar chart ranked best to worst
- **Rolling Volatility** — 6-period rolling standard deviation of log returns
- **Simplified Sharpe Ratio** — mean return / volatility, a basic risk/reward ranking

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Orchestration | Kubernetes (K3S) on AWS EC2 t3.large |
| Containerization | Docker, GHCR (GitHub Container Registry) |
| Scheduling | Kubernetes CronJob (*/30 * * * *) |
| Data Source | CoinGecko API (no key required) |
| Persistence | AWS DynamoDB (partition: coin_id, sort: timestamp) |
| Visualization | Python — matplotlib, seaborn, pandas, numpy |
| Storage / Hosting | AWS S3 Static Website |
| Auth | AWS IAM Role (no hardcoded credentials) |

---

## Repository Structure

```
├── crypto-tracker/          # Your data pipeline (main deliverable)
│   ├── app.py               # Pipeline script — fetch, store, plot, publish
│   ├── Dockerfile           # Container definition
│   └── requirements.txt     # Python dependencies
│
├── iss-reboost/             # Professor's sample pipeline (ISS tracker)
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── k8s/                     # Kubernetes manifests
│   ├── crypto-job.yaml      # CronJob for crypto tracker (every 30 min)
│   ├── iss-job.yaml         # CronJob for ISS tracker (every 15 min)
│   └── simple-job.yaml      # Test CronJob used during setup
│
├── screenshots/             # Dashboard snapshots over the 5-day run
│
├── docs/
│   └── project-instructions.md   # Original assignment README
│
└── README.md                # This file
```

---

## How It Works

Each pod run executes `app.py` which does the following in sequence:

1. Calls the CoinGecko `/simple/price` endpoint for all 6 coins
2. Writes price, market cap, 24hr volume, and 24hr change to DynamoDB
3. Queries the full price history from DynamoDB
4. Computes log returns, rolling volatility, and a simplified Sharpe ratio per coin
5. Generates a 4-panel matplotlib dashboard and uploads it to S3 as `plot.png`
6. Exports the full history as `data.csv` and uploads it to S3

The EC2 instance's IAM role grants S3 and DynamoDB access — no credentials appear anywhere in the code or YAML files.

---

## Key Concepts Demonstrated

**Kubernetes Secrets vs Plain Env Vars** — The S3 bucket name is passed as a plain environment variable in the CronJob YAML since it is not sensitive. API keys (if required) would be stored as Kubernetes Secrets and injected as env vars so the value never appears in any file on disk.

**How Pods Get AWS Permissions** — The EC2 instance has an IAM Role attached. When a pod runs, boto3 automatically discovers credentials from the instance metadata service (IMDS). No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` is needed anywhere.

**Sharpe Ratio** — A standard risk-adjusted return metric from quantitative finance. Here computed as `mean(log_returns) / std(log_returns)` per coin over the tracking window. Higher = better return per unit of risk.

**Log Returns** — `ln(P_t / P_{t-1})` — the standard way to measure price changes in finance because they are time-additive and symmetrical, unlike simple percentage returns.

---

## Setup (Reproducibility)

Full setup instructions are in [`docs/project-instructions.md`](docs/project-instructions.md). At a high level:

1. Create an S3 bucket with static website hosting enabled
2. Launch an EC2 t3.large (Ubuntu 24.04) with an IAM Role for S3 + DynamoDB
3. Install K3S: `curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644`
4. Create the DynamoDB table: `aws dynamodb create-table --table-name crypto-tracking ...`
5. Build and push the Docker image to GHCR
6. Apply the CronJob: `kubectl apply -f k8s/crypto-job.yaml`

---

## Canvas Quiz — Reflection Questions

**Which data source did you choose and why?**

I chose CoinGecko as my data source because I'm genuinely interested in the crypto space and where it's headed. I've always wanted to find a way to combine data science and quantitative methods with financial markets, and this project felt like a natural first step toward that. Being able to pull live price data, store it, and start building analytics around it — using cloud infrastructure on top of that — is exactly the kind of thing I want to keep building on after this class.

**What did you observe in the data — any patterns, spikes, or surprises over the tracking window?**

The biggest thing I noticed was what happened with BlockDAG (BDAG). It had a brief run of small gains early on, then experienced a pretty dramatic crash — roughly 70% down over the tracking window. You can see it clearly in the dashboard: it completely dominates the volatility chart and tanks the Sharpe ratio. The other five coins (BTC, ETH, XRP, SOL, TAO) stayed relatively flat or slightly negative in comparison, which almost made them look stable just because BDAG was so extreme. That contrast actually ended up being one of the more interesting things to look at in the dashboard.

**How do Kubernetes Secrets differ from plain environment variables, and why does that distinction matter?**

Plain environment variables are values you write directly into the manifest YAML — like I did with `S3_BUCKET` and `AWS_REGION` in my CronJob. That's fine for non-sensitive config because it shows up in the file and gets committed to the repo. Kubernetes Secrets are different — they're stored separately inside the cluster, base64-encoded, and injected into pods at runtime so the actual value never has to appear in any file on disk or in source control. The distinction matters because if you stored something like an API key or a database password as a plain env var, it would end up in your YAML, your repo history, and anywhere else that file lives. With a Secret, the sensitive value stays inside the cluster and you just reference it by name in the manifest.

**How do your CronJob pods get permission to read/write to AWS services without credentials appearing anywhere?**

The EC2 instance running the cluster has an IAM Role attached to it. That role has a policy — which we wrote out in JSON — that grants specific permissions to S3 and DynamoDB. When a pod runs on that instance, `boto3` automatically reaches out to the instance metadata service (IMDS) to retrieve temporary credentials scoped to that role. There's no `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` anywhere in the code or YAML — the permissions just flow through the role.

**One thing you would do differently if building this for a real production system.**

A couple things honestly. First, I'd make sure each pipeline has its own dedicated S3 bucket — in this project I ended up using the same bucket for both the ISS tracker and the crypto tracker, which worked but isn't clean. For a real system you'd want clear separation. Second, I'd move away from a static PNG dashboard toward something more interactive — something stakeholders could actually filter and explore rather than just a snapshot image. That'd probably cost more to host but it'd be a lot more useful in practice.

---

## Canvas Quiz — Graduate Questions

**1. If the ISS application were running at much higher frequency (hundreds of writes per minute), what changes would you make to the persistence strategy?**

At that kind of frequency, writing directly to DynamoDB on every pod run would start to create problems — you'd either hit throughput limits or rack up a lot of cost fast. I'd probably introduce a buffer layer, something like SQS or Kinesis, so writes get queued and processed in batches rather than one at a time. On the DynamoDB side I'd switch to on-demand capacity mode or set up provisioned capacity with auto-scaling so it can handle sudden spikes without throttling. I'd also want some kind of alerting — CloudWatch alarms or similar — so that if write failures or unusual patterns start showing up, the right people get notified before it becomes a bigger problem.

**2. Describe at least one way the orbital burn detection logic could produce a false positive, and how you would make it more robust.**

The current logic flags a burn whenever altitude increases by 1 km or more in a single 15-minute interval. One clear way that could fire incorrectly is sensor noise or a bad reading from the API — if the `wheretheiss.at` service returns an outlier value, the delta could easily cross the threshold even if nothing actually happened. Another scenario is atmospheric variation causing a brief, non-burn altitude fluctuation that just happens to hit the threshold. To make it more robust I'd require the altitude gain to be sustained across multiple consecutive readings rather than just one, and ideally cross-reference with velocity data — a real reboost should show a correlated increase in orbital velocity, not just altitude. Adding a rolling average and flagging only deviations above a certain number of standard deviations from recent history would help filter out noise as well.

**3. How does each CronJob pod get AWS permissions without credentials being passed into the container?**

The EC2 node running the cluster has an IAM Role attached at the instance level. When a pod starts, `boto3` automatically queries the AWS Instance Metadata Service (IMDS) at `169.254.169.254` to retrieve short-lived, rotating credentials scoped to that role. The pod never needs to know any access keys — it inherits permissions through the role assigned to the host node. This is the standard pattern for giving EC2-hosted workloads AWS access without hardcoding credentials anywhere.

**4. What are the partition key and sort key of the `iss-tracking` DynamoDB table, and why do they work here but might not elsewhere?**

The partition key is `satellite_id` and the sort key is `timestamp`. This works well for the ISS use case because there's only one satellite being tracked, so all records share the same partition key and the sort key lets you query by time range and get results back in order. The problem is that having a single partition key means all reads and writes go to one partition — what DynamoDB calls a "hot partition." For a single satellite running every 15 minutes that's totally fine, but if you scaled this to tracking hundreds of satellites with high-frequency writes, you'd hammer one partition and hit throughput limits. A better key design for a larger system might include something like a date prefix in the partition key to spread the load, or use a different primary key structure entirely depending on the access patterns.

---

## Author

Mauricio Torres — DS5220 Cloud Computing, Spring 2026
