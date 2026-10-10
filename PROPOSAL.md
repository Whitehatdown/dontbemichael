# Eight changes I'd like to make to Don't Be Michael

Hi Vivek,

I spent some proper time with Don't Be Michael: the code, the design docs, TODOS.md and the tests. I really like what you've built. The docs are honest about what isn't done yet, and the product has a clear idea of who it's for. That honesty is what made it easy to see where the biggest wins are.

Below are the eight changes I think matter most. Each one says what I found, what I'd build and why it matters to the owner using the office. I've put file references in so you can check my reading of the code.

## 1. Make "Michael asks before spending money" a rule the app enforces

**What I found.** The README promises that spending money, deleting things and anything public comes to the owner first. Today that promise lives in the prompt, not the code. Every agent starts in Claude's `bypassPermissions` mode by default: `autoMode` is `true` in `src/main/config.ts:487`, which turns into that flag in `src/shared/agentProvider.ts:193`. TODOS.md says it plainly: "Asks before sending is instructed, not enforced." QuickBooks is the same: once Can make changes is on, any change goes through, at any amount.

**What I'd build.** The default deny hook that TODOS.md already plans, and one Approvals inbox for every action that leaves the computer, not just email. Each connector would get a separate read and write switch, and the owner could set money limits in plain words, like "anything over £200 comes to me".

**Why it matters.** An agent only has to refund the wrong customer once, and the owner stops trusting the whole office. Trust is the product.

## 2. Treat anything that comes from outside as untrusted

**What I found.** Text from inbound emails, webhook bodies and documents reaches the agents with nothing marking it as untrusted. Webhook bodies go straight to Michael as a request (`src/main/index.ts:2474`). Michael is the only agent with no sandbox (`src/shared/folderAccess.ts:63`). He can read every agent's mail token, because each one is written into the shared hive folder (`src/main/index.ts:3719`). So one well crafted email that gets passed up to Michael could end up sending mail from any mailbox in the business. On Windows there's no shell sandbox at all, and no agent has limits on which websites it can reach.

**What I'd build.**
* Once an agent has read outside content, its sending and writing tools need the owner's approval until the owner clears it.
* Sandbox Michael like everyone else.
* Move the mail tokens out of the hive folder.
* Sign hook events, so no agent can pretend to be Michael.
* An allowlist of websites agents can reach.
* A log of every outside action that agents can't edit.

**Why it matters.** These agents hold the business's email, books and customer list. One bad email shouldn't be enough to send any of it out of the business.

## 3. Keep the office open when the laptop is shut

**What I found.** Schedules are timers inside the app. If the app is closed or the Mac is asleep, nothing runs. Missed jobs catch up when the computer wakes, with no limit on how late they can be, so a 9am customer email can go out at 7pm. Webhooks come in through a tunnel to the laptop, and the only alerts are desktop notifications.

**What I'd build.** A way to run the office on a small machine that's always on, or in the cloud, plus a phone app for the Needs you board, with push notifications and approve or reject from your phone. A useful first step on its own: catch up rules ("skip this job if it's more than two hours late"), and an email or push notification when a Needs you card appears.

**Why it matters.** The owners this is built for are on the shop floor all day. That's exactly when the office should be working for them.

## 4. Talk to Claude through its API, not its keyboard

**What I found.** The app gives agents work by typing text into their terminal, waiting 140 ms and pressing Enter (`nudgeWorker`, `src/main/index.ts:6886`). It works out what each agent is doing by matching text on the screen, such as "Do you want to proceed". If Claude Code changes its wording or layout, delivery quietly breaks.

**What I'd build.** Run agents through the Claude Agent SDK, or Claude Code's headless JSON mode. We'd get structured events, clear errors when a plan limit is hit, usage for every turn and safe retries. It's also the foundation for change 3. It can be moved over one agent at a time, behind a switch.

**Why it matters.** Right now, a Claude Code release we don't control can silently stop the whole office.

## 5. Handle plan limits gracefully and show costs in pounds

**What I found.** When a smaller Claude plan hits its limit, nothing in the app notices, and the office just stops midday with no explanation (P1 in TODOS.md, not built yet). Costs only appear as token counts. The circuit breaker skips Michael entirely (`src/main/index.ts:1745`), and spending caps are off by default.

**What I'd build.**
* Cost per job and per agent, in the owner's own currency.
* How much of the plan has been used, with a warning before the limit.
* A clear "the office is on a break until 3pm" banner, with Michael doing the most important jobs first when budget runs low.
* A daily budget for people paying with an API key, with Michael included.

**Why it matters.** An owner who can't see what the office costs can't decide whether it's worth it. An office that stops without saying why looks broken.

## 6. Show the owner what the office got done

**What I found.** There's no record of what got done. The morning report and the first success confirmation are both P1 in TODOS.md and not built. Scheduled results arrive as Report cards that disappear once the owner clicks Got it.

**What I'd build.** A record of outcomes that agents add to as they finish work: emails answered, invoices chased, money recovered, reviews replied to, hours saved. On top of that, a short morning summary and a weekly report.

**Why it matters.** This is the number the owner sees on a Monday morning, and it's what makes them keep paying.

## 7. Stop the app slowing down as the business grows

**What I found.**
* Every routed batch of messages and every task change runs a `git commit` synchronously on the app's main thread (`src/main/hive.ts:3915`). If git is busy, it retries with a sleep that blocks the whole app (`src/main/hive.ts:3997`).
* Every 8 seconds, `refreshOwnerRequests` rescans Michael's inbox, done, outbox and sent folders, all the way back to a fixed date (`src/main/index.ts:6976`, `src/shared/ownerRequests.ts:33`). Its cache is never trimmed (`src/main/hive.ts:2542`).
* Nothing ever prunes old messages or `log.jsonl`.

In the first month, none of this is noticeable. By month six, the window will stutter.

**What I'd build.** Move git and the folder scans off the main thread (a `utilityProcess`), and batch commits instead of making one per change. Keep message details in the SQLite database the app already has (`src/main/db.ts`), so owner requests become one query over open items rather than a folder scan. Archive old messages after a set number of days.

**Why it matters.** The owners who use it most get the worst experience. That's the opposite of what we want.

## 8. Stop losing work

**What I found.**
* `tasks.json` is written by both the app and the agents. The app reads it, merges and writes it back with a plain `writeFileSync` (`src/main/hive.ts:2302` to `2309`), while agents edit the same file with their own tools. Nothing locks it, so whoever writes last wins, and a card can vanish or jump back to an old state.
* The queue of messages waiting to be typed to agents lives in the window's state, saved to browser storage plus a mirror file. The header of `src/main/db.ts` says the message queue mirror was deliberately not built. A crash in the middle of a delivery can lose a message or deliver it twice.

**What I'd build.** Make SQLite, owned by the main process, the one place tasks and the queue are stored. Agents would change tasks through a small tool rather than editing the file directly. Every delivery would carry an id, so a retry can't run the same message twice. In the meantime, switch the tasks write to `atomicWriteJson`, which already sits right next to it (`src/main/hive.ts:3889`), so at least a crash can't leave a half written file.

**Why it matters.** The task board is the owner's picture of their business. If it quietly loses things, they'll never trust it again.

## About me, and why I'm writing this

This is also my job application. I found Don't Be Michael on Wellfound, went through the repo and decided the best way to apply was to show you how I'd contribute. I'd love to join as a paid contributor and build the changes above.

I like to build and ship fast. That means small pull requests that each do one thing, come with tests and sit behind the switches the codebase already uses. Here's what I'd ship first, while we agree on which of the eight bigger changes you want first:

1. **Make the 12 message limit actually stop loops.** Nothing ever increases a message's `hops`, so the check at `src/main/hive.ts:1948` never fires, and two agents can bounce messages back and forth, burning the owner's plan.
2. **Make tests block merges.** Today the only test job is the Windows one, and it's allowed to fail (`.github/workflows/ci.yml:52`). There's no test job on macOS, and `release.yml` doesn't run tests at all.
3. **Move each agent's mail token out of the shared hive folder.** This is already P1 in TODOS.md.
4. **Make the tasks write atomic.** This is the small first half of change 8.

Thanks for reading. I'd be glad to talk it through, or simply open the first pull request so you can see how I work.

## A bit more about me

I'm Shrirang Savale, a GenAI researcher from IIIT Nagpur. I currently build AI products at Lief Care in London, so I know the UK small business world this office is meant for.

Most of my recent work is close to what Don't Be Michael does. I've shipped MCP based agent integrations using LangGraph, the Claude Agent SDK and Mastra, which is why change 4 is the one I'm most excited about. I've built real time voice and conversational AI systems with speaker diarisation, including turning ambient speech into structured data. I've also wired in observability tooling such as Langfuse and LangSmith, which is the same kind of work as the cost and outcome views in changes 5 and 6.

On the machine learning side, I've fine tuned LLMs with LoRA and QLoRA and built LSTM and RNN time series forecasting. I've also worked with diffusion models, vision language models and classical ML, all the way through to MLOps: experiment tracking, model versioning and CI/CD pipelines on AWS SageMaker and Azure. I bring both research depth and hands on engineering, and I think that mix is a good fit for this role.

### Skills

* **LLMs and AI:** LangGraph, LlamaIndex, LangChain, RAG, Graph RAG, LLM fine tuning (LoRA, QLoRA), MCP, Mastra
* **Vector databases and search:** Pinecone, Weaviate, ChromaDB, FAISS, pgvector, embedding pipelines
* **Cloud and MLOps:** AWS (SageMaker, Lambda, Airflow on MWAA, S3, EC2), GCP Vertex AI, Azure OpenAI, Docker, Kubernetes, CI/CD pipelines
* **Machine learning:** PyTorch, TensorFlow, Keras, Transformers, LSTMs, RNNs, NLP, K means clustering, IDM VTON
* **Data:** Python, SQL, Pandas, NumPy, Snowflake, MongoDB, Matplotlib, Seaborn, Tableau, Power BI
* **APIs:** REST, GraphQL, Twitter's OAuth 2 APIs, Arkham, Galxe, Infura

### Achievements

* Amazon ML Challenge 2025: rank 42 in an international competition
* Wunderfund AI Challenge 2026: rank 29
* Blaze International Quant Challenge by BlazeAI: winner in April 2024 and runner up in January 2024
* IIT BHU TechNEX'24 Stellar Analytics: national runner up
* IIIT Designathon 2023: national winner
* Smart India Hackathon 2023: finalist, representing IIIT Nagpur
* Contributed to Pet-Perfect's Hidden Gem of the Play Store award (2022 to 2023)
* Contributed to Ocky-Pocky, the number one kids' app on the Play Store in February and March 2023

### Links

* GitHub: [github.com/Whitehatdown](https://github.com/Whitehatdown)
* LinkedIn: [linkedin.com/in/shrirang-savale](https://www.linkedin.com/in/shrirang-savale)
* Résumé: [Google Drive](https://drive.google.com/file/d/1_OtRmvyXvkcW9U3sxujsw4NC2B4G4W8e/view?usp=sharing)

Best regards,

Shrirang Savale
