Transcribe Google Drive recordings to text files with Palatine Speech

https://n8nworkflows.xyz/workflows/transcribe-google-drive-recordings-to-text-files-with-palatine-speech-20891


# Transcribe Google Drive recordings to text files with Palatine Speech

### 1. Workflow Overview

The purpose of this workflow is to automatically transcribe audio and video recordings stored in Google Drive into plain-text files using the Palatine Speech API, and then save those transcripts back into Google Drive. It removes manual transcription bottlenecks for teams handling calls, memos, or lectures, and ensures that generated transcripts do not trigger infinite loops by filtering out non-media files.

The logic is grouped into three sequential functional blocks:
- **1.1 Input Reception & Filtering:** Monitors a designated Google Drive folder for new files and filters them to allow only valid audio and video formats.
- **1.2 Transcription Processing:** Downloads the validated media file and utilizes the Palatine Speech service to convert speech into text.
- **1.3 Document Conversion & Output:** Converts the transcription text into a `.txt` file named after the original recording and uploads it to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering
- **Overview:** This block triggers whenever a new file is added to a specific Google Drive directory and evaluates the file's metadata to ensure only supported audio or video formats proceed through the workflow.
- **Nodes Involved:** 
  - `Watch Recordings Folder`
  - `Only Audio or Video Files`
- **Node Details:**
  - **Watch Recordings Folder**
    - *Type & Technical Role:* `n8n-nodes-base.googleDriveTrigger` (Google Drive Trigger node)
    - *Configuration:* Set to trigger on the `fileCreated` event by polling a specific folder (`folderToWatch`) every minute.
    - *Key Expressions:* None (uses native trigger parameters).
    - *Input/Output:* Output connects to `Only Audio or Video Files`.
    - *Version Requirements:* v1
    - *Edge Cases/Failure Types:* API rate limits, revoked Google OAuth2 tokens, or misconfigured folder selections.
  - **Only Audio or Video Files**
    - *Type & Technical Role:* `n8n-nodes-base.filter` (Filter node)
    - *Configuration:* Evaluates whether the incoming file matches standard audio/video MIME types or file extensions (`mp3`, `wav`, `m4a`, `ogg`, `oga`, `flac`, `mp4`, `webm`, `aac`, `opus`, `mov`, `mkv`).
    - *Key Expressions:* `={{ /^(audio|video)\//i.test($json.mimeType || '') || /\.(mp3|wav|m4a|ogg|oga|flac|mp4|webm|aac|opus|mov|mkv)$/i.test($json.name || '') }}`
    - *Input/Output:* Input from `Watch Recordings Folder`; output connects to `Download Recording`.
    - *Version Requirements:* v2.2
    - *Edge Cases/Failure Types:* Files with unexpected or missing MIME types will be dropped.

---

#### 2.2 Transcription Processing
- **Overview:** This block retrieves the binary content of the filtered media file from Google Drive and submits it to the Palatine Speech service for automated transcription.
- **Nodes Involved:**
  - `Download Recording`
  - `Palatine Speech Transcription`
- **Node Details:**
  - **Download Recording**
    - *Type & Technical Role:* `n8n-nodes-base.googleDrive` (Google Drive node)
    - *Configuration:* Resource: `file`, Operation: `download`. Uses the file ID from the trigger.
    - *Key Expressions:* `={{ $json.id }}`
    - *Input/Output:* Input from `Only Audio or Video Files`; output connects to `Palatine Speech Transcription`.
    - *Version Requirements:* v3
    - *Edge Cases/Failure Types:* File not found errors, insufficient storage permissions, or timeout issues on very large media files.
  - **Palatine Speech Transcription**
    - *Type & Technical Role:* `n8n-nodes-palatine-speech.palatineSpeech` (Community node)
    - *Configuration:* Resource: `audio`, Operation: `transcribe`, Model: `palatine_large_highspeed`, Binary Property: `data`.
    - *Key Expressions:* Processes the binary data property to populate `$json.transcription`.
    - *Input/Output:* Input from `Download Recording`; output connects to `Transcript to .txt File`.
    - *Version Requirements:* v3 (requires community package `n8n-nodes-palatine-speech`).
    - *Edge Cases/Failure Types:* API authentication errors, service downtime, invalid audio codecs, or execution timeouts on long files.

---

#### 2.3 Document Conversion & Output
- **Overview:** This block transforms the raw transcription string into a structured text file bearing the original file's name and uploads the resulting asset to Google Drive.
- **Nodes Involved:**
  - `Transcript to .txt File`
  - `Save Transcript to Drive`
- **Node Details:**
  - **Transcript to .txt File**
    - *Type & Technical Role:* `n8n-nodes-base.convertToFile` (Convert to File node)
    - *Configuration:* Operation: `toText`, Source Property: `transcription`, Binary Property Name: `data`.
    - *Key Expressions:* `={{ $('Watch Recordings Folder').item.json.name.replace(/\.[^.]+$/, '') }}.txt`
    - *Input/Output:* Input from `Palatine Speech Transcription`; output connects to `Save Transcript to Drive`.
    - *Version Requirements:* v1.1
    - *Edge Cases/Failure Types:* Empty transcription strings or missing filename properties.
  - **Save Transcript to Drive**
    - *Type & Technical Role:* `n8n-nodes-base.googleDrive` (Google Drive node)
    - *Configuration:* Resource: `file`, Operation: `upload` (default upload action via `inputDataFieldName`), Drive ID: `My Drive`, Input Data Field Name: `data`.
    - *Key Expressions:* `={{ $('Watch Recordings Folder').item.json.name.replace(/\.[^.]+$/, '') }}.txt`
    - *Input/Output:* Input from `Transcript to .txt File`; no downstream connections.
    - *Version Requirements:* v3
    - *Edge Cases/Failure Types:* Write permission restrictions on the target folder, duplicate file name conflicts, or credential expiration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Watch Recordings Folder` | `n8n-nodes-base.googleDriveTrigger` | Polls Google Drive for new recordings | None | `Only Audio or Video Files` | # Save Google Drive recording transcripts as text files with Palatine Speech<br><br>## Who is this for?<br>Teams and individuals who accumulate recordings in Drive (calls, memos, lectures) and want each one searchable as text without opening any tool.<br><br>## What it does<br>Drop recordings into an input Google Drive folder and save their plain-text transcripts into a separate output folder. Palatine Speech handles transcription; a media filter ignores text and other non-media files.<br><br>## How it works<br>1. **Watch Recordings Folder** — Google Drive Trigger polls a folder for new files.<br>2. **Download Recording** — downloads the new audio/video file.<br>3. **Palatine Speech Transcription** — converts speech to text (`palatine_large_highspeed`, `$json.transcription`).<br>4. **Transcript to .txt File** — wraps the transcript into a text file named after the recording.<br>5. **Save Transcript to Drive** — uploads `<recording>.txt` back to the same folder.<br><br>## Setup<br>1. Install community node `n8n-nodes-palatine-speech`, import `workflow.json`.<br>2. Attach Google Drive + Palatine Speech credentials.<br>3. Pick the folder to watch in the trigger and the same (or another) folder as the upload target.<br>4. Activate and drop a recording into the folder.<br><br>Select an input recordings folder and a separate output transcripts folder in Save Transcript to Drive. The media filter also ignores `.txt` files, preventing generated transcripts from triggering repeated transcription.<br><br>## Requirements<br>Install the community node `n8n-nodes-palatine-speech` (tested package version 1.1.1). On self-hosted n8n, use Settings → Community Nodes. On n8n Cloud, use the node picker if the verified package is available to your workspace. Add credentials in n8n; no secrets are included in the template.<br><br>## Customization<br>- Select a dedicated output folder for transcripts.<br>- Add a summary step after transcription.<br>- Send the resulting file link to your team.<br><br>### Need help with Palatine Speech?<br><br>Check info here: [speech.palatine.ru](https://speech.palatine.ru/)<br><br>Documentation: [docs.speech.palatine.ru](https://docs.speech.palatine.ru/)<br><br>## 1. Receive input<br>Connect the trigger and review the expected input. |
| `Only Audio or Video Files` | `n8n-nodes-base.filter` | Filters out non-audio/video payloads | `Watch Recordings Folder` | `Download Recording` | # Save Google Drive recording transcripts as text files with Palatine Speech<br>... *(see full note above)* ...<br><br>## 1. Receive input<br>Connect the trigger and review the expected input. |
| `Download Recording` | `n8n-nodes-base.googleDrive` | Downloads target file binary from Drive | `Only Audio or Video Files` | `Palatine Speech Transcription` | # Save Google Drive recording transcripts as text files with Palatine Speech<br>... *(see full note above)* ... |
| `Palatine Speech Transcription` | `n8n-nodes-palatine-speech.palatineSpeech` | Converts audio/video binary to text | `Download Recording` | `Transcript to .txt File` | # Save Google Drive recording transcripts as text files with Palatine Speech<br>... *(see full note above)* ...<br><br>## 2. Process with Palatine Speech<br>Select your credential; inspect the transcription or summary. |
| `Transcript to .txt File` | `n8n-nodes-base.convertToFile` | Formats transcript into a text document | `Palatine Speech Transcription` | `Save Transcript to Drive` | # Save Google Drive recording transcripts as text files with Palatine Speech<br>... *(see full note above)* ... |
| `Save Transcript to Drive` | `n8n-nodes-base.googleDrive` | Uploads generated `.txt` file to Drive | `Transcript to .txt File` | None | # Save Google Drive recording transcripts as text files with Palatine Speech<br>... *(see full note above)* ...<br><br>## 3. Deliver results<br>Configure the destination and test before activation. |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Community Node:** Ensure `n8n-nodes-palatine-speech` is installed via **Settings > Community Nodes** (or available in your workspace node picker).
2. **Create Node 1 (`Watch Recordings Folder`):**
   - Type: `n8n-nodes-base.googleDriveTrigger`
   - Parameters: Set **Event** to `fileCreated`, **Trigger On** to `Specific Folder`, and configure polling to run every minute. Link your Google Drive OAuth2 credential. Select the target folder to monitor.
3. **Create Node 2 (`Only Audio or Video Files`):**
   - Type: `n8n-nodes-base.filter`
   - Parameters: Create an expression condition checking MIME types or file extensions: 
     `={{ /^(audio|video)\//i.test($json.mimeType || '') || /\.(mp3|wav|m4a|ogg|oga|flac|mp4|webm|aac|opus|mov|mkv)$/i.test($json.name || '') }}`
   - Connect `Watch Recordings Folder` (Main) output to `Only Audio or Video Files` (Main).
4. **Create Node 3 (`Download Recording`):**
   - Type: `n8n-nodes-base.googleDrive`
   - Parameters: Set **Resource** to `file`, **Operation** to `download`. Set **File ID** expression to: `={{ $json.id }}`.
   - Connect `Only Audio or Video Files` (Main) output to `Download Recording` (Main).
5. **Create Node 4 (`Palatine Speech Transcription`):**
   - Type: `n8n-nodes-palatine-speech.palatineSpeech`
   - Parameters: Set **Resource** to `audio`, **Operation** to `transcribe`, **Model** to `palatine_large_highspeed`, and **Binary Property** to `data`. Link your Palatine Speech API credential.
   - Connect `Download Recording` (Main) output to `Palatine Speech Transcription` (Main).
6. **Create Node 5 (`Transcript to .txt File`):**
   - Type: `n8n-nodes-base.convertToFile`
   - Parameters: Set **Operation** to `To Text`, **Source Property** to `transcription`, and **Binary Property Name** to `data`. Set **File Name** under options to: `={{ $('Watch Recordings Folder').item.json.name.replace(/\.[^.]+$/, '') }}.txt`.
   - Connect `Palatine Speech Transcription` (Main) output to `Transcript to .txt File` (Main).
7. **Create Node 6 (`Save Transcript to Drive`):**
   - Type: `n8n-nodes-base.googleDrive`
   - Parameters: Set **Resource** to `file`, **Operation** to `upload`, **Drive ID** to `My Drive`, **Folder ID** to your desired output folder, **Input Data Field Name** to `data`, and **Name** to `={{ $('Watch Recordings Folder').item.json.name.replace(/\.[^.]+$/, '') }}.txt`. Link your Google Drive OAuth2 credential.
   - Connect `Transcript to .txt File` (Main) output to `Save Transcript to Drive` (Main).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Palatine Speech Platform Information | [speech.palatine.ru](https://speech.palatine.ru/) |
| Palatine Speech Official Documentation | [docs.speech.palatine.ru](https://docs.speech.palatine.ru/) |