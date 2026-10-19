Split web pages into vector-ready chunks with Ragfeed

https://n8nworkflows.xyz/workflows/split-web-pages-into-vector-ready-chunks-with-ragfeed-20948


# Split web pages into vector-ready chunks with Ragfeed

### 1. Workflow Overview

This workflow is designed for developers building Retrieval-Augmented Generation (RAG) chatbots or knowledge bases who need to convert web pages into clean, token-bounded text chunks ready for vector embeddings and storage. Rather than handling raw HTML parsing directly within n8n, the workflow offloads the scraping and chunking process to the external Ragfeed API.

The execution logic is structured into a single linear sequence:
- **1.1 Input Initiation & Parameter Configuration:** Manually triggers the execution and defines the target webpage URL alongside the maximum token limit per chunk.
- **1.2 API Communication & Data Transformation:** Sends a authenticated POST request to the external scraping service and flattens the returned JSON array of text chunks into individual n8n execution items.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Initiation & Parameter Configuration
- **Overview:** Initializes the workflow execution on demand and prepares the necessary input variables (`url` and `maxTokens`) required by the downstream API request.
- **Nodes Involved:** 
  - `Run manually`
  - `Set page and chunk size`

- **Node Details:**
  - **Node Name:** `Run manually`
    - **Type and Technical Role:** `n8n-nodes-base.manualTrigger` (v1). Serves as the entry point for manual testing and execution.
    - **Configuration Choices:** Default settings (no parameters required).
    - **Key Expressions/Variables:** None.
    - **Input/Output Connections:** Output connects to `Set page and chunk size`.
    - **Edge Cases / Failure Types:** None (manual action).
  
  - **Node Name:** `Set page and chunk size`
    - **Type and Technical Role:** `n8n-nodes-base.set` (v3.4). Assigns data fields to the item data structure.
    - **Configuration Choices:** Configured to set string `url` (default: `https://en.wikipedia.org/wiki/Retrieval-augmented_generation`) and number `maxTokens` (default: `500`).
    - **Key Expressions/Variables:** Literal values assigned via UI parameters.
    - **Input/Output Connections:** Input from `Run manually`; output connects to `Fetch page as chunks`.
    - **Edge Cases / Failure Types:** Invalid URL strings will cause subsequent API requests to fail.

#### Block 1.2: API Communication & Data Transformation
- **Overview:** Communicates with the external scraping service via an HTTP POST request and splits the nested response array so that each content chunk becomes a distinct item for vector indexing.
- **Nodes Involved:**
  - `Fetch page as chunks`
  - `One item per chunk`

- **Node Details:**
  - **Node Name:** `Fetch page as chunks`
    - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2). Performs an HTTP POST request with generic header authentication.
    - **Configuration Choices:** 
      - Method: `POST`
      - URL: `https://api.ragfeed.de/v2/scrape`
      - Body Format: JSON
      - Authentication: Header Auth (`httpHeaderAuth`)
    - **Key Expressions/Variables:** 
      - JSON Body: 
        ```json
        ={
          "url": "{{ $json.url }}",
          "formats": [
            { "type": "chunks", "maxTokens": {{ $json.maxTokens }} }
          ]
        }
        ```
    - **Input/Output Connections:** Input from `Set page and chunk size`; output connects to `One item per chunk`.
    - **Edge Cases / Failure Types:** Authentication errors (invalid API key), network timeouts, rate limits (free plan constraints), or HTTP 4xx/5xx responses from the external API.

  - **Node Name:** `One item per chunk`
    - **Type and Technical Role:** `n8n-nodes-base.splitOut` (v1). Flattens an array field into individual n8n items.
    - **Configuration Choices:** 
      - Field to Split Out: `data.chunks`
    - **Key Expressions/Variables:** References the nested array path `data.chunks` from the previous HTTP response.
    - **Input/Output Connections:** Input from `Fetch page as chunks`; output ends the current workflow branch (intended for connection to embedding and vector store nodes).
    - **Edge Cases / Failure Types:** Expression failure if the API response does not return the expected `data.chunks` array structure (e.g., due to an empty scrape or API error).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run manually` | `n8n-nodes-base.manualTrigger` | Triggers the workflow execution on demand. | None | `Set page and chunk size` | Split web pages into vector-ready chunks with Ragfeed<br><br>### Who it's for<br>Anyone building a RAG chatbot or knowledge base in n8n who wants web pages as clean, ready-to-embed records instead of raw HTML.<br><br>### How it works<br>1. **Set page and chunk size** holds the URL and the maximum chunk size.<br>2. **Fetch page as chunks** calls the Ragfeed API (`POST /v2/scrape`, format `chunks`). Ragfeed renders the page, drops navigation, footers and cookie notices, and splits the main content along its headings.<br>3. **One item per chunk** turns the list into one n8n item per chunk.<br><br>Each chunk has a stable `id` (UUID from the normalized URL and position, so re-fetching the same page overwrites instead of duplicating), `text`, `embedText`, `tokens`, the heading path and flat `metadata` (URL, title, language, fetch time, `hash`).<br><br>### Requirements<br>A Ragfeed API key (starts with `sk-`). The free plan has 1,000 credits a month; one page costs 1 credit, chunks cost nothing extra. Sign-up and dashboard are in German: https://ragfeed.de/en<br><br>### How to customize<br>Connect **One item per chunk** to your embeddings and vector store nodes (Qdrant, Pinecone, Supabase, Postgres/pgvector). Use `id` as the point id and `embedText` as the text to embed. Change `maxTokens` (100 to 4,000) for smaller or larger chunks. |
| `Set page and chunk size` | `n8n-nodes-base.set` | Defines target URL and token threshold variables. | `Run manually` | `Fetch page as chunks` | Split web pages into vector-ready chunks with Ragfeed<br><br>### Who it's for<br>Anyone building a RAG chatbot or knowledge base in n8n who wants web pages as clean, ready-to-embed records instead of raw HTML.<br><br>### How it works<br>1. **Set page and chunk size** holds the URL and the maximum chunk size.<br>2. **Fetch page as chunks** calls the Ragfeed API (`POST /v2/scrape`, format `chunks`). Ragfeed renders the page, drops navigation, footers and cookie notices, and splits the main content along its headings.<br>3. **One item per chunk** turns the list into one n8n item per chunk.<br><br>Each chunk has a stable `id` (UUID from the normalized URL and position, so re-fetching the same page overwrites instead of duplicating), `text`, `embedText`, `tokens`, the heading path and flat `metadata` (URL, title, language, fetch time, `hash`).<br><br>### Requirements<br>A Ragfeed API key (starts with `sk-`). The free plan has 1,000 credits a month; one page costs 1 credit, chunks cost nothing extra. Sign-up and dashboard are in German: https://ragfeed.de/en<br><br>### How to customize<br>Connect **One item per chunk** to your embeddings and vector store nodes (Qdrant, Pinecone, Supabase, Postgres/pgvector). Use `id` as the point id and `embedText` as the text to embed. Change `maxTokens` (100 to 4,000) for smaller or larger chunks.<br><br>Setup<br>1. Create a credential **Header Auth**:<br>   - **Name:** `Authorization`<br>   - **Value:** `Bearer sk-...` (your Ragfeed API key)<br>2. Select it in **Fetch page as chunks**.<br>3. Put your page into **Set page and chunk size** and run the workflow.<br><br>The key lives in the credential, never in the node. |
| `Fetch page as chunks` | `n8n-nodes-base.httpRequest` | Calls the Ragfeed scraping API via POST request. | `Set page and chunk size` | `One item per chunk` | Split web pages into vector-ready chunks with Ragfeed<br><br>### Who it's for<br>Anyone building a RAG chatbot or knowledge base in n8n who wants web pages as clean, ready-to-embed records instead of raw HTML.<br><br>### How it works<br>1. **Set page and chunk size** holds the URL and the maximum chunk size.<br>2. **Fetch page as chunks** calls the Ragfeed API (`POST /v2/scrape`, format `chunks`). Ragfeed renders the page, drops navigation, footers and cookie notices, and splits the main content along its headings.<br>3. **One item per chunk** turns the list into one n8n item per chunk.<br><br>Each chunk has a stable `id` (UUID from the normalized URL and position, so re-fetching the same page overwrites instead of duplicating), `text`, `embedText`, `tokens`, the heading path and flat `metadata` (URL, title, language, fetch time, `hash`).<br><br>### Requirements<br>A Ragfeed API key (starts with `sk-`). The free plan has 1,000 credits a month; one page costs 1 credit, chunks cost nothing extra. Sign-up and dashboard are in German: https://ragfeed.de/en<br><br>### How to customize<br>Connect **One item per chunk** to your embeddings and vector store nodes (Qdrant, Pinecone, Supabase, Postgres/pgvector). Use `id` as the point id and `embedText` as the text to embed. Change `maxTokens` (100 to 4,000) for smaller or larger chunks.<br><br>Setup<br>1. Create a credential **Header Auth**:<br>   - **Name:** `Authorization`<br>   - **Value:** `Bearer sk-...` (your Ragfeed API key)<br>2. Select it in **Fetch page as chunks**.<br>3. Put your page into **Set page and chunk size** and run the workflow.<br><br>The key lives in the credential, never in the node.<br><br>### Fetch and split<br>Ragfeed returns `data.chunks`; **One item per chunk** splits the list. Connect your embedding model and vector store after it. |
| `One item per chunk` | `n8n-nodes-base.splitOut` | Flattens the chunks array into individual execution items. | `Fetch page as chunks` | None | Split web pages into vector-ready chunks with Ragfeed<br><br>### Who it's for<br>Anyone building a RAG chatbot or knowledge base in n8n who wants web pages as clean, ready-to-embed records instead of raw HTML.<br><br>### How it works<br>1. **Set page and chunk size** holds the URL and the maximum chunk size.<br>2. **Fetch page as chunks** calls the Ragfeed API (`POST /v2/scrape`, format `chunks`). Ragfeed renders the page, drops navigation, footers and cookie notices, and splits the main content along its headings.<br>3. **One item per chunk** turns the list into one n8n item per chunk.<br><br>Each chunk has a stable `id` (UUID from the normalized URL and position, so re-fetching the same page overwrites instead of duplicating), `text`, `embedText`, `tokens`, the heading path and flat `metadata` (URL, title, language, fetch time, `hash`).<br><br>### Requirements<br>A Ragfeed API key (starts with `sk-...`). The free plan has 1,000 credits a month; one page costs 1 credit, chunks cost nothing extra. Sign-up and dashboard are in German: https://ragfeed.de/en<br><br>### How to customize<br>Connect **One item per chunk** to your embeddings and vector store nodes (Qdrant, Pinecone, Supabase, Postgres/pgvector). Use `id` as the point id and `embedText` as the text to embed. Change `maxTokens` (100 to 4,000) for smaller or larger chunks.<br><br>### Fetch and split<br>Ragfeed service returns `data.chunks`; **One item per chunk** splits the list. Connect your embedding model and vector store after it. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Add a **Manual Chat/Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.
2. **Create and Configure the Set Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set page and chunk size`.
   - Configure parameters to add two assignments:
     - Field `url` (Type: `string`) = `https://en.wikipedia.org/wiki/Retrieval-augmented_generation`
     - Field `maxTokens` (Type: `number`) = `500`
   - Connect the output of `Run manually` to the input of `Set page and chunk size`.
3. **Configure Authentication:**
   - Create a new **Header Auth** credential in n8n.
   - Set the header Name to `Authorization` and the Value to `Bearer sk-...` (using your Ragfeed API key).
4. **Create and Configure the HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Fetch page as chunks`.
   - Set Method to `POST`.
   - Set URL to `https://api.ragfeed.de/v2/scrape`.
   - Select **Generic Credential Type** -> **Header Auth** and choose your Ragfeed credential.
   - Set **Specify Body** to `JSON` and provide the request body expression:
     ```json
     ={
       "url": "{{ $json.url }}",
       "formats": [
         { "type": "chunks", "maxTokens": {{ $json.maxTokens }} }
       ]
     }
     ```
   - Connect the output of `Set page and chunk size` to the input of `Fetch page as chunks`.
5. **Create and Configure the Split Out Node:**
   - Add a **Split Out** node (`n8n-nodes-base.splitOut`) named `One item per chunk`.
   - Set **Field to Split Out** to `data.chunks`.
   - Connect the output of `Fetch page as chunks` to the input of `One item per chunk`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ragfeed Sign-up and Dashboard (German interface) | [https://ragfeed.de/en](https://ragfeed.de/en) |
| API Pricing & Free Plan Constraints | Free plan includes 1,000 credits/month (1 page = 1 credit, chunks cost 0 extra) |
| Target Vector Store Integration | Ready to connect to Qdrant, Pinecone, Supabase, or Postgres/pgvector via `id` and `embedText` |