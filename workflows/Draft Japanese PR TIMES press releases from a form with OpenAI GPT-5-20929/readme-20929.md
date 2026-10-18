Draft Japanese PR TIMES press releases from a form with OpenAI GPT-5

https://n8nworkflows.xyz/workflows/draft-japanese-pr-times-press-releases-from-a-form-with-openai-gpt-5-20929


# Draft Japanese PR TIMES press releases from a form with OpenAI GPT-5

### 1. Workflow Overview

This workflow automates the generation of Japanese PR TIMES-compliant press release drafts from user-submitted form data. It targets marketing and PR teams who need structured, compliant, and fact-driven drafts ready for review before publishing. The workflow is organized into four core logical blocks:

- **1.1 Input Reception & Company Context:** Collects press release parameters via an interactive n8n form and injects static organization metadata (company name, headquarters, representative, overview, and style rules).
- **1.2 AI Processing & Structured Parsing:** Passes the combined context to an OpenAI LLM chain using a strict prompt framework, enforcing editorial rules and returning a structured JSON output (title, subtitle, lead, body, keywords, and check points).
- **1.3 Validation & Post-Processing:** Evaluates the AI output for subtitle character limits, flags risky announcement types (such as document releases), deduplicates keywords, and aggregates corporate boilerplate text.
- **1.4 Output Display:** Renders the final, formatted press release draft inside an n8n completion form using custom HTML styling, allowing human editors to inspect warnings and check points directly in the browser.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Company Context
**Overview:**  
This block initiates the execution cycle by capturing the raw user-submitted announcement details and enriching them with organization-level metadata required for Japanese corporate communications.

**Nodes Involved:**  
- `When Form Submitted`
- `Set Company Information`

**Node Details:**

- **When Form Submitted**
  - **Type and Technical Role:** `n8n-nodes-base.formTrigger` (Trigger Node)
  - **Configuration Choices:** Configured with custom bilingual labels (Japanese/English) and submission button text ("下書きを作る / Create draft"). Collects 14 distinct fields including `product_name`, `product_definition`, `announcement_type` (dropdown), `release_date`, `facts`, `background`, `features`, `audience`, `url`, `ceo_comment`, `next_steps`, `cta`, `extra`, and `must_keywords`.
  - **Key Expressions or Variables:** None (trigger node).
  - **Input and Output Connections:** Input: None (Trigger); Output: Connects to `Set Company Information`.
  - **Version-Specific Requirements:** Version 2.6.
  - **Edge Cases or Potential Failure Types:** Users submitting empty required fields; invalid date formats in `release_date`.

- **Set Company Information**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node)
  - **Configuration Choices:** Assigns static organizational strings: `company_name` ("サンプル株式会社"), `company_hq` ("本社：東京都千代田区"), `ceo` ("代表取締役：山田 太郎"), `company_overview`, `contact`, and `style_rules`. Retains all incoming form fields via "Include Other Fields".
  - **Key Expressions or Variables:** Static string assignments.
  - **Input and Output Connections:** Input: `When Form Submitted`; Output: Connects to `Draft Press Release`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases or Potential Failure Types:** Missing configuration constants if the user fails to update sample values to real corporate data.

---

#### Block 1.2: AI Processing & Structured Parsing
**Overview:**  
This block leverages OpenAI models within an advanced LangChain framework to synthesize the enriched company context and form data into a strictly formatted Japanese PR TIMES press release.

**Nodes Involved:**  
- `Draft Press Release`
- `OpenAI GPT-5 Mini`
- `Parse Draft Sections`

**Node Details:**

- **Draft Press Release**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent / Chain Node)
  - **Configuration Choices:** Configured with a system prompt detailing corporate PR writing rules, structural constraints (title, 60–100 character subtitle, lead paragraph, specific heading structures using `■`), prohibition of unverified claims, and constraints on keyword counts and editorial check points. Set to retry up to 3 times on failure.
  - **Key Expressions or Variables:** 
    - `{{ $json.company_name }}`
    - `{{ $json.company_hq }}`
    - `{{ $json.ceo }}`
    - `{{ $json.style_rules }}`
    - `{{ $json.product_name }}`
    - `{{ $json.product_definition || '(not provided)' }}`
    - `{{ $json.announcement_type }}`
    - `{{ DateTime.fromISO($json.release_date).toFormat('yyyy年M月d日') }}`
    - `{{ $json.facts }}`
    - `{{ $json.background || '(not provided)' }}`
    - `{{ $json.features || '(not provided)' }}`
    - `{{ $json.audience || '(not provided)' }}`
    - `{{ $json.url || '(not provided)' }}`
    - `{{ $json.ceo_comment || '(not provided)' }}`
    - `{{ $json.next_steps || '(not provided)' }}`
    - `{{ $json.cta || '(not provided)' }}`
    - `{{ $json.extra || '(not provided)' }}`
    - `{{ $json.must_keywords || '(not provided)' }}`
  - **Input and Output Connections:** Input: Main data stream from `Set Company Information`, language model via `OpenAI GPT-5 Mini`, output parser via `Parse Draft Sections`; Output: Connects to `Set Title and Warnings`.
  - **Version-Specific Requirements:** Version 1.9.
  - **Edge Cases or Potential Failure Types:** API timeouts, rate limits, or hallucinations injecting unverified claims (mitigated by prompt guardrails).

- **OpenAI GPT-5 Mini**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Sub-Node)
  - **Configuration Choices:** Model ID configured to `gpt-5-mini`.
  - **Key Expressions or Variables:** None.
  - **Input and Output Connections:** Input: None (Model provider); Output: Connects to `Draft Press Release` via `ai_languageModel`.
  - **Version-Specific Requirements:** Version 1.3. Requires valid OpenAI credentials.
  - **Edge Cases or Potential Failure Types:** Invalid API keys, model deprecation, or insufficient account quota.

- **Parse Draft Sections**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser Sub-Node)
  - **Configuration Choices:** Manual schema configuration enforcing a JSON object with properties: `title` (string), `subtitle` (string), `lead` (string), `body` (string), `keywords` (array of strings), and `check_points` (array of strings). All fields are marked as required.
  - **Key Expressions or Variables:** JSON Schema definition.
  - **Input and Output Connections:** Input: None; Output: Connects to `Draft Press Release` via `ai_outputParser`.
  - **Version-Specific Requirements:** Version 1.3.
  - **Edge Cases or Potential Failure Types:** Model returning malformed JSON that fails schema validation.

---

#### Block 1.3: Validation & Post-Processing
**Overview:**  
This block processes the structured AI output, calculates subtitle character counts, triggers warnings for platform compliance issues, normalizes and deduplicates keyword lists, and appends corporate footer information.

**Nodes Involved:**  
- `Set Title and Warnings`

**Node Details:**

- **Set Title and Warnings**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node)
  - **Configuration Choices:** Maps and computes derived properties from the parsed AI output.
  - **Key Expressions or Variables:**
    - Title: `{{ $json.output.title }}`
    - Subtitle: `{{ $json.output.subtitle }}`
    - Subtitle Length: `{{ $json.output.subtitle.length }}`
    - Subtitle Warning: `{{ $json.output.subtitle.length > 100 ? '⚠️ サブタイトルが ' + $json.output.subtitle.length + ' 文字です（上限100文字）。/ The subtitle is ' + $json.output.subtitle.length + ' characters (PR TIMES limit: 100).' : '' }}`
    - Type Warning: `{{ $('When Form Submitted').item.json.announcement_type.startsWith('資料公開') ? '⚠️ PR TIMES では、ノウハウ資料やチェックリストを「公開した」こと自体が主題のリリースは掲載されないことがあります。新しい取り組みや成果が主題になっているか確かめてください。/ PR TIMES may not publish a release whose main news is only that a guide or checklist was published. Make sure the main topic is a new initiative or result.' : '' }}`
    - Lead: `{{ $json.output.lead }}`
    - Body: `{{ $json.output.body }}`
    - Keywords (Deduplicated & Sliced to 10): `{{ (() => { const seen = new Set(); return $json.output.keywords.map(k => String(k).trim()).filter(k => { const key = k.replace(/[（(].*?[）)]/g, '').replace(/\s/g, '').toLowerCase(); if (!key || seen.has(key)) return false; seen.add(key); return true; }).slice(0, 10).join('、'); })() }}`
    - Keyword Count: `{{ (() => { const seen = new Set(); return $json.output.keywords.map(k => String(k).trim()).filter(k => { const key = k.replace(/[（(].*?[）)]/g, '').replace(/\s/g, '').toLowerCase(); if (!key || seen.has(key)) return false; seen.add(key); return true; }).slice(0, 10).length; })() }}`
    - Check Points: `{{ $json.output.check_points }}`
    - Company Overview: `{{ $('Set Company Information').item.json.company_overview }}`
    - Contact: `{{ $('Set Company Information').item.json.contact }}`
  - **Input and Output Connections:** Input: `Draft Press Release`; Output: Connects to `Display Draft Form`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases or Potential Failure Types:** JavaScript execution errors if array methods receive null or unexpected types.

---

#### Block 1.4: Output Display
**Overview:**  
This block presents the final, validated press release draft and compliance warnings inside an HTML-formatted browser view via an n8n form completion response.

**Nodes Involved:**  
- `Display Draft Form`

**Node Details:**

- **Display Draft Form**
  - **Type and Technical Role:** `n8n-nodes-base.form` (Form Completion Node)
  - **Configuration Choices:** Operation set to completion (`respondWith: showText`). Renders an inline styled HTML container displaying type/subtitle warnings, title, subtitle with character count, lead paragraph, body text, company overview, press contact, keyword list, and check points.
  - **Key Expressions or Variables:**
    - `{{ $json.type_warning }}`
    - `{{ $json.subtitle_warning }}`
    - `{{ $json.title }}`
    - `{{ $json.subtitle }}`
    - `{{ $json.subtitle_length }}`
    - `{{ $json.lead.replace(/\n/g, '<br>') }}`
    - `{{ $json.body.replace(/\n/g, '<br>') }}`
    - `{{ $json.company_overview.replace(/\n/g, '<br>') }}`
    - `{{ $json.contact.replace(/\n/g, '<br>') }}`
    - `{{ $json.keywords }}`
    - `{{ $json.keyword_count }}`
    - `{{ $json.check_points }}`
  - **Input and Output Connections:** Input: `Set Title and Warnings`; Output: None (Terminal node).
  - **Version-Specific Requirements:** Version 2.5.
  - **Edge Cases or Potential Failure Types:** HTML rendering glitches if string replacements encounter unexpected special characters.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration overview. | None | None | ## Draft Japanese PR TIMES press releases from a form with OpenAI<br><br>### How it works<br><br>This workflow collects press release inputs through a form, enriches them with preset company information, and sends the combined context to an OpenAI-powered LLM chain. The AI drafts a Japanese PR TIMES-style press release and returns structured sections. It is told to use only the facts entered in the form, and anything it inferred is listed under "Points to check". The workflow then checks the subtitle length (PR TIMES limit: 100 characters), removes duplicate keywords, warns about announcement types that PR TIMES often rejects, and displays the final draft on the form completion page for review.<br><br>PR TIMES is a Japanese press release service, so the draft, the form examples, and the sample company details are in Japanese.<br><br>### Setup steps<br><br>- Update the company information in the **Set Company Information** node: company_name, company_hq, ceo, company_overview, and contact. These values are inserted into the release as-is.<br>- Connect valid OpenAI credentials to the **OpenAI GPT-5 Mini** node and confirm the selected model is available for your account.<br>- Click **Execute workflow**, open the form, and submit an announcement.<br><br>### Customization<br><br>You can adjust the form fields, the writing rules in **Draft Press Release**, the subtitle length threshold, the required keywords, and the final display fields to match different brands or release formats. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documents company context collection block. | None | None | ## Collect company context<br><br>Starts the workflow from a press release request form and adds fixed company profile details that will be used in the draft. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documents structured draft generation block. | None | None | ## Generate structured draft<br><br>Uses the LLM chain with the OpenAI chat model and structured output parser to create a Japanese PR TIMES-style press release draft in defined sections. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documents validation and display block. | None | None | ## Validate and show draft<br><br>Checks generated title and subtitle fields, calculates subtitle length and warnings, then presents the completed draft for review. |
| `When Form Submitted` | `n8n-nodes-base.formTrigger` | Collects announcement facts and metadata via a web form. | None | `Set Company Information` | |
| `Set Company Information` | `n8n-nodes-base.set` | Injects static corporate details and writing style rules. | `When Form Submitted` | `Draft Press Release` | |
| `Draft Press Release` | `@n8n/n8n-nodes-langchain.chainLlm` | Generates a structured Japanese press release using AI. | `Set Company Information`, `OpenAI GPT-5 Mini`, `Parse Draft Sections` | `Set Title and Warnings` | |
| `OpenAI GPT-5 Mini` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides the language model backend (`gpt-5-mini`). | None | `Draft Press Release` | |
| `Parse Draft Sections` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Parses AI text output into a strict JSON schema. | None | `Draft Press Release` | |
| `Set Title and Warnings` | `n8n-nodes-base.set` | Evaluates subtitle lengths, warnings, keywords, and footers. | `Draft Press Release` | `Display Draft Form` | |
| `Display Draft Form` | `n8n-nodes-base.form` | Displays the final HTML-formatted draft and alerts. | `Set Title and Warnings` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add a **Form Trigger** (`n8n-nodes-base.formTrigger`) named `When Form Submitted`. Set the form title to `PR TIMES 向けプレスリリース下書き / Press release draft for PR TIMES` and button label to `下書きを作る / Create draft`. Add all 14 required form fields (`product_name`, `product_definition`, `announcement_type`, `release_date`, `facts`, `background`, `features`, `audience`, `url`, `ceo_comment`, `next_steps`, `cta`, `extra`, `must_keywords`) with appropriate field types (text, textarea, dropdown, date) as defined in the JSON configuration.
2. **Add Company Context Node:** Create a **Set** node (`n8n-nodes-base.set`) named `Set Company Information`. Connect `When Form Submitted` to it. Configure assignments for `company_name`, `company_hq`, `ceo`, `company_overview`, `contact`, and `style_rules` using appropriate corporate placeholder values. Ensure "Include Other Fields" is enabled.
3. **Configure AI Model Sub-Node:** Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) named `OpenAI GPT-5 Mini`. Set the model parameter to `gpt-5-mini`. Configure valid OpenAI credentials.
4. **Configure Output Parser Sub-Node:** Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) named `Parse Draft Sections`. Define the manual JSON input schema requiring `title`, `subtitle`, `lead`, `body`, `keywords` (array), and `check_points` (array).
5. **Configure LLM Chain Node:** Add a **Basic LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`) named `Draft Press Release`. Set the prompt type to "Define", input text to the prompt template referencing company and announcement variables, and paste the system prompt rules into the message configuration. Connect `Set Company Information` to its main input, `OpenAI GPT-5 Mini` to its `ai_languageModel` input, and `Parse Draft Sections` to its `ai_outputParser` input.
6. **Add Validation and Transformation Node:** Create a **Set** node (`n8n-nodes-base.set`) named `Set Title and Warnings`. Connect `Draft Press Release` to it. Configure assignments to extract `title`, `subtitle`, calculate `subtitle_length`, evaluate `subtitle_warning` (checking against 100 characters) and `type_warning` (checking for document release announcements), clean and deduplicate `keywords`, compute `keyword_count`, pass `check_points`, and fetch `company_overview` and `contact` from upstream nodes.
7. **Add Form Completion Node:** Create a **Form** node (`n8n-nodes-base.form`) named `Display Draft Form` with operation set to completion (`respondWith: showText`). Connect `Set Title and Warnings` to it. Populate the response text with the HTML template rendering the title, subtitle, character count, lead, body, company overview, press contact, keywords, and check points list.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PR TIMES is a Japanese press release distribution service; generated drafts and sample configurations are structured accordingly. | Japanese Press Release Service |
| Always review AI-generated drafts against factual records before publishing. The workflow isolates unverified inferences under "Points to check". | Content Accuracy and Fact-Checking |