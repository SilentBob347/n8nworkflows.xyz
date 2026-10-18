Create social carousel image sets from Google Sheets with ImageStep

https://n8nworkflows.xyz/workflows/create-social-carousel-image-sets-from-google-sheets-with-imagestep-20994


# Create social carousel image sets from Google Sheets with ImageStep

### 1. Workflow Overview

This workflow automates the creation of social media carousel image sets based on data retrieved from Google Sheets. Its primary purpose is to generate four consistent brand-aligned carousel cards per sheet row, using an identical character across all generated assets, and subsequently write the generated public image URLs back to the originating spreadsheet row.

The workflow is structured into five sequential logical blocks:
- **1.1 Input Reception:** Listens for new entries added to a Google Sheet.
- **1.2 Prompt Construction:** Formats incoming row variables into a structured prompt containing a placeholder for character consistency.
- **1.3 AI Image Generation:** Executes an API call via ImageStep using a predefined brand preset to generate four portfolio cards simultaneously.
- **1.4 Data Transformation:** Extracts individual image public URLs from the generation response payload and maps them to distinct row parameters.
- **1.5 Data Persistence:** Updates the specific Google Sheets row using the generated image URLs, job identifier, and status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** This block monitors a targeted Google Sheet and triggers the automation every time a new data row containing a headline and subline is added.
- **Nodes Involved:** 
  - `Google Sheets Trigger`
- **Node Details:**
  - **Google Sheets Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheetsTrigger` (Trigger Node). Polling mechanism that watches a Google Sheets document for added rows.
    - *Configuration Choices:* Configured to poll every minute (`everyMinute`) for the `rowAdded` event. Operates on a targeted document ID and sheet name.
    - *Key Expressions or Variables:* None (polling based).
    - *Input and Output Connections:* Input: None (triggers workflow). Output: Connects to `Build the Prompt`.
    - *Version-specific Requirements:* Type Version 1. Requires valid Google Sheets OAuth2 trigger credentials.
    - *Edge Cases or Potential Failure types:* API rate limits, revoked OAuth credentials, or missing column headers (`headline`, `subline`).

---

#### 2.2 Prompt Construction
- **Overview:** Prepares the generation prompt by combining row-specific content with a template variable that ensures character consistency across card generations.
- **Nodes Involved:**
  - `Build the Prompt`
- **Node Details:**
  - **Build the Prompt**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node). Executes JavaScript code to construct a composite prompt string.
    - *Configuration Choices:* JavaScript execution environment configured to process incoming items and append static modifiers to dynamic row inputs.
    - *Key Expressions or Variables:* Uses JavaScript template literals combining `${item.json.headline}`, `${item.json.subline}`, and the literal text `{{subject.hero}}`.
    - *Input and Output Connections:* Input: `Google Sheets Trigger`. Output: Connects to `Generate 4 Cards`.
    - *Version-specific Requirements:* Type Version 2.
    - *Edge Cases or Potential Failure types:* Unhandled empty strings in `headline` or `subline` fields leading to malformed prompts.

---

#### 2.3 AI Image Generation
- **Overview:** Sends the constructed prompt and configuration settings to the ImageStep API to generate a batch of four consistent carousel images.
- **Nodes Involved:**
  - `Generate 4 Cards`
- **Node Details:**
  - **Generate 4 Cards**
    - *Type and Technical Role:* `n8n-nodes-imagestep.imageStep` (Action Node). Interacts with the ImageStep service to run image generation presets.
    - *Configuration Choices:* Operation set to `run` using resource `preset`. Preset name set to `carousel-brand`. Count set to `4` cards. Wait mechanism enabled (`wait: true`) with a timeout of 300 seconds. `publish` and `storeResult` set to true. `dryRun` set to false.
    - *Key Expressions or Variables:* Prompt parameter evaluated as an expression: `={{ $json.prompt }}`.
    - *Input and Output Connections:* Input: `Build the Prompt`. Output: Connects to `Four URLs Per Row`.
    - *Version-specific Requirements:* Requires the community node `n8n-nodes-imagestep` and valid ImageStep API credentials.
    - *Edge Cases or Potential Failure types:* API timeouts (mitigated by `waitSeconds`), insufficient API balance/credits, or invalid preset names.

---

#### 2.4 Data Transformation
- **Overview:** Parses the multi-item response array returned by the ImageStep execution and maps individual output URLs to dedicated row variables.
- **Nodes Involved:**
  - `Four URLs Per Row`
- **Node Details:**
  - **Four URLs Per Row**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node). Executes JavaScript to normalize nested output arrays into flat key-value pairs.
    - *Configuration Choices:* Iterates over `item.json.outputs` to extract `publicUrl` strings and pairs them with job metadata.
    - *Key Expressions or Variables:* Uses `$('[Google Sheets Trigger]').item.json.row_number` to retain the original row identifier.
    - *Input and Output Connections:* Input: `Generate 4 Cards`. Output: Connects to `Write URLs Back`.
    - *Version-specific Requirements:* Type Version 2.
    - *Edge Cases or Potential Failure types:* Failure to generate all four URLs resulting in undefined array indices (handled safely via fallback empty strings `|| ''`).

---

#### 2.5 Data Persistence
- **Overview:** Updates the originating spreadsheet row with the newly generated image URLs, job ID, and status.
- **Nodes Involved:**
  - `Write URLs Back`
- **Node Details:**
  - **Write URLs Back**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action Node). Modifies existing rows in a Google Sheets document.
    - *Configuration Choices:* Operation set to `update`. Mapping mode configured to `defineBelow` using `row_number` as the matching column (`defaultMatch: true`).
    - *Key Expressions or Variables:* Maps output properties using expressions: `={{ $json.url1 }}`, `={{ $json.url2 }}`, `={{ $json.url3 }}`, `={{ $json.url4 }}`, `={{ $json.jobId }}`, `={{ $json.status }}`, and `={{ $json.row_number }}`.
    - *Input and Output Connections:* Input: `Four URLs Per Row`. Output: None (terminal node).
    - *Version-specific Requirements:* Type Version 4.5. Requires valid Google Sheets OAuth2 API credentials.
    - *Edge Cases or Potential Failure types:* Mismatched row numbers, missing matching columns, or permission errors on the target Google Sheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Google Sheets Trigger` | `n8n-nodes-base.googleSheetsTrigger` | Triggers workflow on new sheet row | None | `Build the Prompt` | ## Same character on every carousel card<br><br>Every row you add to a Google Sheet becomes four social-carousel images with the same character on all of them, published, with the four URLs written back into the row. Built on [ImageStep](https://imagestep.dev/?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel), an image API for automations.<br><br>### How it works<br>1. **Google Sheets Trigger** picks up a new row (`headline`, `subline`).<br>2. **Build the Prompt** writes the row's scene with `{{subject.hero}}` where the character goes; ImageStep fills in the wording saved on the preset. It is a Code node because n8n reads `{{ }}` in a field as its own expression.<br>3. **Generate 4 Cards** runs the ImageStep preset `carousel-brand` with Count 4. Its subject (reference images plus a locked description) goes with every generation, so the fortieth row draws the character the first one did.<br>4. **Four URLs Per Row** and **Write URLs Back** put `url1` to `url4`, the job id and the status into the same row.<br><br>### Setup<br>1. Install the ImageStep node (`n8n-nodes-imagestep`) and add an ImageStep credential with an API key from [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel).<br>2. Save the `carousel-brand` preset once: the [recipe's script](https://github.com/imagestep/imagestep-sdk/tree/main/recipes/social-carousel?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) does it from your reference images.<br>3. Make a sheet with the columns `headline`, `subline`, `url1` to `url4`, `jobId` and `status`, and pick it in both Google Sheets nodes.<br>4. Turn on **Dry Run** in Generate 4 Cards for the first row: it prices the run and spends nothing.<br><br>### Customization tips<br>- Change the scene wording in Build the Prompt, and keep `{{subject.hero}}`.<br>- **Count** sets the cards per row; **Version** pins a preset version.<br>- More pipelines and the pictures a real run made: [imagestep.dev/docs/recipes](https://imagestep.dev/docs/recipes?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel#social-carousel).<br><br>### 1. A new row comes in<br>Every minute the trigger looks for rows added to the sheet. Each row needs `headline` and `subline`. |
| `Build the Prompt` | `n8n-nodes-base.code` | Formats prompt and character placeholder | `Google Sheets Trigger` | `Generate 4 Cards` | ## Same character on every carousel card<br><br>Every row you add to a Google Sheet becomes four social-carousel images with the same character on all of them, published, with the four URLs written back into the row. Built on [ImageStep](https://imagestep.dev/?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel), an image API for automations.<br><br>### How it works<br>1. **Google Sheets Trigger** picks up a new row (`headline`, `subline`).<br>2. **Build the Prompt** writes the row's scene with `{{subject.hero}}` where the character goes; ImageStep fills in the wording saved on the preset. It is a Code node because n8n reads `{{ }}` in a field as its own expression.<br>3. **Generate 4 Cards** runs the ImageStep preset `carousel-brand` with Count 4. Its subject (reference images plus a locked description) goes with every generation, so the fortieth row draws the character the first one did.<br>4. **Four URLs Per Row** and **Write URLs Back** put `url1` to `url4`, the job id and the status into the same row.<br><br>### Setup<br>1. Install the ImageStep node (`n8n-nodes-imagestep`) and add an ImageStep credential with an API key from [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel).<br>2. Save the `carousel-brand` preset once: the [recipe's script](https://github.com/imagestep/imagestep-sdk/tree/main/recipes/social-carousel?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) does it from your reference images.<br>3. Make a sheet with the columns `headline`, `subline`, `url1` to `url4`, `jobId` and `status`, and pick it in both Google Sheets nodes.<br>4. Turn on **Dry Run** in Generate 4 Cards for the first row: it prices the run and spends nothing.<br><br>### Customization tips<br>- Change the scene wording in Build the Prompt, and keep `{{subject.hero}}`.<br>- **Count** sets the cards per row; **Version** pins a preset version.<br>- More pipelines and the pictures a real run made: [imagestep.dev/docs/recipes](https://imagestep.dev/docs/recipes?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel#social-carousel).<br><br>### 2. The scene, not the character<br>The row describes the scene. `{{subject.hero}}` stands in for the character, and ImageStep fills in the saved wording. |
| `Generate 4 Cards` | `n8n-nodes-imagestep.imageStep` | Generates 4 carousel cards using ImageStep | `Build the Prompt` | `Four URLs Per Row` | ## Same character on every carousel card<br><br>Every row you add to a Google Sheet becomes four social-carousel images with the same character on all of them, published, with the four URLs written back into the row. Built on [ImageStep](https://imagestep.dev/?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel), an image API for automations.<br><br>### How it works<br>1. **Google Sheets Trigger** picks up a new row (`headline`, `subline`).<br>2. **Build the Prompt** writes the row's scene with `{{subject.hero}}` where the character goes; ImageStep fills in the wording saved on the preset. It is a Code node because n8n reads `{{ }}` in a field as its own expression.<br>3. **Generate 4 Cards** runs the ImageStep preset `carousel-brand` with Count 4. Its subject (reference images plus a locked description) goes with every generation, so the fortieth row draws the character the first one did.<br>4. **Four URLs Per Row** and **Write URLs Back** put `url1` to `url4`, the job id and the status into the same row.<br><br>### Setup<br>1. Install the ImageStep node (`n8n-nodes-imagestep`) and add an ImageStep credential with an API key from [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel).<br>2. Save the `carousel-brand` preset once: the [recipe's script](https://github.com/imagestep/imagestep-sdk/tree/main/recipes/social-carousel?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) does it from your reference images.<br>3. Make a sheet with the columns `headline`, `subline`, `url1` to `url4`, `jobId` and `status`, and pick it in both Google Sheets nodes.<br>4. Turn on **Dry Run** in Generate 4 Cards for the first row: it prices the run and spends nothing.<br><br>### Customization tips<br>- Change the scene wording in Build the Prompt, and keep `{{subject.hero}}`.<br>- **Count** sets the cards per row; **Version** pins a preset version.<br>- More pipelines and the pictures a real run made: [imagestep.dev/docs/recipes](https://imagestep.dev/docs/recipes?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel#social-carousel).<br><br>### 3. Four cards, one character<br>One job through the pinned preset. Its reference images and locked description are sent with every card. Turn on Dry Run to see the price first. |
| `Four URLs Per Row` | `n8n-nodes-base.code` | Extracts public URLs and metadata | `Generate 4 Cards` | `Write URLs Back` | ## Same character on every carousel card<br><br>Every row you add to a Google Sheet becomes four social-carousel images with the same character on all of them, published, with the four URLs written back into the row. Built on [ImageStep](https://imagestep.dev/?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel), an image API for automations.<br><br>### How it works<br>1. **Google Sheets Trigger** picks up a new row (`headline`, `subline`).<br>2. **Build the Prompt** writes the row's scene with `{{subject.hero}}` where the character goes; ImageStep fills in the wording saved on the preset. It is a Code node because n8n reads `{{ }}` in a field as its own expression.<br>3. **Generate 4 Cards** runs the ImageStep preset `carousel-brand` with Count 4. Its subject (reference images plus a locked description) goes with every generation, so the fortieth row draws the character the first one did.<br>4. **Four URLs Per Row** and **Write URLs Back** put `url1` to `url4`, the job id and the status into the same row.<br><br>### Setup<br>1. Install the ImageStep node (`n8n-nodes-imagestep`) and add an ImageStep credential with an API key from [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel).<br>2. Save the `carousel-brand` preset once: the [recipe's script](https://github.com/imagestep/imagestep-sdk/tree/main/recipes/social-carousel?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) does it from your reference images.<br>3. Make a sheet with the columns `headline`, `subline`, `url1` to `url4`, `jobId` and `status`, and pick it in both Google Sheets nodes.<br>4. Turn on **Dry Run** in Generate 4 Cards for the first row: it prices the run and spends nothing.<br><br>### Customization tips<br>- Change the scene wording in Build the Prompt, and keep `{{subject.hero}}`.<br>- **Count** sets the cards per row; **Version** pins a preset version.<br>- More pipelines and the pictures a real run made: [imagestep.dev/docs/recipes](https://imagestep.dev/docs/recipes?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel#social-carousel).<br><br>### 4. URLs back into the row<br>The four published URLs, the job id and the status go into the row that started it, matched on `row_number`. |
| `Write URLs Back` | `n8n-nodes-base.googleSheets` | Writes URLs, job ID, and status to the sheet | `Four URLs Per Row` | None | ## Same character on every carousel card<br><br>Every row you add to a Google Sheet becomes four social-carousel images with the same character on all of them, published, with the four URLs written back into the row. Built on [ImageStep](https://imagestep.dev/?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel), an image API for automations.<br><br>### How it works<br>1. **Google Sheets Trigger** picks up a new row (`headline`, `subline`).<br>2. **Build the Prompt** writes the row's scene with `{{subject.hero}}` where the character goes; ImageStep fills in the wording saved on the preset. It is a Code node because n8n reads `{{ }}` in a field as its own expression.<br>3. **Generate 4 Cards** runs the ImageStep preset `carousel-brand` with Count 4. Its subject (reference images plus a locked description) goes with every generation, so the fortieth row draws the character the first one did.<br>4. **Four URLs Per Row** and **Write URLs Back** put `url1` to `url4`, the job id and the status into the same row.<br><br>### Setup<br>1. Install the ImageStep node (`n8n-nodes-imagestep`) and add an ImageStep credential with an API key from [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel).<br>2. Save the `carousel-brand` preset once: the [recipe's script](https://github.com/imagestep/imagestep-sdk/tree/main/recipes/social-carousel?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) does it from your reference images.<br>3. Make a sheet with the columns `headline`, `subline`, `url1` to `url4`, `jobId` and `status`, and pick it in both Google Sheets nodes.<br>4. Turn on **Dry Run** in Generate 4 Cards for the first row: it prices the run and spends nothing.<br><br>### Customization tips<br>- Change the scene wording in Build the Prompt, and keep `{{subject.hero}}`.<br>- **Count** sets the cards per row; **Version** pins a preset version.<br>- More pipelines and the pictures a real run made: [imagestep.dev/docs/recipes](https://imagestep.dev/docs/recipes?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel#social-carousel).<br><br>### 4. URLs back into the row<br>The four published URLs, the job id and the status go into the row that started it, matched on `row_number`. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets Trigger Node:**
   - Add a **Google Sheets Trigger** node to the canvas.
   - Set **Event** to `Row Added`.
   - Configure **Poll Times** to run `Every Minute`.
   - Set **Document ID** to your target spreadsheet ID (e.g., replacement for `REPLACE_WITH_SPREADSHEET_ID`) and select your sheet name (e.g., `Sheet1`).
   - Configure and select your **Google Sheets Trigger OAuth2 API** credentials.

2. **Create the Build the Prompt Node:**
   - Add a **Code** node and name it `Build the Prompt`.
   - Set **Mode** to `Run Once for All Items`.
   - Paste the following JavaScript code into the **JavaScript** parameter:
     ```javascript
     return $input.all().map((item) => ({
       json: {
         row_number: item.json.row_number,
         prompt: `${item.json.headline}. ${item.json.subline}. Social carousel card, {{subject.hero}} as the hero, clean background with room for text, 4:5 portrait.`
       }
     }));
     ```
   - Connect the output of `Google Sheets Trigger` to this node.

3. **Create the Generate 4 Cards Node:**
   - Add an **ImageStep** node and name it `Generate 4 Cards`.
   - Configure parameters:
     - **Resource:** `Preset`
     - **Operation:** `Run`
     - **Preset:** `carousel-brand`
     - **Prompt:** `={{ $json.prompt }}`
     - **Count:** `4`
     - **Wait:** `true`
     - **Wait Seconds:** `300`
     - **Dry Run:** `false` (Toggle to `true` initially for testing)
     - **Store Result:** `true`
     - **Publish:** `true`
     - **Options -> Collection:** `carousel`
   - Configure and select your **ImageStep API** credentials (obtained from [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel)).
   - Connect the output of `Build the Prompt` to this node.

4. **Create the Four URLs Per Row Node:**
   - Add a **Code** node and name it `Four URLs Per Row`.
   - Set **Mode** to `Run Once for All Items`.
   - Paste the following JavaScript code into the **JavaScript** parameter:
     ```javascript
     return $input.all().map((item) => {
       const urls = (item.json.outputs || []).map((o) => o.publicUrl);
       return {
         json: {
           row_number: $('Google Sheets Trigger').item.json.row_number,
           jobId: item.json.jobId,
           status: item.json.status,
           url1: urls[0] || '',
           url2: urls[1] || '',
           url3: urls[2] || '',
           url4: urls[3] || ''
         }
       };
     });
     ```
   - Connect the output of `Generate 4 Cards` to this node.

5. **Create the Write URLs Back Node:**
   - Add a **Google Sheets** node and name it `Write URLs Back`.
   - Configure parameters:
     - **Operation:** `Update`
     - **Document ID:** Select the same target spreadsheet ID.
     - **Sheet Name:** Select the same sheet name (e.g., `Sheet1`).
     - **Mapping Mode:** `Define Below`
     - **Matching Columns:** Select `row_number`.
     - **Columns Mapping:** Map the fields as follows:
       - `row_number` ➔ `={{ $json.row_number }}` (Default Match: Enabled)
       - `url1` ➔ `={{ $json.url1 }}`
       - `url2` ➔ `={{ $json.url2 }}`
       - `url3` ➔ `={{ $json.url3 }}`
       - `url4` ➔ `={{ $json.url4 }}`
       - `jobId` ➔ `={{ $json.jobId }}`
       - `status` ➔ `={{ $json.status }}`
   - Configure and select your **Google Sheets OAuth2 API** credentials.
   - Connect the output of `Four URLs Per Row` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| ImageStep API Keys | [imagestep.dev/keys](https://imagestep.dev/keys?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) |
| Preset Recipe Script | [GitHub Recipe Script](https://github.com/imagestep/imagestep-sdk/tree/main/recipes/social-carousel?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel) |
| Additional Pipelines and Recipes | [imagestep.dev/docs/recipes](https://imagestep.dev/docs/recipes?utm_source=n8n-templates&utm_medium=template&utm_campaign=social-carousel#social-carousel) |