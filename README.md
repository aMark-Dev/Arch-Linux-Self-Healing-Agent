# ALSHA (Arch Linux Self-Healing Agent)

ALSHA is a lightweight CLI tool designed to diagnose and suggest fixes for Arch Linux system errors. It operates **100% locally**, leveraging Small Language Models (SLMs) and a Retrieval-Augmented Generation (RAG) architecture to map system logs to Arch Wiki documentation.

> **⚠️ DISCLAIMER**
> This is a **Proof of Concept (PoC)** developed strictly for educational purposes to study local RAG architectures and SLMs. There is no financial interest involved. Use suggestions at your own risk and always verify commands before execution.

---

## Architecture

Unlike generic chatbots, ALSHA implements a **Multi-Stage Reasoning** pipeline tailored for system administration. It prioritizes context preservation (via Parent-Child chunking) and low resource consumption (running on consumer CPUs).

### The "Middle Path" Strategy
We avoid the operational complexity of Docker/Microservices while solving the accuracy issues of naive RAG.

1.  **Triage:** The agent scans `journald` binary logs to isolate the root error string, ignoring noise.
2.  **Retrieval (Parent-Child):** We index small text chunks for search precision but retrieve full parent sections to provide the LLM with complete context (e.g., configuration files, caveats).
3.  **Refinement:** A local Cross-Encoder (`FlashRank`) re-ranks the vector search results to filter out irrelevant hits before they reach the context window.
4.  **Solver:** A quantized **Qwen 2.5 3B** model generates a structured JSON remediation plan (Reasoning + Command + Risk Level).

---

## Tech Stack & Implementation Details

ALSHA is built on a modern Python stack optimized for local inference.

| Component | Technology | Reasoning |
| :--- | :--- | :--- |
| **Model** | `Qwen 2.5 3B Instruct` | Best-in-class reasoning and coding capabilities under 4GB VRAM (`q4_k_m` quantization). |
| **Inference** | `llama-cpp-python` | Native GGUF support with hardware acceleration (Metal/CUDA) and JSON grammar constraint. |
| **Vector DB** | `ChromaDB` | Simple local persistence. *(Roadmap: Migration to `SQLite-vec` for single-file portability).* |
| **Reranker** | `FlashRank` | CPU-optimized Cross-Encoder (~40MB) to fix vector search fuzziness. |
| **Ingestion** | `html2text` + `LangChain` | Parses local HTML (`/usr/share/doc/arch-wiki`) into hierarchical markdown. |
| **Logs** | `systemd-python` | Direct C-binding access to binary journal logs (faster and safer than `subprocess`). |
| **CLI** | `Typer` + `Rich` | Type-safe commands and production-grade terminal output. |

### Key Libraries & Methods

*   **`llama-cpp-python`**: Used for the `Llama` class. We specifically rely on the `n_gpu_layers` parameter for hardware offloading and `response_format` to enforce valid JSON output.
*   **`langchain`**: Utilizes `MarkdownHeaderTextSplitter` for context-aware chunking and `ParentDocumentRetriever` to link granular search vectors to broader document contexts.
*   **`chromadb`**: Uses `PersistentClient` to store vectors locally without running a docker container.
*   **`systemd-python`**: Uses `journal.Reader()` with `seek_tail()` to efficiently read only the latest boot logs.
*   **`pydantic`**: Defines the `CommandSuggestion` schema to validate the LLM's output structure.

---

## Getting Started

### Prerequisites
*   Arch Linux
*   Python 3.11+
*   Package `arch-wiki-docs` installed (`sudo pacman -S arch-wiki-docs`)
*   ~4GB RAM available for the model

### Installation

1.  **Clone the repository:**
    ```
    git clone https://github.com/aMark-Dev/Arch-Linux-Self-Healing-Agent.git
    cd alsha
    ```

2.  **Set up the environment:**
    ```
    python -m venv.venv 
    source.venv/bin/activate
    pip install -r requirements.txt
    ```

3.  **Download the Model:**
    Download `qwen2.5-3b-instruct-q4_k_m.gguf` from HuggingFace and place it in the `./models` directory.

4.  **Ingest Knowledge Base:**
    Parses your local Arch Wiki HTML docs and builds the vector index (takes ~2-5 mins).
    ```python -m alsha.main ingest```

### Usage

Run diagnostics on a specific service or let it scan the latest logs:

```python -m alsha.main fix --service NetworkManager```

### What happens next?

* ALSHA detects if you have a GPU and configures n_gpu_layers automatically.

* It scans logs for the last boot cycle.

* Retrieves and re-ranks relevant Wiki articles.

* Proposes a fix with a reasoning explanation and risk assessment.

---

### Contribution
This is a learning project. If you want to optimize the Prompt Engineering, add support for new log sources, or help switch the backend to SQLite-vec, feel free to open a PR. No financial interest involved.
