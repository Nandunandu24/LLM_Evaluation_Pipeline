# LLM Evaluation Pipeline (AI Auditor)

## 📌 Project Overview
This repository contains a robust, production-ready evaluation pipeline designed to audit AI-User interactions in real-time. It acts as an automated "Judge" that assesses AI responses against a retrieved Context (RAG) for **Hallucination**, **Relevance**, and **Completeness**, while also tracking quantitative metrics like **Latency** and **Cost**.

The pipeline is built to be resilient, capable of recovering from corrupted log files and handling external API failures gracefully.

## ✨ Key Features
* **"Nuclear" Data Cleaning:** A multi-phase recovery system using Regex to extract valid chat/context data even from broken or malformed JSON files.
* **Real-Time LLM Auditing:** Uses Google Gemini (Flash/Pro) to score qualitative metrics on a 1-5 scale.
* **Metric Tracking:** Automatically calculates response latency and estimated token costs.
* **Resilience Logic:** Implements exponential backoff for rate limits (`429`) and model fallback switching for service outages (`503`/`404`).

---

## 🛠️ Local Setup Instructions

1.  **Clone the Repository:**
    ```bash
    git clone [https://github.com/your-username/llm-evaluation-pipeline.git](https://github.com/your-username/llm-evaluation-pipeline.git)
    cd llm-evaluation-pipeline
    ```

2.  **Install Dependencies:**
    You will need Python 3.9+ and the Google Generative AI library.
    ```bash
    pip install google-generativeai
    ```

3.  **Configure API Key:**
    Open the `main.py` file and replace the placeholder with your actual API key:
    ```python
    # main.py
    GOOGLE_API_KEY = "YOUR_ACTUAL_API_KEY_HERE"
    ```

4.  **Run the Pipeline:**
    ```bash
    python main.py
    ```
    *The script will generate simulated chat logs, clean them, run the evaluation, and output a `final_report.json` file.*

---

## 🏗️ Architecture

The pipeline follows a **"Fail-Safe RAG Evaluation"** architecture designed for high reliability.



1.  **Data Ingestion:** The system loads raw `chat_logs` and `context_vectors` (JSON).
2.  **Resurrection Layer (Cleaning):**
    * Instead of standard JSON parsing (which fails on malformed logs), the pipeline uses a **Regex-based extraction engine**.
    * It surgically identifies semantic fields (`"text": "..."`) to salvage usable data from corrupted files.
3.  **Quantitative Analysis:**
    * **Latency:** Calculated from timestamp deltas (ISO 8601).
    * **Cost:** computed via Token Count * Model Unit Price.
4.  **Qualitative Analysis (The Judge):**
    * Constructs a composite prompt: `Context + User Query + AI Response`.
    * Sends this to the LLM Judge to receive structured JSON scores (1-5) for Hallucination and Relevance.
5.  **Reporting:** Aggregates all metrics into a final `evaluation_report.json`.

---

## 🧠 Design Decisions: "Why this way?"

### 1. Why Regex for Data Cleaning?
Standard `json.loads()` is brittle. In real-world logging, files often get truncated, contain invalid escape characters, or mix Python/JSON syntax. A standard parser would crash the pipeline. My "Nuclear Cleaner" approach ensures the pipeline **always** has data to evaluate, essentially "resurrecting" logs that would otherwise be discarded.

### 2. Why "Force-Through" Retry Logic?
External LLM APIs are non-deterministic and prone to transient failures (Rate Limits, Server Overload). A simple script would fail on a `503 Service Unavailable`.
* **Solution:** I implemented an infinite retry loop with **exponential backoff**.
* **Model Switching:** If the primary model (`flash`) is down, the script automatically hot-swaps to a backup model (`pro`) to ensure the audit completes.

### 3. Why Single-Pass Evaluation?
Instead of making separate API calls for "Relevance" and "Hallucination," I combine them into a single prompt. This reduces **inference costs by 50%** and halves the latency compared to multi-call architectures.

---

## 🚀 Scaling Strategy (Millions of Conversations)

If running this script at scale (e.g., millions of daily conversations), the following optimizations ensure cost and latency remain minimal:

1.  **Asynchronous Architecture:**
    * Move the evaluation out of the user's critical path.
    * Use a message queue (e.g., **Kafka** or **RabbitMQ**) to buffer chat logs.
    * Process evaluations in the background via **Celery** workers, ensuring the user's chat experience is never slowed down by the audit.

2.  **Semantic Sampling (Smart Filtering):**
    * **Do not evaluate 100% of logs.** It is cost-prohibitive.
    * **Trigger-based Evaluation:** Only audit chats where:
        * User sentiment is Negative (detected via lightweight BERT).
        * The AI's internal confidence score is below a threshold (e.g., < 0.7).
        * The conversation topic involves "high-risk" categories (e.g., Medical, Financial).

3.  **Model Distillation:**
    * Replace the large, expensive "Generalist Judge" (e.g., Gemini Pro/GPT-4) with a **fine-tuned Small Language Model (SLM)** like Llama-3-8B or Gemini Nano.
    * Fine-tuning a small model specifically for "Hallucination Detection" can achieve similar accuracy at **1/10th the cost**.

## 📊 Sample Output (final_report.json)

```json
{
    "evaluation_id": "eval_auto_001",
    "timestamp": "2025-12-14T20:30:00",
    "quality_scores": {
        "relevance_score": 5,
        "completeness_score": 4,
        "factual_accuracy_score": 5,
        "reasoning": "The response is fully grounded in the provided context..."
    },
    "performance_metrics": {
        "latency_ms": 5000,
        "cost_usd": 0.00021
    }
}
https://colab.research.google.com/drive/19gYoZFgCc7jm8DrNrBUIvpxspVS25CYR?usp=sharing
