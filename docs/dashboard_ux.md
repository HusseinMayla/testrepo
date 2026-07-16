# 📊 Client Dashboard & UX Specifications

This document defines the user interface (UI) principles, component structure, color palette, and layout blueprints for the `chatbotRAG` React administrator portal.

---

## 🎨 Theme & Typography

We prioritize a high-readability, professional, and clutter-free interface that allows non-technical administrators (such as registry clerks or small business support reps) to manage their bots without friction.

* **Primary Font:** `Inter` (sans-serif)
* **Design Aesthetic:** Minimalist flat card layout, high-contrast states, and subtle transitions.
* **Palette Configuration (Variables):**
  ```css
  :root {
    --bg-main: #f8fafc;        /* Slate 50 */
    --bg-card: #ffffff;        /* Pure White */
    --text-primary: #0f172a;   /* Slate 900 */
    --text-secondary: #475569; /* Slate 600 */
    --border-color: #e2e8f0;   /* Slate 200 */
    
    --brand-primary: #4f46e5;  /* Indigo 600 */
    --brand-hover: #4338ca;    /* Indigo 700 */
    --success: #16a34a;        /* Green 600 */
    --warning: #ca8a04;        /* Yellow 600 */
    --danger: #dc2626;         /* Red 600 */
  }
  ```

---

## 🗂️ Core Navigation Layout

The dashboard uses a sidebar-driven layout designed for desktop resolutions, responsive down to tablets:

```text
+-----------------------------------------------------------------------+
|  [chatbotRAG Logo]          Org: "Alpha Corp"     Plan: [ Growth ]    |
+-----------------------------------------------------------------------+
|  [Sidebar]       |  [Main Panel]                                      |
|  * Overview      |  * Metrics Overview (Cards: Total Qs, Tokens, Bot) |
|  * Data Source   |  ------------------------------------------------- |
|  * Bot Settings  |  * Activity Chart (Daily usage trend line)         |
|  * Help & Docs   |  ------------------------------------------------- |
|                  |  * Active Telegram Threads List                    |
+-----------------------------------------------------------------------+
```

---

## 💻 Panel Specifications

### 1. Panel: Overview (Home)
Displays summary cards containing real-time values:
* **Token Usage Card:** Progress bar showing `Current Tokens / Plan Limit` (e.g., `124,500 / 500,000`). Switches color to warning yellow if > 80% and danger red if > 95%.
* **Active Bot Card:** Displays green pulse status indicator `[ ● Connected ]` with the bot's telegram handle (e.g., `@alpha_qabot`).
* **Usage Chart:** Simple line chart using `recharts` plotting daily API queries and total processed tokens over a rolling 14-day window.

### 2. Panel: Knowledge Base Ingestion
Handles client document uploads:
* **File Upload Dropzone:** Drag-and-drop region with state representations:
  * *Default:* "Drag & drop PDF/TXT file here, or click to browse."
  * *DragOver:* Highlighted border and icon swap.
  * *Processing:* Spinners showing file uploading and chunking progress.
* **Document Table:** Lists files showing file size, timestamp, and index status.
  * Status states: `[ Pending ]` (orange), `[ Vectorizing ]` (blue), `[ Active ]` (green), or `[ Failed ]` (red).
* **System Prompt Configurations:** A dedicated `<textarea>` containing the bot's master system instructions. Includes a "Save Changes" button that automatically flushes the configuration to the Supabase database.

### 3. Panel: Bot Integration & Key Management
A simple key validation interface:
* **API Token Input:** An password-obscured input field containing the Telegram bot token. Includes a show/hide toggle.
* **Connection Tester:** A "Test & Save Connection" button which runs the backend webhook validation.
* **Webhook Health Banner:** Displays detailed diagnostics if the webhook receives validation failures from the Telegram server.

### 4. Panel: Account & Billing (Future Proofed)
Allows user to upgrade tiers:
* **Plan Toggle Interface:** Two comparative cards detailing Starter and Growth specifications.
* **Upgrade Button:** Launches checkout session (simulated for MVP; connects to Stripe/Paddle in production).
