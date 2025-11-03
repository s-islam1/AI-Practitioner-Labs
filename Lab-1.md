# 🧠 Local AI Agent Workshop — Windows 11 Setup Guide

> **Run and customize your own AI agent locally** using Ollama, Python, Docker, and Open WebUI — with live web search and RAG (Retrieval-Augmented Generation) integration.

---

## 🚀 Overview

This hands-on workshop walks you step-by-step through:
1. Installing **Ollama** and running local LLMs  
2. Setting up **Python**, **Docker**, and **Open WebUI**  
3. Enabling **live web search** via Tavily API  
4. Implementing **RAG** with LangChain + Ollama  
5. (Optional) RAG integration inside **Open WebUI**

---

## 🧩 1. Install Ollama Locally

### 1.1 Download & Install
➡️ [https://ollama.com/download](https://ollama.com/download)

Run the Windows installer, then open **Command Prompt** and verify:
```bash
ollama --version
````

### 1.2 Pull a Small Model

```bash
ollama pull gemma3:1b
```

> 💡 For better quality (but slower):
> `ollama pull mistral`

### 1.3 Quick Test

```bash
ollama run gemma3:1b
# Ask something, then press Ctrl + C to exit
```

---

## ⚙️ 2. Install Python, Docker Desktop, and Open WebUI

### 2.1 Python (with PATH)

Download from [python.org](https://www.python.org/downloads/windows/)
✅ Check **“Add Python to PATH”** during setup.

Verify installation:

```bash
python --version
pip --version
```

### 2.2 Docker Desktop

Download from [docker.com](https://www.docker.com/products/docker-desktop/)
Launch it and confirm it shows **Running**.

### 2.3 Run Open WebUI (via Docker)

Run in **PowerShell** or **Command Prompt**:

```bash
docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=http://host.docker.internal:11434 -e ENABLE_ADMIN=true --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

Then open [http://localhost:3000](http://localhost:3000) → create a local account.

In **Settings → Models**, set:

```
OLLAMA Base URL: http://host.docker.internal:11434
```

Pick **gemma3:1b** as your default model.

> ✅ **Optional (persistent data)**
>
> ```bash
> docker stop open-webui
> docker rm open-webui
> docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=http://host.docker.internal:11434 -e ENABLE_ADMIN=true -v %USERPROFILE%\openwebui_data:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
> ```

---

## 🌐 3. Enable Web Search (Tavily API)

### 3.1 Get API Key

Create an account at [https://app.tavily.com](https://app.tavily.com) → copy your **API key**.

### 3.2 Configure in Open WebUI

In **Settings → Admin → Tools / Integrations → Web Search**, choose:

```
Provider = Tavily
API Key = <your_key_here>
```

Toggle 🌐 **Web Search ON** in chat for live results.

> 💡 To use only your **local Knowledge Base**, turn **Web Search OFF**.

---

## 📚 4. Implement RAG (Retrieval-Augmented Generation)

### 4.1 Install Python Libraries

```bash
pip install -U langchain langchain-core langchain-community langchain-text-splitters langchain-ollama chromadb pypdf tiktoken
```

### 4.2 Pull an Embedding Model

```bash
ollama pull nomic-embed-text
# Alternatives: mxbai-embed-large, snowflake-arctic-embed
```

### 4.3 Prepare Folder

Create a folder and add PDFs/TXT/MD files:

```
C:\Users\<YOUR_USERNAME>\Desktop\rag_demo\
```

### 4.4 Create the RAG Script

Save as:

```
C:\Users\<YOUR_USERNAME>\Downloads\rag.py
```

```python
import os
from langchain_ollama import OllamaLLM, OllamaEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1) Load documents
docs_dir = r"C:\Users\<YOUR_USERNAME>\Desktop\rag_demo"
docs = []
for name in os.listdir(docs_dir):
    path = os.path.join(docs_dir, name)
    if name.lower().endswith(".pdf"):
        docs.extend(PyPDFLoader(path).load())
    elif name.lower().endswith((".txt", ".md")):
        docs.extend(TextLoader(path, encoding="utf-8").load())

# 2) Split into chunks
splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=120)
chunks = splitter.split_documents(docs)

# 3) Create embeddings + vector store
emb = OllamaEmbeddings(model="nomic-embed-text")
vectordb = Chroma.from_documents(chunks, emb, persist_directory=rf"{docs_dir}\chroma_db")

# 4) Retriever
retriever = vectordb.as_retriever(search_kwargs={"k": 4})

# 5) Local LLM via Ollama
llm = OllamaLLM(model="gemma3:1b")

def ask(query: str):
    rel_docs = retriever.invoke(query)
    context = "\n\n".join(
        f"[{d.metadata.get('source', 'unknown')}]\n{d.page_content[:1200]}"
        for d in rel_docs
    )
    prompt = f"""You are a helpful assistant. Use the CONTEXT to answer the QUESTION.
If not found, say you don't know.

CONTEXT:
{context}

QUESTION:
{query}
"""
    return llm.invoke(prompt)

if __name__ == "__main__":
    while True:
        q = input("\nAsk a question (or 'exit'): ").strip()
        if q.lower() in {"exit", "quit"}:
            break
        print("\n--- Answer ---")
        print(ask(q))
```

> 🔧 Replace `<YOUR_USERNAME>` with your Windows username.

### 4.5 Run It

```bash
python C:\Users\<YOUR_USERNAME>\Downloads\rag.py
```

Try:

```
Summarize the key points in the PDF about exam domains.
```

### 4.6 Troubleshooting

| Issue                                           | Solution                                                 |
| ----------------------------------------------- | -------------------------------------------------------- |
| `ModuleNotFoundError: langchain_text_splitters` | `pip install -U langchain-text-splitters`                |
| Deprecation warnings                            | You’re using the new, correct `langchain-ollama` imports |
| Chroma not saving                               | Auto-persists (no `persist()` needed)                    |
| Want better quality                             | Use `mistral` instead of `gemma3:1b`                     |
| Better recall                                   | Use `mxbai-embed-large` for embeddings                   |

---

## 🧠 5. (Optional) RAG in Open WebUI

No code needed!
Go to:
**Settings → Knowledge / Documents → Create Knowledge Base → Upload Files → Index.**

Then:

* Attach KB in chat
* Turn **Web Search OFF** for local answers
* Turn **ON** for hybrid (local + web)

---

## 🧰 Credits & Resources

* [Ollama Documentation](https://ollama.com)
* [Open WebUI GitHub](https://github.com/open-webui/open-webui)
* [LangChain Docs](https://python.langchain.com)
* [Tavily Web Search API](https://app.tavily.com)
* [Chroma DB](https://docs.trychroma.com)

---

## 🪄 Author Notes

Created by **Shahpar Islam**
Assistant Professor, Cloud Computing – NOVA IET
🔗 [LinkedIn](https://www.linkedin.com/in/shahparislam) · [GitHub](https://github.com/s-islam1)

---
