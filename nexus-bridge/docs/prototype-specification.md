# Nexus Bridge: One Merchant, One Identity
## UI/UX & Interactive Prototype Specification

**To the Developer/AI generating this prototype:** 
Please read this entire document. Build a single-page interactive HTML prototype (using Tailwind CSS via CDN and vanilla JavaScript or Alpine.js for state management). The prototype must look like a production-ready internal Fintech Operations tool and allow a judge to click through a specific hackathon demo script.

---

## 1. Context & Business Logic
*   **The Problem:** Legacy merchant identity fragmentation forces Support teams to cross-reference 4+ siloed systems (Passport, MXM, CPX, ACH) to answer basic queries. 
*   **The Solution (Nexus Bridge):** A connecting identity layer that links a Shared Entity System (SES) to transactions. It uses "Identity Stamping" (injecting `canonical_id` into payloads) and deterministic resolution.
*   **Strict Rule:** Money-movement identity demands deterministic matching. The system NEVER auto-merges ambiguous records. It routes them to an "UNRESOLVED" manual review queue.

---

## 2. Design System (Fintech Enterprise Style)
Maintain a contained, highly professional, and trustworthy design system.
*   **Color Palette:**
    *   Background: `#F8FAFC` (Slate 50)
    *   Surface/Cards: `#FFFFFF` (White)
    *   Primary Accent: `#0F172A` (Slate 900) - for headers, main text.
    *   Brand/Action: `#2563EB` (Blue 600) - primary buttons, active states.
    *   Success/Matched: `#10B981` (Emerald 500) - tags, badges.
    *   Warning/Unresolved: `#F59E0B` (Amber 500) - manual review queue.
    *   Danger: `#EF4444` (Red 500).
    *   Borders: `#E2E8F0` (Slate 200).
*   **Typography:** Inter or system sans-serif (clean, legible data tables).
*   **Layout:** Left sidebar navigation, top header (with global search), and main content area.
*   **Components:** 
    *   Use pill-shaped status badges.
    *   Use monospaced fonts for IDs (e.g., `SES-993-8X`) and JSON payloads.
    *   Use subtle shadows for cards (`shadow-sm` or `shadow-md`).

---

## 3. Screen Structure & Navigation

The prototype operates as a Single Page Application (SPA) with three main views navigated via a left sidebar:
1.  **Unified Search & 360° View** (Default View)
2.  **Live Stamping Simulator** (API Simulation)
3.  **Exception Queue** (Manual Resolution)

---

## 4. Screen-by-Screen Detailed Specifications

### Screen 1: Unified Search & Entity 360° View
**Purpose:** Demonstrate deterministic resolution. Given any legacy ID, find the unified `canonical_id`.

*   **UI Layout:**
    *   **Top Bar:** Large, prominent search bar with placeholder: *"Search by MXM, CPX, Passport ID, or Business Name..."*
    *   **Main Area (Pre-Search State):** Empty state with a brief system benchmark dashboard (e.g., "50k Transactions Evaluated, 12ms Avg Query Latency, 94% Auto-Resolution").
    *   **Main Area (Post-Search State):** The Entity 360 Dashboard.
*   **Interactive Flow & Clicks:**
    1.  **Action:** User types `MXM-77329` in the search bar and presses Enter/Search.
    2.  **Next:** Reveal the "Entity 360 Dashboard".
    3.  **Content Displayed:**
        *   **Header:** "Acme Retail LLC" | Badge: `Canonical ID: SES-8842-ACME` (Green/Success).
        *   **Card 1 (Identity Graph):** A table or tree showing linked IDs mapped to this SES ID: `MXM-77329`, `CPX-0991`, `Passport-A1B2`.
        *   **Card 2 (Financials):** Aggregated balances across sub-accounts.
        *   **Card 3 (Recent Activity):** A unified ledger of recent transactions pulled from all legacy systems.

### Screen 2: Live Identity Stamping Simulator
**Purpose:** Show how transaction payloads inject the `canonical_id` without breaking legacy downstream workflows.

*   **UI Layout:**
    *   Split-screen view (Left: Incoming Request, Right: Processed Output).
    *   **Left Column:** A code block showing a raw JSON legacy transaction payload (contains `source_mxm_id: "MXM-77329"`, but NO canonical ID).
    *   **Right Column:** Empty code block waiting for output.
    *   **Center/Bottom Action:** A large button labeled "Simulate Payload Injection".
*   **Interactive Flow & Clicks:**
    1.  **Action:** User clicks "Simulate Payload Injection".
    2.  **Next:** A brief loading spinner (500ms for effect), then the Right Column populates.
    3.  **Content Displayed:**
        *   The Right column shows the processed JSON. 
        *   *Crucial interaction:* Visually highlight (e.g., yellow background flash or bold green text) the newly injected fields in the output JSON:
            `"source_canonical_id": "SES-8842-ACME"`
            `"destination_canonical_id": "SES-9001-BETA"`
        *   Display a success toast: "Identity stamped deterministically in 14ms."

### Screen 3: Exception Queue (Safe Handling)
**Purpose:** Prove the AI does not auto-merge conflicting records and handles exceptions safely.

*   **UI Layout:**
    *   A data table titled "Unresolved Identity Mappings - Action Required".
    *   Table columns: Received Date, Legacy ID, Entity Name, Conflict Reason, AI Confidence, Action.
    *   **Data Row 1:** Entity Name: "Acme Retail L.L.C." | Conflict: *Mismatched Tax ID vs SES-8842-ACME* | AI Confidence: *82% (Below 95% Threshold)* | Badge: `UNRESOLVED` (Warning/Orange).
*   **Interactive Flow & Clicks:**
    1.  **Action:** User clicks a "Review" button on Data Row 1.
    2.  **Next:** Opens a Modal/Slide-over pane.
    3.  **Modal Content:**
        *   **Side-by-side comparison:** System Record (SES-8842-ACME) vs Incoming Record (MXM-99110).
        *   **AI Reasoning Block:** "AI Alert: Business names match (fuzzy logic), but Incoming Tax ID ends in 4421, while System Tax ID ends in 4422. Auto-merge prevented."
        *   **Action Buttons:** "Merge Entities" (Primary), "Create New Independent Entity" (Secondary).
    4.  **Action:** User clicks "Create New Independent Entity".
    5.  **Next:** Modal closes, row disappears from the queue, toast notification appears: "New SES Canonical ID generated for safe isolation."

---

## 5. Technical Requirements for (The Developer)
*   Build this as a standalone `index.html` file.
*   Embed all CSS (use Tailwind via CDN: `<script src="https://cdn.tailwindcss.com"></script>`).
*   Embed all JavaScript logic in a `<script>` tag at the bottom.
*   Use inline dummy JSON data to power the search and tables.
*   Implement state toggling so clicking the left sidebar buttons hides/shows the relevant sections (Screen 1, 2, or 3).
*   Make sure the UI looks like a premium, enterprise B2B application.