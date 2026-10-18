Generate short AI videos from images with the img2.video API

https://n8nworkflows.xyz/workflows/generate-short-ai-videos-from-images-with-the-img2-video-api-20892


# Generate short AI videos from images with the img2.video API

### 1. Workflow Overview

This workflow automates the process of transforming a static image URL into a short AI-generated video using the external `img2.video` API. Designed to run via manual execution, it handles asynchronous video generation by submitting an initial job request, implementing a polled waiting loop, and outputting the final downloadable MP4 file URL.

The logical flow is organized into four distinct functional blocks:
- **1.1 Input Reception & Configuration:** Initiates the workflow manually and assigns parameters such as the source image URL, AI model ID, text prompt, and API base URL.
- **1.2 Job Submission:** Transmits the generation payload via a secure HTTP POST request to start the asynchronous video creation process.
- **1.3 Asynchronous Polling Loop:** Manages a timed waiting interval (20 seconds) followed by an HTTP GET status check, branching conditionally based on whether the generation task has succeeded.
- **1.4 Output Extraction:** Formats and isolates the final video download URL once the generation job completes successfully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the execution sequence upon manual operator trigger and establishes the required contextual parameters for the video generation request.
- **Nodes Involved:** `Manual Start Trigger`, `Set Image and Model Parameters`
- **Node Details:**
  - **Manual Start Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Acts as the manual entry point for the workflow.
    - *Configuration Choices:* Standard default configuration (no parameters required).
    - *Input/Output Connections:* Output connects directly to `Set Image and Model Parameters`.
    - *Edge Cases/Failure Types:* User intervention required to start; no automated failure states.
  - **Set Image and Model Parameters**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Defines and passes workflow variables.
    - *Configuration Choices:* Assigns four string variables: `imageUrl`, `modelId`, `prompt`, and `baseUrl`.
    - *Key Expressions/Variables:* 
      - `imageUrl`: `https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/Cat03.jpg/640px-Cat03.jpg`
      - `modelId`: `seedance_mini`
      - `prompt`: `Gentle camera zoom in, subtle motion, cinematic lighting`
      - `baseUrl`: `https://api.img2.video`
    - *Input/Output Connections:* Input from `Manual Start Trigger`; output connects to `Post Video Generation Request`.
    - *Edge Cases/Failure Types:* Invalid image URLs will cause downstream API rejection.

#### 2.2 Job Submission
- **Overview:** Sends the initial configuration payload to the `img2.video` generation endpoint to kick off the asynchronous video creation job.
- **Nodes Involved:** `Post Video Generation Request`
- **Node Details:**
  - **Post Video Generation Request**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Makes a REST API call to submit data.
    - *Configuration Choices:* Uses POST method targeting `https://api.img2.video/api/video/generate`. Sends JSON body and header parameters with explicit Bearer token authorization.
    - *Key Expressions/Variables:* 
      - Body references upstream set node: `{{ $('Set Image and Model Parameters').item.json.modelId }}`, `{{ $('Set Image and Model Parameters').item.json.prompt }}`, `{{ $('Set Image and Model Parameters').item.json.imageUrl }}`
      - Header Authorization: `={{ $credentials.Authorization }}`
    - *Input/Output Connections:* Input from `Set Image and Model Parameters`; output connects to `Wait 20 Seconds`.
    - *Edge Cases/Failure Types:* HTTP 401 Unauthorized (invalid credential), HTTP 400 Bad Request (malformed parameters), or rate limiting/quota exhaustion. Requires an HTTP Header Auth credential.

#### 2.3 Asynchronous Polling Loop
- **Overview:** Implements a delay mechanism and polls the job status endpoint repeatedly until the generation task reports a successful state.
- **Nodes Involved:** `Wait 20 Seconds`, `Fetch Job Status`, `Check Job Success`
- **Node Details:**
  - **Wait 20 Seconds**
    - *Type and Technical Role:* `n8n-nodes-base.wait` — Pauses workflow execution for a fixed duration.
    - *Configuration Choices:* Configured with an interval amount of `20` seconds. Uses an n8n webhook resume mechanism.
    - *Input/Output Connections:* Inputs from `Post Video Generation Request` and the `false` branch of `Check Job Success` (creating the loop); output connects to `Fetch Job Status`.
    - *Edge Cases/Failure Types:* Excessive looping if the external service hangs indefinitely (consider adding a maximum retry count counter for production use).
  - **Fetch Job Status**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Queries the current status of the submitted task.
    - *Configuration Choices:* Uses GET method with header authorization.
    - *Key Expressions/Variables:* URL dynamically incorporates the job ID returned from the initial submission: `https://api.img2.video/api/video/jobs/{{ $json.jobId }}`.
    - *Input/Output Connections:* Input from `Wait 20 Seconds`; output connects to `Check Job Success`.
    - *Edge Cases/Failure Types:* Network timeouts or API temporary unresponsiveness returning 5xx server errors.
  - **Check Job Success**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Evaluates conditional logic to branch workflow execution.
    - *Configuration Choices:* Evaluates whether the API response status field matches the success criteria.
    - *Key Expressions/Variables:* Left value: `={{ $json.status }}` | Operator: `equals` | Right value: `succeeded`.
    - *Input/Output Connections:* Input from `Fetch Job Status`. True branch outputs to `Set Video URL`. False branch loops back to `Wait 20 Seconds`.
    - *Edge Cases/Failure Types:* Unexpected API status string variations (e.g., `completed` instead of `succeeded`) will cause infinite waiting loops.

#### 2.4 Output Extraction
- **Overview:** Captures and extracts the final generated video URL once the polling loop confirms success.
- **Nodes Involved:** `Set Video URL`
- **Node Details:**
  - **Set Video URL**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Extracts and normalizes the final output data.
    - *Configuration Choices:* Assigns a single string output variable.
    - *Key Expressions/Variables:* `videoUrl`: `={{ $json.downloadUrl }}`
    - *Input/Output Connections:* Input from the `true` branch of `Check Job Success`.
    - *Edge Cases/Failure Types:* Missing `downloadUrl` property in the final API payload will return empty results.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Start Trigger | `n8n-nodes-base.manualTrigger` | Manual entry point | None | Set Image and Model Parameters | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format. |
| Set Image and Model Parameters | `n8n-nodes-base.set` | Setup values for the request | Manual Start Trigger | Post Video Generation Request | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format.<br><br>## Start and configure<br><br>Manual entry point and setup values for the image-to-video request, including the image URL, model choice, prompt, and API base URL. |
| Post Video Generation Request | `n8n-nodes-base.httpRequest` | Submits asynchronous generation job | Set Image and Model Parameters | Wait 20 Seconds | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format.<br><br>## Submit generation job<br><br>Sends the initial POST request to img2.video to create the asynchronous video generation job. |
| Wait 20 Seconds | `n8n-nodes-base.wait` | Pauses between check intervals | Post Video Generation Request, Check Job Success (false) | Fetch Job Status | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format.<br><br>## Poll until complete<br><br>Waits between checks, polls the img2.video job endpoint, and branches based on whether the video generation has succeeded. If it has not succeeded yet, the workflow loops back to wait and poll again. |
| Fetch Job Status | `n8n-nodes-base.httpRequest` | Queries current job completion status | Wait 20 Seconds | Check Job Success | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format.<br><br>## Poll until complete<br><br>Waits between checks, polls the img2.video job endpoint, and branches based on whether the video generation has succeeded. If it has not succeeded yet, the workflow loops back to wait and poll again. |
| Check Job Success | `n8n-nodes-base.if` | Evaluates if generation succeeded | Fetch Job Status | Set Video URL (true), Wait 20 Seconds (false) | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format.<br><br>## Poll until complete<br><br>Waits between checks, polls the img2.video job endpoint, and branches based on whether the video generation has succeeded. If it has not succeeded yet, the workflow loops back to wait and poll again. |
| Set Video URL | `n8n-nodes-base.set` | Formats final video output URL | Check Job Success (true) | None | ## Generate an AI Video from an Image with img2.video (Grok Imagine, Kling 2.6, Veo 3)<br><br>### How it works<br><br>This workflow manually starts an image-to-video generation job with img2.video using a configured image URL, model ID, prompt, and API base URL. It submits the job, waits in 20-second intervals, polls the job status, and loops until the generation succeeds. When the video is ready, it outputs the resulting video URL.<br><br>### Setup steps<br><br>- Add the required img2.video API credential or API key to the HTTP Request nodes, typically as an authorization header.<br>- Update the Config node with your source imageUrl, desired modelId, prompt, and baseUrl values.<br>- Verify the Submit Job node sends the request body expected by the img2.video generation endpoint.<br>- Confirm the Poll Job and Succeeded? nodes reference the correct job ID, status field, and success condition from the API response.<br><br>### Customization<br><br>Change the modelId to switch between supported video models such as Grok Imagine, Kling 2.6, or Veo 3. Adjust the prompt, image URL, polling interval, and success condition to match your preferred generation behavior and API response format.<br><br>## Return video URL<br><br>Captures the completed video URL once the job succeeds, making the final generated video link available for downstream use. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Point:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) to serve as the workflow entry point.
2. **Configure Parameters:** Create an **Edit Fields (Set)** node (`n8n-nodes-base.set`) named `Set Image and Model Parameters`. Connect the Manual Trigger output to this node. Add four string assignments:
   - `imageUrl`: `https://upload.wikimedia.org/wikipedia/commons/thumb/3/3a/Cat03.jpg/640px-Cat03.jpg`
   - `modelId`: `seedance_mini`
   - `prompt`: `Gentle camera zoom in, subtle motion, cinematic lighting`
   - `baseUrl`: `https://api.img2.video`
3. **Submit Video Generation Request:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Post Video Generation Request`. Connect `Set Image and Model Parameters` to it.
   - *Method:* `POST`
   - *URL:* `https://api.img2.video/api/video/generate`
   - *Specify Body:* `JSON`
   - *JSON Body:*
     ```json
     {
       "modelId": "{{ $('Set Image and Model Parameters').item.json.modelId }}",
       "prompt": "{{ $('Set Image and Model Parameters').item.json.prompt }}",
       "imageUrl": "{{ $('Set Image and Model Parameters').item.json.imageUrl }}"
     }
     ```
   - *Headers:* Add header parameter `Authorization` with value `={{ $credentials.Authorization }}`.
   - *Credentials:* Configure an **HTTP Header Auth** credential with name `Authorization` and value `Bearer <your token>`.
4. **Configure Waiting Interval:** Add a **Wait** node (`n8n-nodes-base.wait`) named `Wait 20 Seconds`. Connect the output of `Post Video Generation Request` to it. Set the amount to `20` seconds.
5. **Fetch Job Status:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Fetch Job Status`. Connect `Wait 20 Seconds` to it.
   - *Method:* `GET`
   - *URL:* `https://api.img2.video/api/video/jobs/{{ $json.jobId }}`
   - *Headers:* Add header parameter `Authorization` with value `={{ $credentials.Authorization }}` using the same credential.
6. **Check Job Success Condition:** Add an **If** node (`n8n-nodes-base.if`) named `Check Job Success`. Connect `Fetch Job Status` to it.
   - *Condition:* Left Value `={{ $json.status }}` equals `succeeded`.
7. **Create Polling Loop and Output Branch:**
   - Connect the **false** output branch of `Check Job Success` back to the input of the `Wait 20 Seconds` node to create the polling loop.
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`) named `Set Video URL`. Connect the **true** output branch of `Check Job Success` to this node.
   - Add a string assignment: `videoUrl` = `={{ $json.downloadUrl }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| API integration requirements and credential configuration | Requires an `img2.video` account and an API token configured as an HTTP Header Auth credential (`Authorization: Bearer <token>`). |
| Supported video models for customization | The `modelId` parameter can be modified to target alternate models such as `Grok Imagine`, `Kling 2.6`, or `Veo 3`. |