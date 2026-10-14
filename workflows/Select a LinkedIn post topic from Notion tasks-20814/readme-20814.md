Select a LinkedIn post topic from Notion tasks

https://n8nworkflows.xyz/workflows/select-a-linkedin-post-topic-from-notion-tasks-20814


# Select a LinkedIn post topic from Notion tasks

### 1. Workflow Overview

This workflow is designed to automate the selection and reservation of a content topic for LinkedIn publication directly from a Notion task. It accepts a webhook trigger containing a Notion task page object, traverses its related relational databases (Projects, Knowledge bases, Taxonomies, and Annotations), identifies the first valid and unpublished combination, and updates both the original task and its parent tasks in Notion.

The logical blocks are organized as follows:
- **1.1 Input Reception:** Receives the incoming Notion task via an HTTP webhook payload.
- **1.2 Project and Taxonomy Filtering:** Fetches related Notion projects and their taxonomies, filtering specifically for projects designated as "Post".
- **1.3 Knowledge and Topic Selection:** Loads related knowledge records and taxonomies for the approved projects, filters for "Assunto" (Topic) types, and iterates through them to inspect their subitems and annotations recursively.
- **1.4 Verification and Notion Sync:** Evaluates candidate annotations for specific tags ("Post", "LinkedIn", and empty results), then updates the original task and its parent pages via the Notion API when a match is found.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception
- **Overview:** Captures the initial webhook request carrying the Notion task object and initiates the workflow execution.
- **Nodes Involved:**
  - `Receber tarefa`

- **Node Details:**
  - **Receber tarefa**
    - **Type and Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger)
    - **Configuration Choices:** Listens for incoming HTTP `POST` requests at a specific webhook path using the last node's result mode.
    - **Key Expressions / Variables:** Extracts the Notion task object from `body.tarefa`.
    - **Input / Output Connections:** Input: External HTTP request; Output: `Carregar projetos e taxonomias da tarefa`.
    - **Version-Specific Requirements:** Version 2.1.
    - **Edge Cases / Failure Types:** Invalid JSON payloads or malformed Notion schemas in the webhook body will cause downstream JavaScript steps to fail when property lookups return `undefined`.

---

#### Block 1.2: Project and Taxonomy Filtering
- **Overview:** Queries Notion to load related project relations and their associated taxonomies, filtering out any projects that do not carry the "Post" taxonomy tag.
- **Nodes Involved:**
  - `Carregar projetos e taxonomias da tarefa`
  - `Filtrar projetos do tipo Post`

- **Node Details:**
  - **Carregar projetos e taxonomias da tarefa**
    - **Type and Technical Role:** `n8n-nodes-base.executeWorkflow` (Sub-workflow execution node)
    - **Configuration Choices:** Calls the sub-workflow named "Enriquecer Página do Notion" using defined page and properties mapping to expand the `"🎯 Projetos"` relation and its nested `"🏷️ Taxonomia"` property.
    - **Key Expressions / Variables:** `={{ $json.body.tarefa }}` for the target page; dynamic JSON array specifying the property expansion structure.
    - **Input / Output Connections:** Input: `Receber tarefa`; Output: `Filtrar projetos do tipo Post`.
    - **Version-Specific Requirements:** Version 1.3. Sub-workflow ID: `hBvVpMuQo3608cVw`.
    - **Edge Cases / Failure Types:** Sub-workflow failure or invalid Notion API tokens will interrupt execution. If the task has no projects, the output list will be empty.
  - **Filtrar projetos do tipo Post**
    - **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript execution node)
    - **Configuration Choices:** Iterates through expanded project objects and checks nested taxonomies for a title matching `"Post"` (case-insensitive).
    - **Key Expressions / Variables:** Custom vanilla JavaScript inspecting `item.json["🎯 Projetos"]`.
    - **Input / Output Connections:** Input: `Carregar projetos e taxonomias da tarefa`; Output: `Carregar conhecimentos e taxonomias dos projetos`.
    - **Version-Specific Requirements:** Version 2.
    - **Edge Cases / Failure Types:** If no projects match the "Post" criteria, the array evaluates to empty, halting downstream branch progression.

---

#### Block 1.3: Knowledge and Topic Selection
- **Overview:** Expands approved projects to load associated knowledge bases, filters them for "Assunto" types, breaks them into individual items, and iterates through them in batches to locate valid publication annotations.
- **Nodes Involved:**
  - `Carregar conhecimentos e taxonomias dos projetos`
  - `Filtrar conhecimentos do tipo Assunto`
  - `Separar assuntos em itens`
  - `Percorrer assuntos candidatos`
  - `Carregar subitens e anotações do assunto`
  - `Selecionar primeiro assunto com anotação disponível para LinkedIn`

- **Node Details:**
  - **Carregar conhecimentos e taxonomias dos projetos**
    - **Type and Technical Role:** `n8n-nodes-base.executeWorkflow` (Sub-workflow execution node)
    - **Configuration Choices:** Invokes the "Enriquecer Página do Notion" sub-workflow to expand `"📚 Conhecimentos"` and its nested `"🏷️ Taxonomias"` property.
    - **Key Expressions / Variables:** `={{ $json }}` passing the current project item object.
    - **Input / Output Connections:** Input: `Filtrar projetos do tipo Post`; Output: `Filtrar conhecimentos do tipo Assunto`.
    - **Version-Specific Requirements:** Version 1.3.
    - **Edge Cases / Failure Types:** API rate limits or network errors when expanding deep Notion relations.
  - **Filtrar conhecimentos do tipo Assunto**
    - **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript execution node)
    - **Configuration Choices:** Filters knowledge items to retain only those possessing a taxonomy titled `"Assunto"` (case-insensitive).
    - **Key Expressions / Variables:** Iterates over `item.json["📚 Conhecimentos"]`.
    - **Input / Output Connections:** Input: `Carregar conhecimentos e taxonomias dos projetos`; Output: `Separar assuntos em itens`.
    - **Version-Specific Requirements:** Version 2.
    - **Edge Cases / Failure Types:** Empty filter results terminate data flow.
  - **Separar assuntos em itens**
    - **Type and Technical Role:** `n8n-nodes-base.splitOut` (Data transformation node)
    - **Configuration Choices:** Splits the aggregated `conhecimentos` array into individual items.
    - **Key Expressions / Variables:** Field to split out: `conhecimentos`.
    - **Input / Output Connections:** Input: `Filtrar conhecimentos do tipo Assunto`; Output: `Percorrer assuntos candidatos`.
    - **Version-Specific Requirements:** Version 1.
    - **Edge Cases / Failure Types:** Fails if the target field is not an array.
  - **Percorrer assuntos candidatos**
    - **Type and Technical Role:** `n8n-nodes-base.splitInBatches` (Loop control node)
    - **Configuration Choices:** Processes candidate topics iteratively in batches.
    - **Key Expressions / Variables:** Standard loop control configuration.
    - **Input / Output Connections:** Input: `Separar assuntos em itens` (and feedback loop from `Encontrou assunto e anotação?`); Output: Loops to `Carregar subitens e anotações do assunto`.
    - **Version-Specific Requirements:** Version 3.
    - **Edge Cases / Failure Types:** Infinite loops if exit conditions are misconfigured.
  - **Carregar subitens e anotações do assunto**
    - **Type and Technical Role:** `n8n-nodes-base.executeWorkflow` (Sub-workflow execution node)
    - **Configuration Choices:** Recursively expands the `"Subitem"` hierarchy of the knowledge record, retrieving nested `"📝 Anotações"` along with their `"🏷️ Taxonomias"` and `"📦 Resultados"`.
    - **Key Expressions / Variables:** `={{ $json }}` passing current candidate object; recursive property definitions set to true.
    - **Input / Output Connections:** Input: `Percorrer assuntos candidatos`; Output: `Selecionar primeiro assunto com anotação disponível para LinkedIn`.
    - **Version-Specific Requirements:** Version 1.3.
    - **Edge Cases / Failure Types:** Deep recursive trees may hit execution time limits or large payload sizes.
  - **Selecionar primeiro assunto com anotação disponível para LinkedIn**
    - **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript execution node)
    - **Configuration Choices:** Evaluates the knowledge tree recursively to find the first valid annotation that features both `"Post"` and `"LinkedIn"` taxonomies and has an empty `"📦 Resultados"` relation.
    - **Key Expressions / Variables:** Custom recursive search functions (`findFirstValidKnowledge`, `annotationIsValid`, `getTitle`).
    - **Input / Output Connections:** Input: `Carregar subitens e anotações do assunto`; Output: `Encontrou assunto e anotação?`.
    - **Version-Specific Requirements:** Version 2.
    - **Edge Cases / Failure Types:** Missing database properties or unexpected schema structures will cause property accessor errors (`TypeError`).

---

#### Block 1.4: Verification and Notion Sync
- **Overview:** Checks whether a valid topic and annotation were successfully resolved, then updates the original Notion task and its parent tasks to establish the content relations.
- **Nodes Involved:**
  - `Encontrou assunto e anotação?`
  - `Vincular assunto e anotação à tarefa`
  - `Separar tarefas-pai`
  - `Vincular assunto e anotação à tarefa-pai`

- **Node Details:**
  - **Encontrou assunto e anotação?**
    - **Type and Technical Role:** `n8n-nodes-base.if` (Conditional branching node)
    - **Configuration Choices:** Evaluates whether both `📚 Conhecimentos` and `📝 Anotação` result arrays contain elements (are not empty).
    - **Key Expressions / Variables:** `={{ $json['📚 Conhecimentos'] }}` and `={{ $json['📝 Anotação'] }}` set to `notEmpty`.
    - **Input / Output Connections:** Input: `Selecionar primeiro assunto com anotação disponível para LinkedIn`; Output True: `Vincular assunto e anotação à tarefa`; Output False: Loops back to `Percorrer assuntos candidatos`.
    - **Version-Specific Requirements:** Version 2.3.
    - **Edge Cases / Failure Types:** Incorrect data typing in condition checks.
  - **Vincular assunto e anotação à tarefa**
    - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (API integration node)
    - **Configuration Choices:** Sends an HTTP `PATCH` request to the Notion REST API to update page properties with the selected knowledge and annotation IDs. Authenticated via generic HTTP Bearer Auth credential.
    - **Key Expressions / Variables:** Dynamic URL pointing to `https://api.notion.com/v1/pages/{{ $('Receber tarefa').item.json.body.tarefa.id }}`; Header `Notion-Version` set to `={{ $vars.Notion_Version }}`; JSON body stringifying relation IDs.
    - **Input / Output Connections:** Input: `Encontrou assunto e anotação?` (True branch); Output: `Separar tarefas-pai`.
    - **Version-Specific Requirements:** Version 4.4. Credential: Notion (`KncxYTjzSiUzpiOa`).
    - **Edge Cases / Failure Types:** Notion API rate limits (HTTP 429), expired integration tokens, or schema validation failures (HTTP 400).
  - **Separar tarefas-pai**
    - **Type and Technical Role:** `n8n-nodes-base.splitOut` (Data transformation node)
    - **Configuration Choices:** Unpacks the `"Item pai"` relation array returned from the Notion task update response into separate items.
    - **Key Expressions / Variables:** `properties['Item pai'].relation`.
    - **Input / Output Connections:** Input: `Vincular assunto e anotação à tarefa`; Output: `Vincular assunto e anotação à tarefa-pai`.
    - **Version-Specific Requirements:** Version 1.
    - **Edge Cases / Failure Types:** If the task has no parent items, the array is empty and execution stops here.
  - **Vincular assunto e anotação à tarefa-pai**
    - **Type and Technical Role:** `n8n-nodes-base.httpRequest` (API integration node)
    - **Configuration Choices:** Sends an HTTP `PATCH` request to update each related parent page ID with the exact same knowledge and annotation relations.
    - **Key Expressions / Variables:** Dynamic URL `https://api.notion.com/v1/pages/{{ $json.id }}`; header `Notion-Version` set to `={{ $vars.Notion_Version }}`; payload mapped from the selection code node.
    - **Input / Output Connections:** Input: `Separar tarefas-pai`; Output: None (Terminal node).
    - **Version-Specific Requirements:** Version 4.4. Credential: Notion (`KncxYTjzSiUzpiOa`).
    - **Edge Cases / Failure Types:** Insufficient Notion workspace permissions on parent pages resulting in HTTP 403 forbidden responses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Receber tarefa` | `webhook` | Receives webhook POST request with Notion task page object. | None | `Carregar projetos e taxonomias da tarefa` | ### Como funciona<br><br>Este workflow seleciona um assunto disponível para uma publicação no LinkedIn a partir de uma tarefa do Notion recebida por webhook.<br><br>Primeiro, carrega os projetos relacionados à tarefa, mantém aqueles classificados como Post e reúne seus conhecimentos classificados como Assunto. Depois, percorre os candidatos, carregando seus subitens e anotações.<br><br>A seleção busca um conhecimento do tipo Assunto com uma anotação que tenha as taxonomias Post e LinkedIn e nenhum resultado vinculado. Ao encontrar a primeira combinação válida, o fluxo vincula o assunto e a anotação à tarefa e às suas tarefas-pai, encerrando a busca sem percorrer os candidatos restantes.<br><br>Se todos os candidatos forem esgotados sem correspondência, o fluxo termina sem atualizar as tarefas.<br><br>### Configuração<br><br>- Configure a credencial de acesso ao Notion nos módulos de atualização e permita que a integração acesse as páginas utilizadas.<br>- Disponibilize o subworkflow “Enriquecer Página do Notion” e selecione-o nos três módulos de carregamento.<br>- Defina a variável `Notion_Version`, utilizada no cabeçalho das requisições ao Notion.<br>- Confira os nomes das propriedades, relações e taxonomias utilizados nos filtros e atualizações, adaptando-os à sua estrutura.<br>- Envie uma requisição POST ao webhook com o objeto da página da tarefa no campo `tarefa` do corpo JSON. |
| `Carregar projetos e taxonomias da tarefa` | `executeWorkflow` | Expands project and taxonomy relations via sub-workflow. | `Receber tarefa` | `Filtrar projetos do tipo Post` | ## Preparar assuntos candidatos<br><br>Carrega os projetos relacionados à tarefa recebida e suas taxonomias. Mantém os projetos do tipo Post, enriquece seus conhecimentos e reúne aqueles classificados como Assunto para a busca de uma anotação disponível. |
| `Filtrar projetos do tipo Post` | `code` | Filters projects to keep only those tagged as "Post". | `Carregar projetos e taxonomias da tarefa` | `Carregar conhecimentos e taxonomias dos projetos` | ## Preparar assuntos candidatos<br><br>Carrega os projetos relacionados à tarefa recebida e suas taxonomias. Mantém os projetos do tipo Post, enriquece seus conhecimentos e reúne aqueles classificados como Assunto para a busca de uma anotação disponível. |
| `Carregar conhecimentos e taxonomias dos projetos` | `executeWorkflow` | Loads knowledge and taxonomy relations for post projects. | `Filtrar projetos do tipo Post` | `Filtrar conhecimentos do tipo Assunto` | ## Preparar assuntos candidatos<br><br>Carrega os projetos relacionados à tarefa recebida e suas taxonomias. Mantém os projetos do tipo Post, enriquece seus conhecimentos e reúne aqueles classificados como Assunto para a busca de uma anotação disponível. |
| `Filtrar conhecimentos do tipo Assunto` | `code` | Filters knowledge records to retain those tagged as "Assunto". | `Carregar conhecimentos e taxonomias dos projetos` | `Separar assuntos em itens` | ## Preparar assuntos candidatos<br><br>Carrega os projetos relacionados à tarefa recebida e suas taxonomias. Mantém os projetos do tipo Post, enriquece seus conhecimentos e reúne aqueles classificados como Assunto para a busca de uma anotação disponível. |
| `Separar assuntos em itens` | `splitOut` | Splits knowledge array items into individual rows. | `Filtrar conhecimentos do tipo Assunto` | `Percorrer assuntos candidatos` | ## Preparar assuntos candidatos<br><br>Carrega os projetos relacionados à tarefa recebida e suas taxonomias. Mantém os projetos do tipo Post, enriquece seus conhecimentos e reúne aqueles classificados como Assunto para a busca de uma anotação disponível. |
| `Percorrer assuntos candidatos` | `splitInBatches` | Loops through candidate topics iteratively. | `Separar assuntos em itens`, `Encontrou assunto e anotação?` | `Carregar subitens e anotações do assunto` (loop out) | ## Buscar uma anotação disponível<br><br>Percorre os assuntos candidatos individualmente e carrega seus subitens e anotações. Em cada hierarquia, busca o primeiro conhecimento do tipo Assunto com uma anotação classificada como Post e LinkedIn, sem resultados vinculados. |
| `Carregar subitens e anotações do assunto` | `executeWorkflow` | Recursively expands topic subitems, annotations, and taxonomies. | `Percorrer assuntos candidatos` | `Selecionar primeiro assunto com anotação disponível para LinkedIn` | ## Buscar uma anotação disponível<br><br>Percorre os assuntos candidatos individualmente e carrega seus subitens e anotações. Em cada hierarquia, busca o primeiro conhecimento do tipo Assunto com uma anotação classificada como Post e LinkedIn, sem resultados vinculados. |
| `Selecionar primeiro assunto com anotação disponível para LinkedIn` | `code` | Evaluates tree to pick the first valid unpublished LinkedIn annotation. | `Carregar subitens e anotações do assunto` | `Encontrou assunto e anotação?` | ## Buscar uma anotação disponível<br><br>Percorre os assuntos candidatos individualmente e carrega seus subitens e anotações. Em cada hierarquia, busca o primeiro conhecimento do tipo Assunto com uma anotação classificada como Post e LinkedIn, sem resultados vinculados. |
| `Encontrou assunto e anotação?` | `if` | Checks if valid knowledge and annotation were identified. | `Selecionar primeiro assunto com anotação disponível para LinkedIn` | `Vincular assunto e anotação à tarefa` (True), `Percorrer assuntos candidatos` (False) | ## Vincular a primeira correspondência<br><br>Ao encontrar assunto e anotação válidos, segue pela saída verdadeira, vincula ambos à tarefa e às tarefas-pai e encerra a busca sem retornar ao loop. Caso contrário, a saída falsa retorna ao loop para avaliar o próximo candidato. |
| `Vincular assunto e anotação à tarefa` | `httpRequest` | Updates original Notion task page with selected relations via API. | `Encontrou assunto e anotação?` | `Separar tarefas-pai` | ## Vincular a primeira correspondência<br><br>Ao encontrar assunto e anotação válidos, segue pela saída verdadeira, vincula ambos à tarefa e às tarefas-pai e encerra a busca sem retornar ao loop. Caso contrário, a saída falsa retorna ao loop para avaliar o próximo candidato. |
| `Separar tarefas-pai` | `splitOut` | Unpacks parent task relations into individual items. | `Vincular assunto e anotação à tarefa` | `Vincular assunto e anotação à tarefa-pai` | ## Vincular a primeira correspondência<br><br>Ao encontrar assunto e anotação válidos, segue pela saída verdadeira, vincula ambos à tarefa e às tarefas-pai e encerra a busca sem retornar ao loop. Caso contrário, a saída falsa retorna ao loop para avaliar o próximo candidato. |
| `Vincular assunto e anotação à tarefa-pai` | `httpRequest` | Replicates content relations on parent Notion pages via API. | `Separar tarefas-pai` | None | ## Vincular a primeira correspondência<br><br>Ao encontrar assunto e anotação válidos, segue pela saída verdadeira, vincula ambos à tarefa e às tarefas-pai e encerra a busca sem retornar ao loop. Caso contrário, a saída falsa retorna ao loop para avaliar o próximo candidato. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger (`Receber tarefa`):**
   - Add a **Webhook** node. Set HTTP Method to `POST`, Response Mode to `Last Node`, and path as required.
2. **Add Project Expansion Sub-workflow (`Carregar projetos e taxonomias da tarefa`):**
   - Add an **Execute Workflow** node. Select the sub-workflow ID `hBvVpMuQo3608cVw` ("Enriquecer Página do Notion").
   - Map input parameters: Set `Page` to `={{ $json.body.tarefa }}` and define `Properties` targeting `🎯 Projetos` with nested property `🏷️ Taxonomia`.
3. **Filter Post Projects (`Filtrar projetos do tipo Post`):**
   - Add a **Code** node. Insert JavaScript logic to iterate over `item.json["🎯 Projetos"]` and verify if any nested `🏷️ Taxonomia` has a plain text title matching `"Post"`.
4. **Load Project Knowledge (`Carregar conhecimentos e taxonomias dos projetos`):**
   - Add an **Execute Workflow** node referencing sub-workflow `hBvVpMuQo3608cVw`.
   - Set input `Page` to `={{ $json }}` and configure `Properties` to expand `📚 Conhecimentos` and its nested `🏷️ Taxonomias`.
5. **Filter Knowledge Topics (`Filtrar conhecimentos do tipo Assunto`):**
   - Add a **Code** node to inspect expanded knowledge records and retain items where taxonomies include `"Assunto"`.
6. **Split Topics (`Separar assuntos em itens`):**
   - Add a **Split Out** node targeting the `conhecimentos` array field.
7. **Iterate Candidates (`Percorrer assuntos candidatos`):**
   - Add a **Split In Batches** node to process candidate topics iteratively.
8. **Load Subitems and Annotations (`Carregar subitens e anotações do assunto`):**
   - Add an **Execute Workflow** node calling `hBvVpMuQo3608cVw`.
   - Map input `Page` to `={{ $json }}` and configure recursive properties to expand `Subitem` with nested `📝 Anotações` (`🏷️ Taxonomias`, `📦 Resultados`) and `🏷️ Taxonomias`.
9. **Select Valid Annotation (`Selecionar primeiro assunto com anotação disponível para LinkedIn`):**
   - Add a **Code** node implementing recursive traversal logic to locate the first topic node containing an annotation with `"Post"`, `"LinkedIn"`, and an empty `📦 Resultados` array.
10. **Add Conditional Branch (`Encontrou assunto e anotação?`):**
    - Add an **If** node checking that both `={{ $json['📚 Conhecimentos'] }}` and `={{ $json['📝 Anotação'] }}` arrays are not empty.
11. **Update Original Task (`Vincular assunto e anotação à tarefa`):**
    - Add an HTTP Request node (Method: `PATCH`, Authentication: Generic Credential Type -> HTTP Bearer Auth using Notion credential).
    - URL: `https://api.notion.com/v1/pages/{{ $('Receber tarefa').item.json.body.tarefa.id }}`.
    - Set header parameter `Notion-Version` to `={{ $vars.Notion_Version }}`.
    - Set JSON body containing the relation objects for `📚 Conhecimentos` and `📝 Anotação`.
12. **Extract Parent Tasks (`Separar tarefas-pai`):**
    - Add a **Split Out** node with field `properties['Item pai'].relation`.
13. **Update Parent Tasks (`Vincular assunto e anotação à tarefa-pai`):**
    - Add an HTTP Request node (Method: `PATCH`, Authentication: HTTP Bearer Auth).
    - URL: `https://api.notion.com/v1/pages/{{ $json.id }}`.
    - Set header parameter `Notion-Version` to `={{ $vars.Notion_Version }}` and mirror the relation payload from the selection step.
14. **Establish Connections:**
    - Connect the nodes sequentially as documented in the overview and summary table, routing the `False` branch of `Encontrou assunto e anotação?` back to `Percorrer assuntos candidatos` to continue iteration.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Notion API Integration Setup | Ensure the connected Notion integration token has explicit read/write access to all relevant databases (`Tarefas`, `Projetos`, `Conhecimentos`, `Anotação`, `Taxonomias`, `Resultados`). |
| Sub-workflow Requirement | The workflow depends on a reusable sub-workflow named **"Enriquecer Página do Notion"** (`hBvVpMuQo3608cVw`) to handle relational page expansion dynamically. |
| Global Variables | Ensure the workflow variable `Notion_Version` is defined in your n8n environment to populate the required Notion API version header correctly. |