# 🛠️ Engineering Log: Optimization & Debugging Sprint

This document tracks the technical challenges, investigations, and resolutions encountered during the final development phase of the **FilmClub** Salesforce App.

---

## 1. Data Integrity: The Decimal Truncation Fix

**The Problem:** Movies with high-precision ratings were being rounded down in the UI (e.g., *The Godfather* was displaying as **9.0** instead of its actual IMDb score of **9.2**).

**The Investigation:**
- **Field Definition:** Verified that the `IMDb_Rating__c` custom field was correctly configured with `Length: 2` and `Decimal Places: 1`.
- **Flow Logic:** Discovered that the **Flow Variable** used to capture the API response was set to a "Number" data type with `0` decimal places, causing the system to truncate the decimal before the record was created.

**The Fix:**
- Updated the Flow variable and the associated mapping logic to support **1 Decimal Place**.
- **Result:** The system now successfully captures and stores movie ratings with 100% accuracy.

**Key Lesson:** Data is only as precise as its narrowest "hop." Always verify decimal settings at every stage of the data pipeline—from the API response to the Flow variables and finally the database.

---

## 2. Logic Refinement: The "Strict" Linear Star Scale

**The Problem:**
The initial star rating formula used uneven thresholds, leading to a visual representation that felt inconsistent with the 10-point IMDb scale.

**The Fix:**
Refactored the Star Rating Formula Field to a **Linear 10-point scale**. This ensures that every 1.0 point change in the IMDb score corresponds exactly to a half-star change in the UI.

**Final Logic Implementation:**
```sql
/* Linear 10-point Scale Mapping 
   Ensures consistent visual weighting across the library.
*/
IF(IMDb_Rating__c >= 9.5, "⭐⭐⭐⭐⭐",
IF(IMDb_Rating__c >= 8.5, "⭐⭐⭐⭐✫",
IF(IMDb_Rating__c >= 7.5, "⭐⭐⭐⭐",
IF(IMDb_Rating__c >= 6.5, "⭐⭐⭐✫",
IF(IMDb_Rating__c >= 5.5, "⭐⭐⭐",
IF(IMDb_Rating__c >= 4.5, "⭐⭐✫",
IF(IMDb_Rating__c >= 3.5, "⭐⭐",
IF(IMDb_Rating__c >= 2.5, "⭐✫",
IF(IMDb_Rating__c >= 1.5, "⭐",
IF(IMDb_Rating__c > 0, "✫",
"No Rating"))))))))))
```

**Result:** The Godfather (9.2) now correctly renders as 4.5 Stars (⭐⭐⭐⭐✫), setting a high bar where only 9.5+ scores earn the "Elite" 5th star.

---

## 3. Metadata Management: The Dependency Lock

**The Problem:** Attempted to convert the `Language__c` field from a __Restricted Picklist__ to a __Text__ field to accommodate multi-language strings returned by the OMDb API (e.g., "English, German, French").

**The Roadblock:** Salesforce blocked the change with a `RESTRICTED_PICKLIST` error because the field was referenced by multiple versions of an active Flow.

**Architectural Decision:** To maintain the stability of the core "Search & Fetch" automation, I chose to maintain the current field structure rather than risk a full metadata "nuke" and rebuild. This ensures the app remains 100% functional while highlighting a future roadmap item for metadata refactoring.

**Key Lesson:** Understanding metadata dependencies is crucial for platform stability. Knowing when to pivot vs. when to rebuild is a core skill for any Salesforce Developer.
