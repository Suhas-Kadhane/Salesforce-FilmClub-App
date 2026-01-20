# 🎬 Project FilmClub: Agentforce & OMDb API Integration Post-Mortem

## 📌 Project Context
This project aims to build an autonomous AI Agent (Agentforce Agent) within a **Salesforce Developer Org** that acts as a Movie Librarian. The agent is designed to take a movie title from a user via chat, fetch metadata (Director, Year, Plot) from the **OMDb API**, and create a record in a custom `Movie__c` object.

__Core Tech Stack:__

- Salesforce Agentforce (Service Agent)
- Salesforce Flow Builder (Autolaunched Flows)
- External Services (OMDb API Integration)
- Apex (JSON Parsing / Optional)

---

## 🏗️ Technical Architecture
* **AI Layer:** Salesforce Agentforce (Einstein Copilot)
* **Logic Layer:** Autolaunched Flow (`Agent_Action_Add_Movie`)
* **Integration Layer:** Named Credentials & External Services (Open Movie Database API)
* **Data Layer:** Custom Salesforce Object `Movie__c`

---

## 🔍 Specific AI Agent Error Log (Technical Breakdown)

| **Issue/Category** | **Specific Error/Response** | **What was happening under the hood** |
| :--- | :--- | :--- |
| **The "Flinch"** | Agent starts to help, then says: *"I’m here to help with support related to our services."* | **Topic Confusion:** The Agent identified the "Movie" topic but hit a "Confidence Threshold" error. It defaulted to the System Topic to avoid making a mistake. |
| **The Clarification Loop** | *"Could you clarify what you'd like me to do with the movie 'Inception'?"* | **Missing Mapping:** The Agent recognized the title but didn't feel "authorized" to pass that string directly into the Flow without a second confirmation. |
| **The Refusal** | *"It seems I can't directly add movies to FilmClub for you."* | **Capability Gap:** Triggered when the Action was visible but the "Allow AI to run Flows" permission or the "System Context" wasn't fully synced. |
| **The Zero Action Count** | `Actions - 0 (Zero)` in the Reasoning Tab. | **Handshake Failure:** The Topic was selected, but the "Brain" could not find the "Hands." The connection between the Topic and the Flow Action was broken. |
| **The Loopback Error** | Agent repeats your instruction back to you instead of running it. | **Instruction Paradox:** The instructions were interpreted as "Conversation" rather than "Logic." The Agent thought it should *talk* about the action instead of *executing* it. |
| **The Ghost Agent** | Agent reappears in the list after deletion. | **Metadata Persistence:** Salesforce was still holding the Agent's configuration in the cache, preventing a "clean" re-install of the logic. |

---

## ⚠️ The Battle Log: Challenges & Errors

During the 24-hour integration window, we encountered several high-level "handshake" failures between the AI and the Salesforce Metadata.

### 1. The "Classification" Failures
* **Symptom:** Agent ignores the movie request or says "I can't assist with that."
* **Root Cause:** The Agent failed to map the user's intent to the specific **Topic**. 
* **Fixes Attempted:** Rewrote Topic Instructions using "Robotic Commands" and strict role definitions to drown out the System's default Customer Service topic.

### 2. The "Clarification Loop" (The Flinch)
* **Symptom:** The Agent asks, *"Would you like me to add this movie or something else?"* then immediately crashes into a default support message.
* **Technical Insight:** This is a **Confidence Threshold** issue. The AI identifies the action but "flinches" because it isn't 100% sure it's allowed to pass data to the Flow.
* **Fixes Attempted:** * Unchecked "Ask user for value" in Action settings.
    * Modified Input Instructions to explicitly command the AI to "assume the title and run."

### 3. The "Actions - 0" Debugging Wall
* **Symptom:** Reasoning Tab shows the correct Topic is selected, but `Actions - 0` are triggered.
* **Root Cause:** Metadata Sync Lag. In Dev Orgs, the Agent's "Brain" often loses the connection to the "Body" (the Flow) after the Flow is updated.
* **Fixes Attempted:** The **"Handshake Reset"**—deleting and re-adding the Action to the Topic to force a metadata refresh.

### 4. Permission & Context Blockers
* **Symptom:** Agent claims it "doesn't have permission" to add movies.
* **Fixes Attempted:** * Enabled **"Flow User"** on the System Admin profile.
    * Updated Flow settings to **"System Context Without Sharing"** to ensure the Agent could write to the database regardless of the running user's restrictions.

---

## 📝 Documented Error Codes & Behaviors

| Error/Behavior | Platform Message | Technical Resolution |
| :--- | :--- | :--- |
| **Topic Competition** | "I'm here to help with support related to our services." | Used **Negative Constraints** ("You are FORBIDDEN from...") in instructions. |
| **Handshake Lag** | `Actions - 0` (Zero) | Performed a full Action delete/re-add and Agent deactivation/reactivation. |
| **API Parameter Mismatch** | OMDb 401 Unauthorized | Corrected Named Credential URL structure to properly append `?apikey={!HTMLENCODE($Credential.Password)}`. |
| **UI Ghosting** | Agent reappears after deletion. | Identified as a browser/Salesforce cache issue; required Incognito mode and hard logouts. |

---

## 🛠️ Lessons Learned for Salesforce Engineers

1.  **Instruction Sincerity:** AI Agents in Salesforce respond better to "Command" syntax than "Conversational" syntax when triggering Flows.
2.  **The 1% Rule:** If an input variable is even slightly ambiguous, the Agent will default to a "clarification question" which often breaks the logic flow in early-stage Dev Orgs.
3.  **Metadata is King:** Always ensure the Flow is not just "Saved" but **Activated** and marked **"Available for Output"** for any response variables.
4.  **Patience with Dev Orgs:** Background processes for Agentforce metadata can take 5-10 minutes to sync. Rapid testing often leads to "false negative" results.

---

## 📅 Final Status
The **Flow Logic** and **API Integration** are 100% verified. The **Agent Connection** is currently being rebuilt from scratch (`FilmClub_v2`) to eliminate "metadata debt" and instruction clutter accumulated during initial debugging.
