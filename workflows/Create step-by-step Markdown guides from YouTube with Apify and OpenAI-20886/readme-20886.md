Create step-by-step Markdown guides from YouTube with Apify and OpenAI

https://n8nworkflows.xyz/workflows/create-step-by-step-markdown-guides-from-youtube-with-apify-and-openai-20886


# Create step-by-step Markdown guides from YouTube with Apify and OpenAI

### 1. Workflow Overview

This workflow is designed to automatically convert public YouTube tutorial videos into structured, downloadable Markdown guides complete with timestamps, screenshots, and concise instructions. Its primary target users are technical writers, course creators, and support teams looking to transform video content into text documentation quickly.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Captures the user's target YouTube link and parameters via an interactive n8n form, then establishes global variables for runtime limits and AI configurations.
- **1.2 Video Processing:** Calls the Apify Frameskim actor synchronously to extract segmented frames, timestamps, and transcripts based on user parameters.
- **1.3 AI Processing & Structuring:** Cleans the extracted data frames and routes them through an OpenAI Large Language Model (LLM) to generate a short imperative title and a single-sentence instruction for each video segment.
- **1.4 Document Assembly & Delivery:** Compiles the structured steps, screenshots, and timestamped YouTube links into a single Markdown file, which is then served back to the user via the form completion page.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block initiates the process by presenting a user-facing form to collect the target URL and optional parameters, followed by a set node that standardizes default configuration variables.

- **Submit YouTube tutorial**
  - **Type and technical role:** Form Trigger node acting as the workflow entry point.
  - **Configuration choices:** Configured with custom form fields (`YouTube URL`, `Start`, `End`, `Seconds between steps`) and set to respond using the last node in the execution chain.
  - **Key expressions or variables used:** `{{ $json['YouTube URL'] }}`, `{{ $json['Start (optional)'] }}`, `{{ $json['End (optional)'] }}`.
  - **Input and output connections:** Inputs: None (Trigger). Outputs: Connects to *Configure guide settings*.
  - **Version-specific requirements:** n8n-nodes-base.formTrigger version 2.2.
  - **Edge cases or potential failure types:** Invalid URLs or missing public access/captions on the YouTube video will cause downstream extraction failures.

- **Configure guide settings**
  - **Type and technical role:** Set node used to define workflow environment parameters and fallback constants.
  - **Configuration choices:** Manually assigns values for `intervalSeconds` (30), `maxFrames` (30), `language` (`en`), `maxChargeUsd` (1), and `openaiModel` (`gpt-4o-mini`). Includes other incoming fields.
  - **Key expressions or variables used:** Static numerical and string assignments.
  - **Input and output connections:** Inputs: *Submit YouTube tutorial*. Outputs: Connects to *Run Frameskim on Apify*.
  - **Version-specific requirements:** n8n-nodes-base.set version 3.4.
  - **Edge cases or potential failure types:** None.

---

#### 2.2 Video Processing
This block interfaces with the Apify platform to analyze the source YouTube video and extract actionable visual and textual frames.

- **Run Frameskim on Apify**
  - **Type and technical role:** HTTP Request node executing an external API call to an Apify actor.
  - **Configuration choices:** Sends a POST request to the Apify synchronous dataset run endpoint (`https://api.apify.com/v2/acts/emttn~frameskim/run-sync-get-dataset-items`) with a 310-second timeout. Uses generic HTTP Header Authentication.
  - **Key expressions or variables used:** 
    - Request Body: Constructs a JSON string containing video parameters (`url`, `start`, `end`, `every`, `maxFrames`, `lang`) mapped from form fields and settings.
    - Query Parameters: `maxTotalChargeUsd` set to `={{ $json.maxChargeUsd }}`.
  - **Input and output connections:** Inputs: *Configure guide settings*. Outputs: Connects to *Prepare steps from frames*.
  - **Version-specific requirements:** n8n-nodes-base.httpRequest version 4.2.
  - **Edge cases or potential failure types:** 
    - Authentication errors if the Apify API token is invalid.
    - HTTP 408 Timeout errors if video processing exceeds 300 seconds (mitigated by narrowing start/end ranges).
    - Cost cap limits halting execution if the run exceeds `maxChargeUsd`.

---

#### 2.3 AI Processing & Structuring
This block parses raw Apify data items, filters valid frames, and leverages an LLM to generate user-friendly guide text.

- **Prepare steps from frames**
  - **Type and technical role:** Code node (JavaScript) used to clean, validate, and sequence raw dataset output from Apify.
  - **Configuration choices:** Separates summary objects from frame items, filters out items missing images, and structures data into numbered step objects.
  - **Key expressions or variables used:** `$input.all()`, array mapping and filtering logic.
  - **Input and output connections:** Inputs: *Run Frameskim on Apify*. Outputs: Connects to *Write step title and instruction*.
  - **Version-specific requirements:** n8n-nodes-base.code version 2.
  - **Edge cases or potential failure types:** Throws an explicit error (`Frameskim returned no frames`) if zero valid frames are returned.

- **Write step title and instruction**
  - **Type and technical role:** Advanced AI / Basic LLM Chain node that processes individual frames into structured text instructions.
  - **Configuration choices:** Uses prompt chaining with a defined system message instructing the model to output a strict JSON object containing a short title and a single-sentence instruction based exclusively on the transcript excerpt.
  - **Key expressions or variables used:** `=Step {{ $json.step }} of a tutorial titled "{{ $json.videoTitle }}" (at {{ $json.ts }}).\n\nTranscript excerpt:\n{{ $json.text }}`
  - **Input and output connections:** Inputs: Main connection from *Prepare steps from frames*; AI model connection from *OpenAI model for step text*. Outputs: Connects to *Assemble Markdown guide*.
  - **Version-specific requirements:** `@n8n/n8n-nodes-langchain.chainLlm` version 1.7.
  - **Edge cases or potential failure types:** LLM rate limits, API key authentication failures, or non-JSON compliant responses handled via fallback string parsing in downstream code.

- **OpenAI model for step text**
  - **Type and technical role:** Language Model sub-node providing the underlying chat model configuration for the LLM chain.
  - **Configuration choices:** Configured to use the model ID evaluated from settings (`gpt-4o-mini`) with a low temperature of `0.2` for deterministic output.
  - **Key expressions or variables used:** `={{ $('Configure guide settings').first().json.openaiModel }}`
  - **Input and output connections:** Inputs: None. Outputs: Connects to *Write step title and instruction* via an `ai_languageModel` link.
  - **Version-specific requirements:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` version 1.2.
  - **Edge cases or potential failure types:** Invalid OpenAI API credentials or selecting an unavailable model ID.

---

#### 2.4 Document Assembly & Delivery
This block compiles all processed steps into a formatted Markdown document, converts it to a downloadable file, and returns it to the user.

- **Assemble Markdown guide**
  - **Type and technical role:** Code node (JavaScript) combining segment metadata and LLM text outputs into a cohesive Markdown document.
  - **Configuration choices:** Implements a parsing helper to clean LLM markdown blocks and safely extract JSON titles and instructions, appending screenshots and timestamped YouTube deep links.
  - **Key expressions or variables used:** `$('Prepare steps from frames').all()`, `$input.all()`.
  - **Input and output connections:** Inputs: *Write step title and instruction*. Outputs: Connects to *Save guide as Markdown file*.
  - **Version-specific requirements:** n8n-nodes-base.code version 2.
  - **Edge cases or potential failure types:** Malformed LLM JSON output handled via a fallback catch block assigning raw text as instructions.

- **Save guide as Markdown file**
  - **Type and technical role:** Convert to File node transforming text data into a downloadable binary file.
  - **Configuration choices:** Operates in `toText` mode, sourcing data from the `markdown` property and outputting to binary property `data` with filename `guide.md`.
  - **Key expressions or variables used:** Binary data property mapping.
  - **Input and output connections:** Inputs: *Assemble Markdown guide*. Outputs: Connects to *Return guide to the form*.
  - **Version-specific requirements:** n8n-nodes-base.convertToFile version 1.1.
  - **Edge cases or potential failure types:** Out-of-memory errors on extremely large video runs (though capped by `maxFrames`).

- **Return guide to the form**
  - **Type and technical role:** Form node operating in completion mode to finalize the user interaction.
  - **Configuration choices:** Responds with binary data (`returnBinary`), presenting a success message displaying the total step count and video title while initiating the browser download for `guide.md`.
  - **Key expressions or variables used:** `={{ $('Assemble Markdown guide').first().json.steps }}`, `={{ $('Assemble Markdown guide').first().json.title }}`.
  - **Input and output connections:** Inputs: *Save guide as Markdown file*. Outputs: None (Terminal node).
  - **Version-specific requirements:** n8n-nodes-base.form version 2.5.
  - **Edge cases or potential failure types:** Browser popup blockers or download restrictions preventing automatic file retrieval.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Submit YouTube tutorial | `n8n-nodes-base.formTrigger` | Collects YouTube URL and optional processing parameters via an n8n form. | None | Configure guide settings | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI<br><br>## Who's it for<br><br>Technical writers, support teams and course creators who would rather skim a written guide than scrub through a video tutorial.<br><br>## How it works<br><br>1. A form takes a public YouTube URL and an optional start, end and step interval.<br>2. An HTTP Request node runs the Frameskim actor on Apify, which returns a screenshot, its timestamp and the spoken words for each step.<br>3. OpenAI writes a short title and one instruction per step, using only the transcript.<br>4. A Code node builds a Markdown guide with a screenshot and a video link per step; the form's last page downloads it as `guide.md`.<br><br>## How to set up<br><br>1. Copy your Apify API token from Settings > API & Integrations.<br>2. In **Run Frameskim on Apify**, create a Header Auth credential: name `Authorization`, value `Bearer <your token>`.<br>3. Add your OpenAI credential in **OpenAI model for step text**.<br>4. Open the form and submit a tutorial URL.<br><br>## Requirements<br><br>- An Apify account. Frameskim costs about $0.12 per video plus $0.015 per frame, capped at $1 per run.<br>- An OpenAI API key.<br>- A public video with captions.<br><br>If YouTube blocks the video download, Frameskim falls back to YouTube's storyboard images: low resolution screenshots at $0.002 each.<br><br>## How to customize<br><br>Edit **Configure guide settings** to change the step interval, frame limit, caption language, cost cap or OpenAI model. Swap OpenAI for any chat model, or send the Markdown to Notion, Google Docs or email.<br>## 1. Input and settings<br>The form asks for the YouTube URL and an optional start, end and interval. **Configure guide settings** holds every value you may want to change: `intervalSeconds` (used when the form leaves the interval empty), `maxFrames`, `language` (caption language), `maxChargeUsd` (cost cap per run) and `openaiModel`. |
| Configure guide settings | `n8n-nodes-base.set` | Establishes runtime variables and default configurations for the workflow. | Submit YouTube tutorial | Run Frameskim on Apify | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 1. Input and settings... |
| Run Frameskim on Apify | `n8n-nodes-base.httpRequest` | Executes the Apify Frameskim actor synchronously to extract video frames and transcripts. | Configure guide settings | Prepare steps from frames | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 2. Run Frameskim on Apify<br>Calls the Apify actor `emttn/frameskim` in `segment` mode:<br>`POST https://api.apify.com/v2/acts/emttn~frameskim/run-sync-get-dataset-items`<br><br>Credential: **Header Auth**, name `Authorization`, value `Bearer YOUR_APIFY_TOKEN`. It is stored in n8n, never in the workflow. The `maxTotalChargeUsd` query parameter caps what a run can cost.<br><br>Apify waits up to 300 s for a synchronous run and then answers 408. Frameskim stops itself before that limit; if you ever get a 408, narrow the start and end range. |
| Prepare steps from frames | `n8n-nodes-base.code` | Filters, validates, and numbers raw frame items returned by Apify. | Run Frameskim on Apify | Write step title and instruction | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI... |
| Write step title and instruction | `@n8n/n8n-nodes-langchain.chainLlm` | Uses an LLM to generate concise step titles and instructions from transcript excerpts. | Prepare steps from frames, OpenAI model for step text | Assemble Markdown guide | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 3. Write each step<br>One LLM call per frame turns the words spoken there into a short title and one instruction, without inventing details. Replace the OpenAI model with Anthropic, Gemini, Ollama or any other chat model sub-node. |
| OpenAI model for step text | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Supplies the LLM configuration (`gpt-4o-mini`) for the instruction generation step. | None | Write step title and instruction | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 3. Write each step... |
| Assemble Markdown guide | `n8n-nodes-base.code` | Formats step data, LLM responses, and media links into a complete Markdown document. | Write step title and instruction | Save guide as Markdown file | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 4. Build the guide and return it<br>The Code node pairs each answer with its screenshot and a link to that second of the video, and **Save guide as Markdown file** turns it into `guide.md`. **Return guide to the form** is the form's last page: it shows a short message and downloads the file in the browser of whoever filled the form. The Markdown is also in the `markdown` field, ready for Notion, Google Docs or email.<br><br>Screenshot links point to your Apify storage and expire with your plan's data retention; download the images to keep the guide long term. |
| Save guide as Markdown file | `n8n-nodes-base.convertToFile` | Converts the generated Markdown string into a downloadable binary file (`guide.md`). | Assemble Markdown guide | Return guide to the form | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 4. Build the guide and return it... |
| Return guide to the form | `n8n-nodes-base.form` | Serves the form completion page, showing status metrics and triggering the file download. | Save guide as Markdown file | None | # Create step-by-step guides from YouTube tutorials with Apify and OpenAI...<br>## 4. Build the guide and return it... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Add a **Form Trigger** node named `Submit YouTube tutorial`.
   - Set form title to `YouTube tutorial to step-by-step guide`.
   - Add fields:
     - `YouTube URL` (Required, type: String/Text, placeholder: `https://youtu.be/...`)
     - `Start (optional)` (Optional, type: String/Text, placeholder: `1:00`)
     - `End (optional)` (Optional, type: String/Text, placeholder: `10:00`)
     - `Seconds between steps` (Optional, type: Number, placeholder: `30`)
   - Set Response Mode to `lastNode`.

2. **Create the Settings Node:**
   - Add a **Set** node named `Configure guide settings`.
   - Assign manual values:
     - `intervalSeconds` = `30` (Number)
     - `maxFrames` = `30` (Number)
     - `language` = `en` (String)
     - `maxChargeUsd` = `1` (Number)
     - `openaiModel` = `gpt-4o-mini` (String)
   - Connect `Submit YouTube tutorial` output to `Configure guide settings`.

3. **Create the Apify HTTP Request Node:**
   - Add an **HTTP Request** node named `Run Frameskim on Apify`.
   - Set Method to `POST`, URL to `https://api.apify.com/v2/acts/emttn~frameskim/run-sync-get-dataset-items`.
   - Configure Authentication as Generic Credential Type -> `httpHeaderAuth` (Create credential with Name: `Authorization`, Value: `Bearer <YOUR_APIFY_TOKEN>`).
   - Set Request Options timeout to `310000` ms.
   - Configure Query Parameters: Add parameter with Name `maxTotalChargeUsd` and Value `={{ $json.maxChargeUsd }}`.
   - Specify Body as JSON with expression:
     ```json
     ={{ JSON.stringify({ mode: 'segment', url: $json['YouTube URL'], start: $json['Start (optional)'] || '', end: $json['End (optional)'] || '', every: Number($json['Seconds between steps']) || $json.intervalSeconds, maxFrames: $json.maxFrames, quality: 'standard', lang: $json.language }) }}
     ```
   - Connect `Configure guide settings` output to `Run Frameskim on Apify`.

4. **Create the Data Preparation Code Node:**
   - Add a **Code** node named `Prepare steps from frames`.
   - Insert JavaScript to parse items, strip summary metadata, and map frame items into structured step objects containing `step`, `videoId`, `videoTitle`, `t`, `ts`, `imageUrl`, and `text`.
   - Connect `Run Frameskim on Apify` output to `Prepare steps from frames`.

5. **Create the OpenAI Model and LLM Chain Nodes:**
   - Add an **OpenAI Chat Model** node named `OpenAI model for step text`. Configure credential with your OpenAI API key and set Model ID to `={{ $('Configure guide settings').first().json.openaiModel }}` with Temperature `0.2`.
   - Add a **Basic LLM Chain** node named `Write step title and instruction`. Set prompt type to define, text expression to:
     ```text
     =Step {{ $json.step }} of a tutorial titled "{{ $json.videoTitle }}" (at {{ $json.ts }}).

     Transcript excerpt:
     {{ $json.text }}
     ```
   - Set the system message instruction: `You turn a transcript excerpt from a tutorial video into one step of a written guide. Use only what the excerpt says; never invent details, commands or values. Reply with a single JSON object and nothing else: {"title": "<short imperative step title, max 8 words>", "instruction": "<one sentence telling the reader what to do>"}. If the excerpt has no useful narration, use title "Continue" and describe the screenshot generically in one sentence.`
   - Connect `OpenAI model for step text` to `Write step title and instruction` via the `ai_languageModel` input.
   - Connect `Prepare steps from frames` output to the main input of `Write step title and instruction`.

6. **Create the Markdown Assembly Code Node:**
   - Add a **Code** node named `Assemble Markdown guide`.
   - Insert JavaScript to pair LLM responses with video segments, clean JSON wrappers, and construct a complete Markdown string containing headers, instructions, screenshot embeds, and timestamped YouTube links.
   - Connect `Write step title and instruction` output to `Assemble Markdown guide`.

7. **Create the File Conversion Node:**
   - Add a **Convert to File** node named `Save guide as Markdown file`.
   - Set Operation to `toText`, Source Property to `markdown`, and Binary Property Name to `data` with file name `guide.md`.
   - Connect `Assemble Markdown guide` output to `Save guide as Markdown file`.

8. **Create the Form Completion Node:**
   - Add a **Form** node named `Return guide to the form`.
   - Set Operation to `completion`, Respond With to `returnBinary`, Completion Title to `Your guide is ready`, Completion Message expression to `={{ $('Assemble Markdown guide').first().json.steps }} steps from "{{ $('Assemble Markdown guide').first().json.title }}". guide.md is downloading; if it does not start, check your browser's download settings.`, and Input Data Field Name to `data`.
   - Connect `Save guide as Markdown file` output to `Return guide to the form`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify API Token Setup | Copy token from Apify account Settings > API & Integrations to configure Header Auth (`Authorization: Bearer <token>`). |
| Cost & Fallback Disclaimer | Frameskim pricing is approximately $0.12 per video plus $0.015 per frame (capped at $1 per run). Falls back to storyboard images ($0.002 each) if video downloads fail. |
| Image Retention Warning | Screenshot links reference Apify storage and will expire according to your Apify plan retention policy; download images to preserve guides long-term. |