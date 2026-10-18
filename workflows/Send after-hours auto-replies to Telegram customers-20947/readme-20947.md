Send after-hours auto-replies to Telegram customers

https://n8nworkflows.xyz/workflows/send-after-hours-auto-replies-to-telegram-customers-20947


# Send after-hours auto-replies to Telegram customers

### 1. Workflow Overview

The purpose of the **"Send after-hours auto-replies to Telegram customers"** workflow is to automate out-of-office responses for small businesses, freelancers, and solo founders managing customer inquiries via Telegram. By capturing incoming messages, evaluating them against configured operational hours, and conditionally firing an automated response, the workflow ensures that customers receive immediate acknowledgment even during nights or weekends. 

The workflow logic is categorized into a single sequential processing pipeline:
- **1.1 Event Inception & Configuration:** Captures incoming Telegram chat messages and establishes operational parameters (business hours, active days, and response copy).
- **1.2 Temporal Evaluation & Conditional Routing:** Computes the current server time against the business rules to flag after-hours traffic and filters out messages falling within operational hours.
- **1.3 Automated Action Execution:** Dispatches the custom reply text directly back to the customer's Telegram chat identifier if the message arrived outside business hours.

---

### 2. Block-by-Block Analysis

---

#### Block 1: Event Inception & Configuration

##### Overview
This block initiates the automation sequence by listening for live incoming messages sent to the designated Telegram bot and appending static configuration parameters required for time-based evaluation.

##### Nodes Involved
- `Telegram Trigger (New Message)`
- `Set Business Hours Config`

##### Node Details

###### Telegram Trigger (New Message)
- **Type and Technical Role:** `n8n-nodes-base.telegramTrigger` (Trigger Node) - Listens for webhooks or long-polling updates from Telegram based on configured event types.
- **Configuration Choices:** Configured to watch for `"message"` updates.
- **Key Expressions or Variables:** None (consumes native incoming webhook payload).
- **Input and Output Connections:** 
  - Input: None (Workflow entry point)
  - Output: Connects to `Set Business Hours Config` via main execution output `0`.
- **Version-Specific Requirements:** Version 1.5. Requires active Telegram Bot API credentials.
- **Edge Cases / Potential Failure Types:** API connectivity drops, invalid bot token credentials, or Telegram servers failing to deliver webhook updates.
- **Sub-Workflow Reference:** None.

###### Set Business Hours Config
- **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation / Setup Node) - Injects manual variable assignments into the data stream.
- **Configuration Choices:** Mode set to `manual`. Appends predefined operational configuration fields: `open_hour` (9), `close_hour` (18), `workday_start` (1), `workday_end` (5), and `reply_message` (custom string). Preserves incoming item fields (`includeOtherFields: true`).
- **Key Expressions or Variables:** Static numbers and string assignments.
- **Input and Output Connections:** 
  - Input: Receives payload from `Telegram Trigger (New Message)`.
  - Output: Connects to `Check Business Hours` via main execution output `0`.
- **Version-Specific Requirements:** Version 3.4.
- **Edge Cases / Potential Failure Types:** Type mismatches if downstream nodes expect string evaluations instead of numbers (handled via explicit parsing in subsequent script nodes).
- **Sub-Workflow Reference:** None.

---

#### Block 2: Temporal Evaluation & Conditional Routing

##### Overview
This block executes a JavaScript computation to determine if the message arrival time falls outside established working parameters, subsequently routing or filtering the item payload.

##### Nodes Involved
- `Check Business Hours`
- `IF After Hours`

##### Node Details

###### Check Business Hours
- **Type and Technical Role:** `n8n-nodes-base.code` (Scripting Node) - Evaluates execution time against configuration parameters using custom JavaScript code.
- **Configuration Choices:** Mode set to `runOnceForAllItems`. Executes a script mapping input properties, parsing hours, weekdays, and utilizing `$now` instance variables to output a boolean flag (`is_business_hours`).
- **Key Expressions or Variables:** 
  - `$input.all()`
  - `$now.hour`
  - `$now.weekday`
  - Optional fallback chaining operators (`??`).
- **Input and Output Connections:** 
  - Input: Receives consolidated payload from `Set Business Hours Config`.
  - Output: Connects to `IF After Hours` via main execution output `0`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Timezone mismatches between the local server time (`$now`) and the business's actual operating timezone; missing structural properties in the upstream Telegram payload (mitigated by optional chaining `?.`).
- **Sub-Workflow Reference:** None.

###### IF After Hours
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control Node) - Splits the execution path based on boolean evaluation conditions.
- **Configuration Choices:** Uses version 2 condition evaluation. Evaluates whether `{{ $json.is_business_hours }}` evaluates to `false`.
- **Key Expressions or Variables:** `={{ $json.is_business_hours }}`
- **Input and Output Connections:** 
  - Input: Receives evaluated items from `Check Business Hours`.
  - Output: 
    - `true` path (After Hours): Connects to `Send Auto-Reply` via main output `0`.
    - `false` path (Business Hours): Unconnected (terminates execution path).
- **Version-Specific Requirements:** Version 2.2.
- **Edge Cases / Potential Failure Types:** Data parsing failures if upstream properties are missing or undefined.
- **Sub-Workflow Reference:** None.

---

#### Block 3: Automated Action Execution

##### Overview
This block handles the final dispatch of the automated text message back to the customer via the Telegram API if the conversation falls into the after-hours execution branch.

##### Nodes Involved
- `Send Auto-Reply`

##### Node Details

###### Send Auto-Reply
- **Type and Technical Role:** `n8n-nodes-base.telegram` (Action Node) - Sends text messages to chats using the Telegram Bot API.
- **Configuration Choices:** Resource set to `message`, operation set to `sendMessage`.
- **Key Expressions or Variables:** 
  - Text parameter: `={{ $json.reply_message }}`
  - Chat ID parameter: `={{ $json.chat_id }}`
- **Input and Output Connections:** 
  - Input: Receives filtered after-hours items from the `true` output of `IF After Hours`.
  - Output: None (Terminal node).
- **Version-Specific Requirements:** Version 1.2. Requires active Telegram Bot API credentials.
- **Edge Cases / Potential Failure Types:** API rate-limiting, expired chat identifiers, or instances where the customer has not initiated the bot conversation first (Telegram restrictions prevent bots from initiating blind outbound chats).
- **Sub-Workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Trigger (New Message) | `n8n-nodes-base.telegramTrigger` | Listens for incoming chat messages. | None | Set Business Hours Config | Auto-reply to customers after business hours on Telegram... (See description) / ### 1. Telegram Trigger<br>Listens for every new message sent to your Telegram bot. Add your bot token in the node credentials. |
| Set Business Hours Config | `n8n-nodes-base.set` | Assigns business hours and reply text configuration. | Telegram Trigger (New Message) | Check Business Hours | Auto-reply to customers after business hours on Telegram... (See description) / ### 2. Set Business Hours Config<br>All settings in one place: opening hour, closing hour, working days (1 = Monday ... 7 = Sunday) and the auto-reply text. Edit these - no code needed. |
| Check Business Hours | `n8n-nodes-base.code` | Compares current server time against schedule rules. | Set Business Hours Config | IF After Hours | Auto-reply to customers after business hours on Telegram... (See description) / ### 3. Check Business Hours<br>Compares the current time (n8n server timezone) with your schedule and flags each message as inside or outside business hours. |
| IF After Hours | `n8n-nodes-base.if` | Filters messages to process only after-hours traffic. | Check Business Hours | Send Auto-Reply | Auto-reply to customers after business hours on Telegram... (See description) / ### 4. IF After Hours<br>Routes the flow: messages inside business hours stop here (no reply); after-hours messages continue to the auto-reply. |
| Send Auto-Reply | `n8n-nodes-base.telegram` | Dispatches the configured response back to the customer. | IF After Hours | None | Auto-reply to customers after business hours on Telegram... (See description) / ### 5. Send Auto-Reply<br>Sends your custom reply text to the customer through your Telegram bot. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance and set the execution order configuration to version `v1`.
2. **Add Node 1: Telegram Trigger**
   - Type: `Telegram Trigger` (`n8n-nodes-base.telegramTrigger`)
   - Name: `Telegram Trigger (New Message)`
   - Parameters: Set **Updates** to `message`.
   - Credentials: Configure a new Telegram API credential by entering your Telegram Bot token obtained via BotFather.
3. **Add Node 2: Set**
   - Type: `Set` (`n8n-nodes-base.set`)
   - Name: `Set Business Hours Config`
   - Parameters:
     - Set **Mode** to `manual`.
     - Add the following assignments under fields:
       - Name: `open_hour`, Type: `Number`, Value: `9`
       - Name: `close_hour`, Type: `Number`, Value: `18`
       - Name: `workday_start`, Type: `Number`, Value: `1`
       - Name: `workday_end`, Type: `Number`, Value: `5`
       - Name: `reply_message`, Type: `String`, Value: `Hi! Thanks for reaching out.\n\nOur business hours are Monday to Friday, 9 AM to 6 PM.\nWe'll get back to you first thing in the morning.\n\nIf it's urgent, tell us what's going on and we'll take a look right away.`
     - Ensure **Include Other Fields** is enabled.
   - Connect `Telegram Trigger (New Message)` output to `Set Business Hours Config` input.
4. **Add Node 3: Code**
   - Type: `Code` (`n8n-nodes-base.code`)
   - Name: `Check Business Hours`
   - Parameters:
     - Set **Mode** to `Run Once for All Items`.
     - Paste the following JavaScript implementation:
       ```javascript
       const items = $input.all();
       return items.map(item => {
         const cfg = item.json;
         const openHour = Number(cfg.open_hour ?? 9);
         const closeHour = Number(cfg.close_hour ?? 18);
         const workdayStart = Number(cfg.workday_start ?? 1); // 1 = Monday
         const workdayEnd = Number(cfg.workday_end ?? 5);     // 5 = Friday
         const hour = $now.hour;       // n8n instance timezone
         const weekday = $now.weekday; // 1 = Monday .. 7 = Sunday
         const isBusinessHours =
           hour >= openHour && hour < closeHour &&
           weekday >= workdayStart && weekday <= workdayEnd;
         const msg = cfg.message;
         return { json: {
           text: msg?.text ?? '',
           chat_id: msg?.chat?.id ?? '',
           reply_message: cfg.reply_message ?? '',
           is_business_hours: isBusinessHours,
         }};
       });
       ```
   - Connect `Set Business Hours Config` output to `Check Business Hours` input.
5. **Add Node 4: If**
   - Type: `If` (`n8n-nodes-base.if`)
   - Name: `IF After Hours`
   - Parameters:
     - Set condition rule **Left Value** to `={{ $json.is_business_hours }}`
     - Set **Operator** to Boolean `false`.
   - Connect `Check Business Hours` output to `IF After Hours` input.
6. **Add Node 5: Telegram**
   - Type: `Telegram` (`n8n-nodes-base.telegram`)
   - Name: `Send Auto-Reply`
   - Parameters:
     - Set **Resource** to `Message`.
     - Set **Operation** to `Send Message`.
     - Set **Chat ID** to `={{ $json.chat_id }}`.
     - Set **Text** to `={{ $json.reply_message }}`.
   - Credentials: Link to the same Telegram API credential configured in step 2.
   - Connect the **true** (top) output of `IF After Hours` to `Send Auto-Reply` input.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Try 2 free automations (this one + appointment reminders) | [Gumroad Link](https://zayco3d.gumroad.com/l/muestra-2-automatizaciones-telegram) |
| Want 11 more automations like this one? Full 12-automation pack | [Gumroad Link](https://zayco3d.gumroad.com/l/pack-telegram-n8n) |