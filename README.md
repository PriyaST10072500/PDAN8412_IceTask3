# 🌍 Global Situation Dashboard


## 📌 What Was Built

The **Global Situation Dashboard** now includes three locally-running AI models that analyze live global event data, including:

* 🌍 Earthquakes
* 🌋 Volcanic activity
* 💻 Cybersecurity incidents
* 🔐 CVEs
* 🦠 Ransomware
* ✈️ Aircraft
* 🚢 Maritime activity
* 🌐 Other global events

All AI models run entirely on the local machine.

| Model       | Type             | How It Works                                                |
| ----------- | ---------------- | ----------------------------------------------------------- |
| **Model A** | Deterministic    | Always produces the same output for the same input          |
| **Model B** | Probabilistic    | Uses sampling, so repeated runs can differ                  |
| **Model C** | Tencent R3-Skill | Two-stage skill router using a bi-encoder and cross-encoder |



# ⚠️ Issues Faced and Solutions

## 🔴 Problem 1 - Model Size

The originally suggested `llama3.1` model is **4.7 GB**, which was too large for a standard laptop.

This pushed memory usage past **5 GB**, causing:

* The system to freeze
* Requests to time out after 5+ minutes
* The dashboard to become unusable

### ✅ Solution

Four models were tested to find one that was both capable and memory efficient.

| Attempt | Model            |        Size | Result                                           |
| ------- | ---------------- | ----------: | ------------------------------------------------ |
| 1st     | `llama3.1`       |      4.7 GB | System froze and requests timed out              |
| 2nd     | `phi3:mini`      |      3.9 GB | Still too heavy                                  |
| 3rd     | `tinyllama`      |     ~600 MB | Fast but incoherent and hallucinated Python code |
| 4th     | **`qwen2:0.5b`** | **~500 MB** | Fast enough, coherent, and fits in memory        |

The result demonstrated that using the biggest model is not necessarily the best approach. It is better to find the **smallest model capable of completing the required task**.



## 🔴 Problem 2 - Large Event Objects

The dashboard's event objects contained `__threeObjPoint`, which included massive Three.js geometry information such as:

* Cylinders
* Materials
* Matrices
* UUIDs

When `JSON.stringify()` was used, this information was included in the prompt.

This resulted in the model receiving thousands of tokens of irrelevant 3D rendering data instead of useful intelligence.

### ✅ Solution

A `cleanEvent()` function was added to strip the Three.js internals before sending the data to the AI.

It:

* Removes `__threeObjPoint`
* Removes `__threeObjLabel`
* Removes `__threeObjDot`
* Truncates `details` to 200 characters
* Limits arrays to 5 items per category
* Keeps only:

  * `title`
  * `type`
  * `location`
  * `time`
  * `details`
  * `lat`
  * `lng`
  * `size`
  * `color`

This reduced the prompt size by approximately **80%** and eliminated the model's tendency to describe JSON structures instead of analyzing them.



## 🔴 Problem 3 - CPU Inference Time

With CPU-only inference, even the 500 MB model can take **30-60 seconds** to respond.

The default fetch timeout therefore terminated requests before the model had finished.

### ✅ Solution

The following timeout and response-length controls were implemented:

* **Frontend:** 240-second `AbortController` timeout
* **Backend:** 300-second timeout for Ollama calls
* **Backend R3-Skill:** Handled directly by Flask
* **`num_predict: 512`:** Caps the response length



## 🔴 Problem 4 - R3-Skill Memory Overhead

Tencent R3-Skill requires two separate models:

* `tencent/R3-embedding-0.6b`
* `tencent/R3-rerank-0.6b`

Each requires approximately **2.4 GB** of memory.

Running these alongside Ollama and the dashboard created a significant memory requirement.

### ✅ Solution - Lightweight R3 Implementation

A streamlined Flask microservice was built that:

* Loads `tencent/R3-embedding-0.6b` and `tencent/R3-rerank-0.6b` once at startup
* Runs on a dedicated port (`5051`)
* Uses PyTorch
* Returns skill names rather than paragraphs
* Provides extremely fast inference
* Can be killed independently if memory becomes critical



# 🏗️ Architecture

```text
┌─────────────┐      ┌─────────────┐      ┌─────────────────┐
│  Browser    │─────▶│  Node.js    │─────▶│  Python Service │
│  (React)    │◀─────│  (port 5050)│◀─────│  (port 5051)    │
└─────────────┘      └─────────────┘      └─────────────────┘
                            │                     │
                            ▼                     ▼
                     ┌─────────────┐      ┌─────────────┐
                     │  Ollama     │      │  R3-Skill   │
                     │  qwen2:0.5b │      │  PyTorch    │
                     │  (port 11434)│      │  (2 models) │
                     └─────────────┘      └─────────────┘
```

## 🔄 Four Processes

Four processes run simultaneously:

### 1. R3-Skill Microservice

```bash
python server/r3_server.py
```

**Port:** `5051`

### 2. Ollama

```bash
ollama run qwen2:0.5b
```

**Port:** `11434`

### 3. Node.js API Gateway

```bash
node server/index.js
```

**Port:** `5050`

### 4. Vite Dashboard

```bash
npm run dev
```

**Port:** `5173`



# 🤖 Model Details

## 🔵 Model A - Deterministic

**Endpoint:**

```text
POST /api/assistant/deterministic
```

### Configuration

```javascript
options: {
  temperature: 0,
  top_k: 1,
  seed: 42,
  num_predict: 512
}
```

### What This Means

At `temperature: 0`, the model's probability distribution collapses to a single point.

Combined with `top_k: 1`, there is no choice between different tokens. The model selects the highest-probability token at every step.

The fixed `seed: 42` ensures reproducibility.

Therefore:

```text
Same prompt
     ↓
Same token sequence
     ↓
Same output
```

### Use Case

* Consistent risk scoring
* Repeatable intelligence reports
* Automated alerting where variance is unacceptable



## 🟣 Model B - Probabilistic

**Endpoint:**

```text
POST /api/assistant/probabilistic
```

### Configuration

```javascript
options: {
  temperature: 0.8,
  top_p: 0.9,
  top_k: 40,
  num_predict: 512
}
```

No seed is used, so the random number generation varies naturally between runs.

### What This Means

`temperature: 0.8` allows lower-probability tokens to become viable.

`top_p: 0.9` implements nucleus sampling by limiting sampling to tokens whose cumulative probability reaches 0.9.

`top_k: 40` limits the candidates considered to the top 40 tokens.

Therefore:

```text
Same prompt
     ↓
Different token sampling
     ↓
Different wording / emphasis
     ↓
Different output
```

### Use Case

* Brainstorming threat hypotheses
* Generating diverse analyst perspectives
* Creative summarization



## 🟢 Model C - Tencent R3-Skill

**Endpoint:**

```text
POST /api/r3/route
```

The request is proxied to the Python service running on port `5051`.

### What It Is

R3-Skill is not a chatbot.

It is a **retrieval system** that matches an event description against a library of predefined response skills and returns the best match.

### Stage 1 - Recall

**Bi-Encoder**

`R3-Embedding` converts the query and skills into dense vectors.

Cosine similarity is then used to quickly recall the most likely candidates.

### Stage 2 - Rerank

**Cross-Encoder**

`R3-Reranker` takes the query and each candidate skill as a joint input and calculates a more accurate relevance score.

The highest-scoring skill wins.

```text
Event Description
       ↓
R3-Embedding
       ↓
Top Candidate Skills
       ↓
R3-Reranker
       ↓
Best Matching Skill
```



# 📚 Skill Library

The skill library is stored in:

```text
server/skills.jsonl
```

The available skills are:

| Skill                  | Description                                  |
| ---------------------- | -------------------------------------------- |
| `disaster-response`    | Earthquakes, tsunamis, volcanic eruptions    |
| `cyber-alert`          | CVEs, ransomware, data breaches              |
| `infrastructure-check` | Internet outages, power, transportation      |
| `maritime-redirect`    | Reroute vessels from hazardous zones         |
| `aircraft-grounding`   | No-fly orders around dangerous airspace      |
| `space-debris-warn`    | Geomagnetic storms, orbital collisions       |
| `threat-intel-brief`   | Correlated threats, emerging attack patterns |
| `public-health-alert`  | Biological threats, pandemics                |

### Example

```text
Query:
"earthquake in Japan magnitude 7"

Result:
disaster-response
Confidence: 77.50

Runner-up:
space-debris-warn (77.50)
aircraft-grounding (77.50)
```

### Use Case

R3-Skill provides **automated triage**.

When an event occurs, it can identify which response playbook should be activated.



# ⚙️ How to Run

## 📋 Prerequisites

* Node.js 20+
* Python 3.10+ with pip
* Ollama installed locally
* ~4 GB free RAM for peak usage



## 1️⃣ Install Dependencies

### Frontend / Dashboard

```bash
npm install
```

### Python / R3-Skill

```bash
pip install torch transformers flask flask-cors numpy
```



## 2️⃣ Pull the LLM

```bash
ollama pull qwen2:0.5b
```

If more RAM is available, `qwen2:0.5b` can be replaced with a larger model in:

```text
server/index.js
```

The code is model-agnostic.



## 3️⃣ Start the Four Services

### Terminal 1 - Ollama

```bash
ollama run qwen2:0.5b
```

### Terminal 2 - R3-Skill Python Service

```bash
python server/r3_server.py
```

This will download approximately **4.8 GB** of R3 model weights on the first run.

### Terminal 3 - Node.js Backend

```bash
node server/index.js
```

### Terminal 4 - Vite Dashboard

```bash
npm run dev
```



## 4️⃣ Open the Dashboard

Navigate to:

```text
http://localhost:5173
```



# 📡 API Endpoints

| Endpoint                       | Method | Body                    | Description                   |
| ------------------------------ | ------ | ----------------------- | ----------------------------- |
| `/api/health`                  | GET    | `-`                     | Check if server is alive      |
| `/api/assistant`               | POST   | `{question, dashboard}` | Original assistant (balanced) |
| `/api/assistant/deterministic` | POST   | `{question, dashboard}` | Model A - deterministic       |
| `/api/assistant/probabilistic` | POST   | `{question, dashboard}` | Model B - probabilistic       |
| `/api/r3/route`                | POST   | `{query}`               | Model C - R3-Skill router     |



# 📊 Deterministic vs. Probabilistic

| Aspect                | Model A - Deterministic | Model B - Probabilistic        |
| --------------------- | ----------------------- | ------------------------------ |
| **Decoding strategy** | Greedy                  | Nucleus sampling               |
| **Temperature**       | 0                       | 0.8                            |
| **Top-k**             | 1                       | 40                             |
| **Top-p**             | N/A                     | 0.9                            |
| **Seed**              | Fixed (42)              | None                           |
| **Output behaviour**  | Identical every run     | Varies in wording and emphasis |
| **Trade-off**         | Boring but reliable     | Insightful but inconsistent    |

## 🧪 How It Was Proved

The same question was asked through both models:

```text
"What is the global risk level?"
```

The deterministic model returned rigid, nearly identical structured reports on repeated runs.

The probabilistic model changed its:

* Phrasing
* Threat emphasis
* Analysis ordering

between runs using the same dashboard snapshot.

This demonstrated the difference between deterministic and probabilistic model behaviour.



# 📁 Files Added / Modified

| File                  | Change                                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `server/index.js`     | Added `/api/assistant/deterministic`, `/api/assistant/probabilistic`, and `/api/r3/route` endpoints                |
| `server/r3_server.py` | **New** - Flask microservice running Tencent R3-Skill                                                              |
| `server/skills.jsonl` | **New** - Skill library for R3-Skill routing                                                                       |
| `src/App.jsx`         | Added three AI buttons - Original, Deterministic, Probabilistic - plus R3-Skill Router panel and data sanitization |



# 📝 Notes

* **All models run locally.** No OpenAI, Claude, or cloud AI APIs are used.
* The only external calls are for live data feeds such as USGS earthquakes, NASA EONET, NVD CVEs, and other live event data.
* **R3-Skill was the stretch goal.** The brief required investigating whether the model could be installed and implemented.
* R3-Skill was successfully implemented despite requiring PyTorch and approximately **4.8 GB of model weights**.
* **Memory management was the dominant challenge.**
* The progression from `llama3.1` → `phi3:mini` → `tinyllama` → `qwen2:0.5b` demonstrates that model selection is a resource-constrained optimization problem.
* **Data sanitization was critical.** Removing `__threeObjPoint` and truncating descriptions prevented the model from wasting its context window on 3D geometry metadata instead of threat intelligence.
