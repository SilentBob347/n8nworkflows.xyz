Answer HR policy questions in Telegram with Groq, Gemini and Google Drive

https://n8nworkflows.xyz/workflows/answer-hr-policy-questions-in-telegram-with-groq--gemini-and-google-drive-20873


# Answer HR policy questions in Telegram with Groq, Gemini and Google Drive

### 1. Workflow Overview

This workflow implements a retrieval-augmented generation (RAG) HR policy assistant accessible via Telegram. It indexes an HR policy document from Google Drive into an in-memory vector store using Google Gemini embeddings. When an employee sends a message to the Telegram bot, an AI agent utilizes a Groq chat model, conversation memory, and the vector store to search for relevant policy passages and generate context-aware answers.

The workflow is divided into two primary logical blocks:

- **1.1 Knowledge Base Ingestion:** Manually triggered pipeline that downloads the HR policy PDF from Google Drive, splits and processes the text, generates embeddings via Google Gemini, and loads the data into an in-memory vector store.
- **1.2 Telegram RAG Assistant:** Automatically triggered pipeline that listens for incoming Telegram messages, evaluates the queries using an AI agent powered by Groq, retrieves context using Gemini embeddings and the vector store, maintains per-chat history via a memory buffer, and replies to the user on Telegram.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Knowledge Base Ingestion

##### Overview
This block handles the manual initialization and updating of the vector database by fetching the HR policy document from Google Drive, chunking it, generating vector embeddings with Google Gemini, and storing the chunks in memory.

##### Nodes Involved
- When Manually Triggered
- Download PDF from Google Drive
- Load PDF Data
- Generate Gemini Embeddings
- Store in Vector Database

##### Node Details

- **When Manually Triggered**
  - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger Node)
  - *Configuration Choices:* Initiates execution on-demand.
  - *Input and Output Connections:* Output connects to *Download PDF from Google Drive*.
  - *Edge Cases:* None; execution is strictly manual.

- **Download PDF from Google Drive**
  - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Action Node)
  - *Configuration Choices:* Operation set to `download`, targeting a selected file ID via list mode.
  - *Input and Output Connections:* Input from *When Manually Triggered*; output connects to *Store in Vector Database* (Main data connection).
  - *Edge Cases:* Fails if Google Drive OAuth credentials are invalid, file permissions are missing, or the targeted file is deleted.

- **Load PDF Data**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` (AI Data Loader)
  - *Configuration Choices:* Data type configured as binary.
  - *Input and Output Connections:* Connects to *Store in Vector Database* via `ai_document`.
  - *Edge Cases:* Fails if the downloaded binary data is corrupted, password-protected, or not a valid PDF format.

- **Generate Gemini Embeddings**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` (AI Embedding Model)
  - *Configuration Choices:* Uses default Google Gemini embedding configuration.
  - *Input and Output Connections:* Connects to *Store in Vector Database* via `ai_embedding`.
  - *Edge Cases:* API rate limits, invalid Google Gemini API keys, or temporary Google API outages.

- **Store in Vector Database**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.vectorStoreInMemory` (AI Vector Store)
  - *Configuration Choices:* Mode set to `insert`, memory key configured as `vector_store_key`.
  - *Input and Output Connections:* Main input from *Download PDF from Google Drive*; AI document input from *Load PDF Data*; AI embedding input from *Generate Gemini Embeddings*.
  - *Edge Cases:* In-memory storage is ephemeral; data is cleared when n8n restarts, requiring a manual re-run.

---

#### Block 1.2: Telegram RAG Assistant

##### Overview
This block intercepts incoming Telegram queries, processes them through an LLM agent with conversation memory and document retrieval tools, and sends a grounded response back to the user.

##### Nodes Involved
- When Message on Telegram
- HR Q&A Agent
- Groq Chatbot Model
- Memory Buffer for Chat
- HR Policy Vector Storage
- Retrieve Gemini Embeddings
- Send Reply on Telegram

##### Node Details

- **When Message on Telegram**
  - *Type and Technical Role:* `n8n-nodes-base.telegramTrigger` (Trigger Node)
  - *Configuration Choices:* Listens for `message` updates.
  - *Input and Output Connections:* Output connects to *HR Q&A Agent*.
  - *Edge Cases:* Webhook configuration issues or expired Telegram bot tokens.

- **HR Q&A Agent**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
  - *Configuration Choices:* Prompt type set to define; input text mapped to `{{ $json.message.text }}`; custom system message enforces role constraints, mandatory tool usage, grounding rules, maximum length (200 words), language matching, and plain text formatting.
  - *Input and Output Connections:* Main input from *When Message on Telegram*; language model input from *Groq Chatbot Model*; memory input from *Memory Buffer for Chat*; tool input from *HR Policy Vector Storage*; main output connects to *Send Reply on Telegram*.
  - *Edge Cases:* Agent loop limits, context window exhaustion, or hallucination if tool search returns empty results.

- **Groq Chatbot Model**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (AI Language Model)
  - *Configuration Choices:* Model configured to `openai/gpt-oss-20b`.
  - *Input and Output Connections:* Connects to *HR Q&A Agent* via `ai_languageModel`.
  - *Edge Cases:* Invalid Groq API keys, rate limits, or service unavailability.

- **Memory Buffer for Chat**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` (AI Memory)
  - *Configuration Choices:* Session key/ID type configured as custom key using `{{ $('When Message on Telegram').item.json.message.chat.id }}`.
  - *Input and Output Connections:* Connects to *HR Q&A Agent* via `ai_memory`.
  - *Edge Cases:* Memory build-up during long-running sessions.

- **HR Policy Vector Storage**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.vectorStoreInMemory` (AI Vector Store)
  - *Configuration Choices:* Mode set to `retrieve-as-tool`, memory key set to `vector_store_key`, tool description explains search scope (leave, working hours, benefits, conduct, etc.).
  - *Input and Output Connections:* Connects to *HR Q&A Agent* via `ai_tool`; embedding input connected from *Retrieve Gemini Embeddings*.
  - *Edge Cases:* Fails if the vector store has not been populated by the ingestion block (e.g., after an n8n restart).

- **Retrieve Gemini Embeddings**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` (AI Embedding Model)
  - *Configuration Choices:* Default configuration for retrieval embedding.
  - *Input and Output Connections:* Connects to *HR Policy Vector Storage* via `ai_embedding`.
  - *Edge Cases:* API connectivity issues or invalid credentials.

- **Send Reply on Telegram**
  - *Type and Technical Role:* `n8n-nodes-base.telegram` (Action Node)
  - *Configuration Choices:* Text output mapped to `{{ $json.output }}`, chat ID mapped to `{{ $('When Message on Telegram').item.json.message.chat.id }}`.
  - *Input and Output Connections:* Input from *HR Q&A Agent*.
  - *Edge Cases:* Telegram API errors, blocked bots, or messaging limits if users send requests too rapidly.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Overview & Setup Instructions | None | None | ## Answer HR policy questions in Telegram with RAG (Groq, Gemini embeddings, Google Drive)<br><br>### How it works<br><br>This workflow builds a retrieval-augmented HR policy assistant for Telegram. A manual ingestion path downloads an HR policy PDF from Google Drive, parses it, creates Gemini embeddings, and inserts the content into an in-memory vector store. When a Telegram message arrives, an AI agent uses Groq, chat memory, and the policy vector store to answer the employee’s question, then sends the response back to Telegram.<br><br>### Setup steps<br><br>- Configure Google Drive credentials and point the download node to the HR policy PDF file.<br>- Configure Google Gemini credentials for both embedding nodes used during insert and retrieval.<br>- Configure Groq credentials for the chat model used by the HR Policy Agent.<br>- Configure Telegram bot credentials for the trigger and reply nodes, including the correct chat/message fields.<br>- Run the manual knowledge-base loading path before answering questions so the in-memory vector store is populated.<br><br>### Customization<br><br>Swap the Google Drive file for another policy document, adjust the agent prompt and Groq model, tune PDF loading/chunking behavior, or replace the in-memory vector store with a persistent vector database if the index must survive workflow restarts. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Ingestion Documentation | None | None | ## Download policy PDF<br><br>Starts the knowledge-base setup manually and retrieves the HR policy PDF from Google Drive. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Vectorization Documentation | None | None | ## Build vector knowledge base<br><br>Loads the PDF content, generates Gemini embeddings, and inserts the embedded policy chunks into the in-memory vector store. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Telegram Trigger Documentation | None | None | ## Receive Telegram question<br><br>Listens for incoming Telegram messages that contain HR policy questions. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Agent RAG Documentation | None | None | ## Answer with RAG agent<br><br>Uses the HR Policy Agent with Groq, chat memory, the policy vector store, and Gemini retrieval embeddings to produce a context-aware answer. |
| **Sticky Note5** | `n8n-nodes-base.stickyNote` | Telegram Reply Documentation | None | None | ## Send Telegram reply<br><br>Sends the generated HR policy answer back to the Telegram conversation. |
| **When Manually Triggered** | `n8n-nodes-base.manualTrigger` | Manual Trigger | None | Download PDF from Google Drive | |
| **Download PDF from Google Drive** | `n8n-nodes-base.googleDrive` | File Download | When Manually Triggered | Store in Vector Database | ## Download policy PDF<br><br>Starts the knowledge-base setup manually and retrieves the HR policy PDF from Google Drive. |
| **Store in Vector Database** | `@n8n/n8n-nodes-langchain.vectorStoreInMemory` | Vector Database Insert | Download PDF from Google Drive, Load PDF Data, Generate Gemini Embeddings | None | ## Build vector knowledge base<br><br>Loads the PDF content, generates Gemini embeddings, and inserts the embedded policy chunks into the in-memory vector store. |
| **Generate Gemini Embeddings** | `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` | Embedding Generator | None | Store in Vector Database | ## Build vector knowledge base<br><br>Loads the PDF content, generates Gemini embeddings, and inserts the embedded policy chunks into the in-memory vector store. |
| **Load PDF Data** | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | Document Parser | None | Store in Vector Database | ## Build vector knowledge base<br><br>Loads the PDF content, generates Gemini embeddings, and inserts the embedded policy chunks into the in-memory vector store. |
| **When Message on Telegram** | `n8n-nodes-base.telegramTrigger` | Telegram Trigger | None | HR Q&A Agent | ## Receive Telegram question<br><br>Listens for incoming Telegram messages that contain HR policy questions. |
| **HR Q&A Agent** | `@n8n/n8n-nodes-langchain.agent` | RAG AI Agent | When Message on Telegram, Groq Chatbot Model, Memory Buffer for Chat, HR Policy Vector Storage | Send Reply on Telegram | ## Answer with RAG agent<br><br>Uses the HR Policy Agent with Groq, chat memory, the policy vector store, and Gemini retrieval embeddings to produce a context-aware answer. |
| **Send Reply on Telegram** | `n8n-nodes-base.telegram` | Telegram Response | HR Q&A Agent | None | ## Send Telegram reply<br><br>Sends the generated HR policy answer back to the Telegram conversation. |
| **Groq Chatbot Model** | `@n8n/n8n-nodes-langchain.lmChatGroq` | LLM Chat Model | None | HR Q&A Agent | ## Answer with RAG agent<br><br>Uses the HR Policy Agent with Groq, chat memory, the policy vector store, and Gemini retrieval embeddings to produce a context-aware answer. |
| **Memory Buffer for Chat** | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Chat Memory Buffer | None | HR Q&A Agent | ## Answer with RAG agent<br><br>Uses the HR Policy Agent with Groq, chat memory, the policy vector store, and Gemini retrieval embeddings to produce a context-aware answer. |
| **HR Policy Vector Storage** | `@n8n/n8n-nodes-langchain.vectorStoreInMemory` | Vector Store Tool | Retrieve Gemini Embeddings | HR Q&A Agent | ## Answer with RAG agent<br><br>Uses the HR Policy Agent with Groq, chat memory, the policy vector store, and Gemini retrieval embeddings to produce a context-aware answer. |
| **Retrieve Gemini Embeddings** | `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` | Embedding Generator | None | HR Policy Vector Storage | ## Answer with RAG agent<br><br>Uses the HR Policy Agent with Groq, chat memory, the policy vector store, and Gemini retrieval embeddings to produce a context-aware answer. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to recreate the workflow manually in n8n:

1. **Create the Ingestion Trigger & Download Node:**
   - Add a **When Manually Triggered** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Download PDF from Google Drive** node (`n8n-nodes-base.googleDrive`). Configure credentials (Google Drive OAuth2), set the operation to `download`, and select the target HR policy PDF file ID. Connect *When Manually Triggered* to this node.

2. **Setup the Vector Store Insertion Pipeline:**
   - Add a **Store in Vector Database** node (`@n8n/n8n-nodes-langchain.vectorStoreInMemory`). Set mode to `insert` and set the memory key to `vector_store_key`. Connect the main output of *Download PDF from Google Drive* to the main input of this node.
   - Add a **Load PDF Data** node (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) with data type set to binary. Connect its `ai_document` output to the `ai_document` input of the *Store in Vector Database* node.
   - Add a **Generate Gemini Embeddings** node (`@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`) with valid Google Gemini credentials. Connect its `ai_embedding` output to the `ai_embedding` input of the *Store in Vector Database* node.

3. **Create the Telegram Trigger & Agent Core:**
   - Add a **When Message on Telegram** node (`n8n-nodes-base.telegramTrigger`) with Telegram credentials. Set updates to listen for `message`.
   - Add an **HR Q&A Agent** node (`@n8n/n8n-nodes-langchain.agent`). Set prompt type to define, text input to `={{ $json.message.text }}`, and configure the system message:
     > You are an HR policy assistant. Employees ask you questions about the company's HR policies via Telegram.
     > Rules:
     > - Always search the HR policy knowledge base tool before answering.
     > - Answer only from the information found in the knowledge base. If the answer is not there, say so and suggest contacting the HR team.
     > - Keep the answer clear and practical, maximum 200 words.
     > - Reply in the same language the user writes in.
     > - Use plain text without Markdown tables.
   - Connect the main output of *When Message on Telegram* to the main input of the *HR Q&A Agent*.

4. **Attach Models, Memory, and Tools to the Agent:**
   - Add a **Groq Chatbot Model** node (`@n8n/n8n-nodes-langchain.lmChatGroq`) with Groq credentials. Set the model to `openai/gpt-oss-20b`. Connect its `ai_languageModel` output to the `ai_languageModel` input of the *HR Q&A Agent*.
   - Add a **Memory Buffer for Chat** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`). Set session ID type to custom key with session key `={{ $('When Message on Telegram').item.json.message.chat.id }}`. Connect its `ai_memory` output to the `ai_memory` input of the *HR Q&A Agent*.
   - Add a second **HR Policy Vector Storage** node (`@n8n/n8n-nodes-langchain.vectorStoreInMemory`). Set mode to `retrieve-as-tool`, memory key to `vector_store_key`, and provide a clear tool description: `Search the company HR policy document. Use it for every question about leave, working hours, benefits, conduct, recruitment, salaries or any other HR topic.` Connect its `ai_tool` output to the `ai_tool` input of the *HR Q&A Agent*.
   - Add a **Retrieve Gemini Embeddings** node (`@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`) with Google Gemini credentials. Connect its `ai_embedding` output to the `ai_embedding` input of the *HR Policy Vector Storage* node.

5. **Configure the Reply Output:**
   - Add a **Send Reply on Telegram** node (`n8n-nodes-base.telegram`) using the same Telegram credentials. Set text to `={{ $json.output }}` and chat ID to `={{ $('When Message on Telegram').item.json.message.chat.id }}`. Connect the main output of the *HR Q&A Agent* to this node.

6. **Initialization:**
   - Run the manual trigger once to populate the in-memory vector store before testing the Telegram bot.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| The text provided originates exclusively from an automated n8n workflow, respecting content policies and utilizing safe, public data. | n8n Workflow Automation Reference |
| In-memory vector store limitation: The vector index is volatile and clears upon n8n restarts. Re-running the manual ingestion trigger is required after a restart unless swapped for persistent storage. | n8n Vector Store Documentation |