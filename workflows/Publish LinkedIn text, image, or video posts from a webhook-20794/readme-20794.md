Publish LinkedIn text, image, or video posts from a webhook

https://n8nworkflows.xyz/workflows/publish-linkedin-text--image--or-video-posts-from-a-webhook-20794


# Publish LinkedIn text, image, or video posts from a webhook

### 1. Workflow Overview

This workflow automates the publishing of LinkedIn posts containing text, images (up to 20), or native video via a single webhook entry point. It downloads media files from provided URLs, handles LinkedIn’s asset registration and binary upload protocols, publishes the content utilizing LinkedIn's REST and UGC APIs, and returns the newly created post URL or a descriptive error response.

The workflow logic is divided into four distinct functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming HTTP POST requests, injects user configuration variables, and validates payload requirements.
- **1.2 Routing & Media Handling:** Determines whether the post requires a video, multiple/single images, or text-only, executing the corresponding registration, file downloading, and binary upload cycles.
- **1.3 Post Creation & Evaluation:** Builds appropriate API payloads based on media type, submits them to LinkedIn, and evaluates the HTTP status response.
- **1.4 Response & Error Handling:** Extracts the post URN to form the public URL for successful executions, or normalizes error outputs into an HTTP 500 response payload for failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block listens for webhook triggers, injects default author and API version configurations, and rigorously checks input parameters to fail fast on malformed requests.

- **Receive Post Request**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Trigger)
  - **Configuration Choices:** Configured to listen for HTTP `POST` requests on the path `linkedin-publish` using response mode `responseNode`.
  - **Key Expressions / Variables:** N/A (Entry point)
  - **Input / Output Connections:** Input: External HTTP request | Output: `Set LinkedIn Settings`
  - **Edge Cases / Failure Types:** Invalid URL path, unsupported HTTP methods (e.g., GET instead of POST).

- **Set LinkedIn Settings**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data transformation)
  - **Configuration Choices:** Assigns static default variables for `author_id` (`YOUR_LINKEDIN_PERSON_ID`) and `linkedin_version` (`202608`) while passing through other incoming fields.
  - **Key Expressions / Variables:** `YOUR_LINKEDIN_PERSON_ID`, `202608`
  - **Input / Output Connections:** Input: `Receive Post Request` | Output: `Validate and Normalize Input`
  - **Edge Cases / Failure Types:** Failure to update placeholder credentials.

- **Validate and Normalize Input**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data processing)
  - **Configuration Choices:** Executes custom JavaScript to parse JSON body payloads, trimming whitespace and validating mandatory constraints.
  - **Key Expressions / Variables:** Uses `{{ $input.first().json }}` to read incoming body parameters (`text`, `author_id`, `image_urls`, `video_url`).
  - **Input / Output Connections:** Input: `Set LinkedIn Settings` | Output: Main output connects to `Is Video?`; Error output connects to `Build Error Message`.
  - **Edge Cases / Failure Types:** Missing `text` field, unconfigured `author_id` placeholder, exceeding the 20-image limit, or supplying both images and video simultaneously.

---

#### 2.2 Routing & Media Handling
This block directs the normalized payload down specific logic tracks depending on whether the post contains a video, images, or purely text.

- **Is Video?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control)
  - **Configuration Choices:** Evaluates if the `video_url` property is not empty.
  - **Key Expressions / Variables:** `={{ $json.video_url }}`
  - **Input / Output Connections:** Input: `Validate and Normalize Input` | Output (True): `Register Video Upload` | Output (False): `Has Images?`
  - **Edge Cases / Failure Types:** Malformed URL strings evaluating as truthy.

- **Has Images?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control)
  - **Configuration Choices:** Checks if the `image_urls` array length is greater than or equal to 1.
  - **Key Expressions / Variables:** `={{ $json.image_urls.length }}`
  - **Input / Output Connections:** Input: `Is Video?` (False branch) | Output (True): `Split Images` | Output (False): `Build Text Post`
  - **Edge Cases / Failure Types:** Empty or unparsed arrays.

- **Register Video Upload**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API integration)
  - **Configuration Choices:** POST request to LinkedIn's assets API (`registerUpload`) utilizing predefined OAuth2 credentials and custom RESTli protocol headers.
  - **Key Expressions / Variables:** JSON body referencing `{{ $('Validate and Normalize Input').first().json.author_urn }}`.
  - **Input / Output Connections:** Input: `Is Video?` (True branch) | Output: `Download Video` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Expired OAuth tokens, invalid author URN, LinkedIn API rate limits.

- **Download Video**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network request)
  - **Configuration Choices:** GET request targeting the remote video source URL, saving output as binary file data into `data`.
  - **Key Expressions / Variables:** `={{ $('Validate and Normalize Input').first().json.video_url }}`
  - **Input / Output Connections:** Input: `Register Video Upload` | Output: `Upload Video to LinkedIn` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Unreachable remote URLs, timeouts on large video files, unsupported binary formats.

- **Upload Video to LinkedIn**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API integration)
  - **Configuration Choices:** POST request to the dynamic upload URL returned by LinkedIn registration, sending binary input data with header `media-type-family: VIDEO`.
  - **Key Expressions / Variables:** `={{ $('Register Video Upload').first().json.value.uploadMechanism['com.linkedin.digitalmedia.uploading.MediaUploadHttpRequest'].uploadUrl }}`
  - **Input / Output Connections:** Input: `Download Video` | Output: `Create Video Post` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Network interruption during large payload transfer, rejected binary signatures.

- **Split Images**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data processing)
  - **Configuration Choices:** JavaScript map operation transforming an array of image URLs into individual discrete items for iteration.
  - **Key Expressions / Variables:** `{{ $('Validate and Normalize Input').first().json }}`
  - **Input / Output Connections:** Input: `Has Images?` (True branch) | Output: `Register Image Upload`
  - **Edge Cases / Failure Types:** Empty array iteration anomalies.

- **Register Image Upload**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API integration)
  - **Configuration Choices:** POST request to LinkedIn's initialize upload endpoint (`/rest/images?action=initializeUpload`) using dynamic API version headers.
  - **Key Expressions / Variables:** `={{ $('Validate and Normalize Input').first().json.api_version }}`
  - **Input / Output Connections:** Input: `Split Images` | Output: `Download Image` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Invalid API version headers, unauthorized accounts.

- **Download Image**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network request)
  - **Configuration Choices:** GET request retrieving binary image data for each split item, mapping output to file property `data`.
  - **Key Expressions / Variables:** `={{ $('Split Images').item.json.url }}`
  - **Input / Output Connections:** Input: `Register Image Upload` | Output: `Upload Image to LinkedIn` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Broken image links, unsupported file types (e.g., WebP incompatibility depending on LinkedIn version).

- **Upload Image to LinkedIn**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API integration)
  - **Configuration Choices:** PUT request uploading binary image data to the specific initialization upload URL provided by LinkedIn.
  - **Key Expressions / Variables:** `={{ $('Register Image Upload').item.json.value.uploadUrl }}`
  - **Input / Output Connections:** Input: `Download Image` | Output: `Build Image Post` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Upload URL expiration, payload size limits.

---

#### 2.3 Post Creation & Evaluation
This block compiles processed assets into formatted JSON payloads, dispatches creation requests to LinkedIn, and evaluates HTTP response codes.

- **Build Image Post**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data processing)
  - **Configuration Choices:** JavaScript validation ensuring all split images successfully uploaded, applying escaping rules for LinkedIn's text formatting format, and building a single or multi-image payload structure.
  - **Key Expressions / Variables:** Reads item counts across `Split Images`, `Register Image Upload`, and inputs.
  - **Input / Output Connections:** Input: `Upload Image to LinkedIn` | Output: `Create Post`
  - **Edge Cases / Failure Types:** Partial upload failures throwing mismatch errors between expected and uploaded counts.

- **Build Text Post**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data processing)
  - **Configuration Choices:** JavaScript escaping utility formatting commentary strings for text-only posts destined for the Posts API.
  - **Key Expressions / Variables:** `{{ $('Validate and Normalize Input').first().json }}`
  - **Input / Output Connections:** Input: `Has Images?` (False branch) | Output: `Create Post`
  - **Edge Cases / Failure Types:** Unescaped reserved characters causing layout breaks.

- **Create Video Post**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API integration)
  - **Configuration Choices:** POST request to `/v2/ugcPosts` using UGC API specifications, configured never to throw errors natively to allow status evaluation downstream.
  - **Key Expressions / Variables:** References author URN, commentary text, and asset URN from `Register Video Upload`.
  - **Input / Output Connections:** Input: `Upload Video to LinkedIn` | Output: `Accepted by LinkedIn?` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Asset processing delays on LinkedIn's servers.

- **Create Post**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API integration)
  - **Configuration Choices:** POST request to LinkedIn's REST posts endpoint (`/rest/posts`) using dynamic version headers and returning full response parameters without throwing errors.
  - **Key Expressions / Variables:** `={{ JSON.stringify($json.payload) }}`
  - **Input / Output Connections:** Input: `Build Image Post`, `Build Text Post` | Output: `Accepted by LinkedIn?` (Error branch to `Build Error Message`).
  - **Edge Cases / Failure Types:** Payload validation rejections from LinkedIn.

- **Accepted by LinkedIn?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control)
  - **Configuration Choices:** Evaluates whether the HTTP status code returned by the post creation step is strictly less than 300.
  - **Key Expressions / Variables:** `={{ $json.statusCode }}`
  - **Input / Output Connections:** Input: `Create Post`, `Create Video Post` | Output (True): `Extract Post URL` | Output (False): `Build Error Message`
  - **Edge Cases / Failure Types:** Non-2xx status codes passing unhandled.

---

#### 2.4 Response & Error Handling
This block formats successful responses with generated public URLs or normalizes failure reasons into structured error payloads.

- **Extract Post URL**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data processing)
  - **Configuration Choices:** Extracts the resource identifier from the `x-restli-id` response header and constructs the public LinkedIn update URL.
  - **Key Expressions / Variables:** `{{ $input.first().json.headers }}`
  - **Input / Output Connections:** Input: `Accepted by LinkedIn?` (True branch) | Output: `Respond with Post URL`
  - **Edge Cases / Failure Types:** Missing response headers throwing missing ID errors.

- **Respond with Post URL**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook response)
  - **Configuration Choices:** Responds with a JSON object containing the generated public URL and URN.
  - **Key Expressions / Variables:** `{{ JSON.stringify({ url: $json.url, urn: $json.urn }) }}`
  - **Input / Output Connections:** Input: `Extract Post URL` | Output: None (Terminal node)
  - **Edge Cases / Failure Types:** Closed incoming socket connections.

- **Build Error Message**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data processing)
  - **Configuration Choices:** Normalizes errors originating from either HTTP failure responses or workflow node exceptions into a standardized object shape.
  - **Key Expressions / Variables:** `{{ $input.first().json }}`
  - **Input / Output Connections:** Input: Error outputs from multiple nodes | Output: `Respond with Error`
  - **Edge Cases / Failure Types:** Unstructured error objects falling back to generic descriptions.

- **Respond with Error**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Webhook response)
  - **Configuration Choices:** Responds to the webhook caller with an HTTP 500 status code and a JSON error payload.
  - **Key Expressions / Variables:** `{{ JSON.stringify({ error: $json.error }) }}`
  - **Input / Output Connections:** Input: `Build Error Message` | Output: None (Terminal node)
  - **Edge Cases / Failure Types:** Response timeout.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Receive Post Request** | `n8n-nodes-base.webhook` | Webhook trigger for incoming POST requests | None | Set LinkedIn Settings | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it. |
| **Set LinkedIn Settings** | `n8n-nodes-base.set` | Assigns default author ID and API version variables | Receive Post Request | Validate and Normalize Input | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 1. Receive and validate<br>Settings and input checks run first, so a bad request fails fast with a readable error instead of half a post. |
| **Validate and Normalize Input** | `n8n-nodes-base.code` | Validates required fields, limits, and mutual exclusivity | Set LinkedIn Settings | Is Video?, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 1. Receive and validate<br>Settings and input checks run first, so a bad request fails fast with a readable error instead of half a post. |
| **Is Video?** | `n8n-nodes-base.if` | Routes execution based on presence of video URL | Validate and Normalize Input | Register Video Upload, Has Images? | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it. |
| **Has Images?** | `n8n-nodes-base.if` | Routes execution based on presence of image URLs | Is Video? | Split Images, Build Text Post | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Register Video Upload** | `n8n-nodes-base.httpRequest` | Registers a video upload session with LinkedIn | Is Video? | Download Video, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Download Video** | `n8n-nodes-base.httpRequest` | Downloads the video binary file from the external URL | Register Video Upload | Upload Video to LinkedIn, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Upload Video to LinkedIn** | `n8n-nodes-base.httpRequest` | Uploads the downloaded video binary to LinkedIn | Download Video | Create Video Post, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Create Video Post** | `n8n-nodes-base.httpRequest` | Submits the native video UGC post payload to LinkedIn | Upload Video to LinkedIn | Accepted by LinkedIn?, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Split Images** | `n8n-nodes-base.code` | Splits an array of image URLs into discrete items | Has Images? | Register Image Upload | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Register Image Upload** | `n8n-nodes-base.httpRequest` | Initializes an image upload session with LinkedIn | Split Images | Download Image, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Download Image** | `n8n-nodes-base.httpRequest` | Downloads binary image data for each split URL | Register Image Upload | Upload Image to LinkedIn, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Upload Image to LinkedIn** | `n8n-nodes-base.httpRequest` | Uploads binary image data to the initialization upload URL | Download Image | Build Image Post, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 2. Upload media<br>Images are registered with LinkedIn, downloaded from your URLs and uploaded one by one. A video uses the assets upload in a single request. |
| **Build Image Post** | `n8n-nodes-base.code` | Compiles uploaded image assets into a Posts API payload | Upload Image to LinkedIn | Create Post | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Build Text Post** | `n8n-nodes-base.code` | Formats commentary for text-only Posts API payloads | Has Images? | Create Post | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Create Post** | `n8n-nodes-base.httpRequest` | Submits the text or image post payload to LinkedIn's REST API | Build Image Post, Build Text Post | Accepted by LinkedIn?, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Accepted by LinkedIn?** | `n8n-nodes-base.if` | Verifies whether LinkedIn accepted the post creation request | Create Post, Create Video Post | Extract Post URL, Build Error Message | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Extract Post URL** | `n8n-nodes-base.code` | Extracts post URN from response headers to build public URL | Accepted by LinkedIn? | Respond with Post URL | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Respond with Post URL** | `n8n-nodes-base.respondToWebhook` | Returns the public post URL and URN via webhook response | Extract Post URL | None | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Build Error Message** | `n8n-nodes-base.code` | Normalizes various failure inputs into a single error format | Multiple error paths | Respond with Error | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |
| **Respond with Error** | `n8n-nodes-base.respondToWebhook` | Responds to the webhook caller with an HTTP 500 error payload | Build Error Message | None | ## Publish LinkedIn posts with images or video from one webhook<br><br>**How it works**<br>- A webhook receives `text` plus optional `image_urls` (1 to 20) or one `video_url`.<br>- The workflow picks the route: text only, single image, multi-image post (Posts API) or native video (upload, then post).<br>- Media is downloaded from your URLs and uploaded to LinkedIn, so nothing has to be stored first.<br>- The response contains the public post URL, or HTTP 500 with LinkedIn's reason.<br><br>**Setup steps**<br>1. Create a LinkedIn app with the *Share on LinkedIn* product and an n8n **LinkedIn OAuth2 API** credential (scope `w_member_social`). Attach it to every LinkedIn HTTP node.<br>2. In "Set LinkedIn Settings" enter your person ID as `author_id` (the `sub` value of `GET https://api.linkedin.com/v2/userinfo`).<br>3. POST e.g. `{\"text\": \"Hello LinkedIn\", \"image_urls\": [\"https://example.com/a.png\"]}` to the webhook.<br><br>**Good to know**<br>- The native LinkedIn node cannot post multi-image carousels or native video, hence HTTP Request.<br>- `Linkedin-Version` is a monthly API version. Update it in the settings node when LinkedIn retires yours.<br>- Text is escaped for LinkedIn's "little" format, so `(`, `*` or `_` show up literally.<br>- Protect the webhook with Header Auth before you expose it.<br><br>## 3. Create the post and respond<br>Posts API for text and images, ugcPosts for video. Success counts only when LinkedIn returns a post ID. Every failure ends in one error response. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Credentials
1. Create a LinkedIn app via the LinkedIn Developer Portal with the **Share on LinkedIn** product enabled.
2. In n8n, create a new **LinkedIn OAuth2 API** credential using the required scope (`w_member_social`).

#### Step 2: Create Trigger and Configuration Nodes
1. **Receive Post Request (`n8n-nodes-base.webhook`)**
   - **Parameters:** Set `HTTP Method` to `POST`, path to `linkedin-publish`, and `Response Mode` to `Response Node`.
2. **Set LinkedIn Settings (`n8n-nodes-base.set`)**
   - **Parameters:** Add assignments: `author_id` (string) = `YOUR_LINKEDIN_PERSON_ID`, `linkedin_version` (string) = `202608`. Enable `Include Other Fields`.
   - **Connection:** Connect `Receive Post Request` output to this node.
3. **Validate and Normalize Input (`n8n-nodes-base.code`)**
   - **Parameters:** Set `Mode` to Run Once for All Items. Insert the validation JavaScript snippet handling body parsing, validation rules (text required, image count $\le 20$, mutual exclusivity of images vs video), and author URN formatting (`urn:li:person:`).
   - **Settings:** Set `On Error` to `Continue Using Error Output`.
   - **Connection:** Connect `Set LinkedIn Settings` output to this node.

#### Step 3: Implement Routing & Media Path Nodes
1. **Is Video? (`n8n-nodes-base.if`)**
   - **Parameters:** Create condition where `{{ $json.video_url }}` is not empty.
   - **Connection:** Connect main output of `Validate and Normalize Input` here.
2. **Has Images? (`n8n-nodes-base.if`)**
   - **Parameters:** Create condition where `{{ $json.image_urls.length }}` is greater than or equal to `1`.
   - **Connection:** Connect `Is Video?` false output here.
3. **Video Branch Nodes:**
   - **Register Video Upload (`n8n-nodes-base.httpRequest`)**: POST to `https://api.linkedin.com/v2/assets?action=registerUpload`. Set Authentication to `LinkedIn OAuth2 API`. Add header `X-Restli-Protocol-Version` = `2.0.0`. Specify JSON body containing `registerUploadRequest` referencing `urn:li:digitalmediaRecipe:feedshare-video` and owner URN. Enable `On Error` -> `Continue Using Error Output`.
   - **Download Video (`n8n-nodes-base.httpRequest`)**: GET request to `{{ $('Validate and Normalize Input').first().json.video_url }}` with response format set to file (`outputPropertyName`: `data`). Enable `On Error` -> `Continue Using Error Output`.
   - **Upload Video to LinkedIn (`n8n-nodes-base.httpRequest`)**: POST request to upload URL from previous step. Set content type to `binaryData`, input data field name to `data`, authentication to `LinkedIn OAuth2 API`, and header `media-type-family` = `VIDEO`. Enable `On Error` -> `Continue Using Error Output`.
   - **Create Video Post (`n8n-nodes-base.httpRequest`)**: POST to `https://api.linkedin.com/v2/ugcPosts`. Authentication: `LinkedIn OAuth2 API`. Header `X-Restli-Protocol-Version` = `2.0.0`. Specify JSON body with UGC share content (`shareMediaCategory: 'VIDEO'`). Enable `Never Error` and `Full Response` in options. Enable `On Error` -> `Continue Using Error Output`.
4. **Image Branch Nodes:**
   - **Split Images (`n8n-nodes-base.code`)**: JavaScript mapping over `image_urls` to produce one item per image.
   - **Register Image Upload (`n8n-nodes-base.httpRequest`)**: POST to `https://api.linkedin.com/rest/images?action=initializeUpload`. Authentication: `LinkedIn OAuth2 API`. Headers: `Linkedin-Version` = `{{ $('Validate and Normalize Input').first().json.api_version }}`, `X-Restli-Protocol-Version` = `2.0.0`. Specify JSON body with `initializeUploadRequest`. Enable `On Error` -> `Continue Using Error Output`.
   - **Download Image (`n8n-nodes-base.httpRequest`)**: GET request to `{{ $('Split Images').item.json.url }}` with response format set to file (`outputPropertyName`: `data`). Enable `On Error` -> `Continue Using Error Output`.
   - **Upload Image to LinkedIn (`n8n-nodes-base.httpRequest`)**: PUT request to `{{ $('Register Image Upload').item.json.value.uploadUrl }}` with content type `binaryData` and input data field name `data`. Authentication: `LinkedIn OAuth2 API`. Enable `On Error` -> `Continue Using Error Output`.
   - **Build Image Post (`n8n-nodes-base.code`)**: JavaScript code validating uploaded counts matching expected totals and formatting Posts API multi/single-image payloads with escaped text formatting. Enable `On Error` -> `Continue Using Error Output`.
5. **Text-Only Branch Nodes:**
   - **Build Text Post (`n8n-nodes-base.code`)**: JavaScript code building a text-only Posts API payload with escaped commentary formatting.

#### Step 4: Implement Post Creation & Response Nodes
1. **Create Post (`n8n-nodes-base.httpRequest`)**
   - **Parameters:** POST request to `https://api.linkedin.com/rest/posts`. Authentication: `LinkedIn OAuth2 API`. Headers: `Linkedin-Version` = `{{ $('Validate and Normalize Input').first().json.api_version }}`, `X-Restli-Protocol-Version` = `2.0.0`. Body type: JSON (`{{ JSON.stringify($json.payload) }}`). Enable `Never Error` and `Full Response` in options.
   - **Settings:** `On Error` -> `Continue Using Error Output`.
   - **Connections:** Connect outputs from `Build Image Post` and `Build Text Post` here.
2. **Accepted by LinkedIn? (`n8n-nodes-base.if`)**
   - **Parameters:** Condition checking if `{{ $json.statusCode }}` is less than `300`.
   - **Connections:** Connect outputs from `Create Post` and `Create Video Post` here.
3. **Extract Post URL (`n8n-nodes-base.code`)**
   - **Parameters:** JavaScript code extracting `x-restli-id` header to form `https://www.linkedin.com/feed/update/{urn}`.
   - **Connection:** Connect true branch of `Accepted by LinkedIn?`.
4. **Respond with Post URL (`n8n-nodes-base.respondToWebhook`)**
   - **Parameters:** Respond with JSON: `{{ JSON.stringify({ url: $json.url, urn: $json.urn }) }}`.
   - **Connection:** Connect `Extract Post URL` output here.
5. **Build Error Message & Respond with Error (`n8n-nodes-base.code` & `n8n-nodes-base.respondToWebhook`)**
   - **Parameters (Code):** Normalizes errors from `Validate and Normalize Input`, HTTP upload steps, or API rejections.
   - **Parameters (Respond to Webhook):** Response Code: `500`, Respond with JSON: `{{ JSON.stringify({ error: $json.error }) }}`.
   - **Connection:** Connect error outputs from all primary operational nodes into `Build Error Message`, then connect to `Respond with Error`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure you replace `YOUR_LINKEDIN_PERSON_ID` in the settings node with your actual LinkedIn person URN numeric identifier (obtained via `GET https://api.linkedin.com/v2/userinfo`). | Configuration prerequisite |
| Update the `Linkedin-Version` monthly version string (e.g., `202608`) whenever LinkedIn deprecates your active API version. | API lifecycle management |
| Protect the webhook URL with header authentication or a webhook secret token before exposing it to external callers. | Security best practice |