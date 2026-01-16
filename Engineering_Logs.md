# 📜 Engineering Log: Troubleshooting & Error Resolution

This section documents the "Battle Scars", the errors encountered during development and how they were solved.

---

## Error 1: "Callout Failed: Unauthorized"

- **Symptoms:** Flow crashed immediately during the API call step.  
- **Root Cause:** The Principal access was not assigned to the user profile/permission set.  
- **Resolution:** Created a Permission Set `OMDb_API_Access`, added the `OMDb_External - OMDb_Principal` to the External Credential Principal Access section, and assigned the set to the user.

---

## Error 2: "Invalid API Key (401 Error)"

- **Symptoms:** The flow ran, but the OMDb response returned `{"Response":"False","Error":"Invalid API key!"}`.  
- **Root Cause:** The API key was generated but not activated via the OMDb confirmation email.  
- **Resolution:** Located the activation email, clicked the verification link, and verified the key using a direct browser URL test:  
  `http://www.omdbapi.com/?t=Tenet&apikey=XXXXX`

---

## Error 3: "Input Variables Not Supported for Debugging"

- **Symptoms:** When debugging the flow, a message stated:  
  `"This automation has no variables that allow input access."`  
- **Root Cause:** The Screen Component variable was not explicitly marked as "Available for Input."  
- **Resolution:** Identified that Screen Flows can be debugged by simply clicking **Run** instead of trying to pass variables manually in the debug setup, or by checking the "Available for Input" box in the Variable Manager.

---

## Error 4: The "Duplicate Spawner"

- **Symptoms:** Searching for the same movie multiple times created multiple identical records.  
- **Root Cause:** Missing "Pre-Check" logic in the flow.  
- **Resolution:** Added a Get Records element at the start of the flow to search for existing titles. Implemented a Decision Element to branch:  
  - **Outcome – Yes:** Skip API call and alert user.  
  - **Outcome – No:** Proceed with API call and record creation.
