# Developer guide on setting up Zotero links in notion

This document describes how to set up the “Notion-friendly Zotero links” system for the lab:

**Notion `https://...` link → browser opens a tiny GitHub Pages page → redirects to `zotero://...` → Zotero Desktop opens the PDF/highlight.**

## 0) What you are building

You will maintain **two pieces**:

1. **Bridge website** (GitHub Pages)
- A static `index.html` hosted at a stable lab-owned URL (recommended).
- It reads a `#zotero://...` fragment and redirects to Zotero Desktop.
1. **Zotero “Copy Notion Link” action**
- A Zotero Actions & Tags action that generates the correct link format for Notion.
- It creates either:
    - annotation link (highlight) → `[highlight](https://bridge/#zotero://open-pdf/...&annotation=...)`
    - non-annotation link → `(https://bridge/#zotero://select/... or open-pdf...)`

(Your current “Obsidian-friendly” action is based on the standard Actions & Tags copy-link logic, including the `open-pdf` vs `select` logic and annotation handling. )

---

## 1) Prerequisites (lab-wide)

- A **lab GitHub organization** (recommended) to host the bridge page.
- A **Zotero Group Library** for shared papers.
- Recommended: enable **Zotero group attachment file syncing** so teammates can open PDFs and annotation links reliably.

---

## 2) Create the bridge repo in the lab GitHub organization

### 2.1 Create a repository

In the lab GitHub organization:

1. Create a new repo, e.g. `zotero-bridge`
2. Make it **Public** (simplest for GitHub Pages)
3. Create the repo

### 2.2 Add `index.html` (secure minimal redirect)

Create `index.html` in repo root with these two security mitigations:

- Only redirect if the target starts with **`zotero://`** (blocks redirecting to `https://`, `file://`, other app schemes)
- Add **no-referrer** policy to reduce referrer leakage
- The working scripts is as follows
    
    ```jsx
    <!doctype html>
    <html>
      <head>
        <meta charset="utf-8" />
        <title>Zotero Bridge</title>
    
        <!-- Mitigate referrer leakage -->
        <meta name="referrer" content="no-referrer" />
    
        <script>
          function safeZoteroUri(raw) {
            if (!raw) return null;
            const s = raw.trim();
    
            // Only allow zotero:// links
            if (!s.startsWith("zotero://")) return null;
    
            // Optional hardening: block control characters
            if (/[\u0000-\u001F\u007F]/.test(s)) return null;
    
            return s;
          }
    
          window.onload = () => {
            // Preferred: fragment form
            // https://YOURNAME.github.io/zotero-bridge/#zotero://open-pdf/...
            let raw = window.location.hash ? window.location.hash.slice(1) : "";
    
            const msg = document.getElementById("msg");
            const a = document.getElementById("fallback");
    
            const zoteroUri = safeZoteroUri(raw);
            if (!zoteroUri) {
              msg.textContent =
                "Invalid or missing link. Use: https://.../#zotero://open-pdf/...";
              return;
            }
    
            msg.textContent = "Opening Zotero…";
            a.href = zoteroUri;
            a.textContent = "Click here if it didn’t open automatically.";
    
            // Redirect to Zotero desktop
            window.location.href = zoteroUri;
          };
        </script>
      </head>
      <body>
        <p id="msg">Loading…</p>
        <p><a id="fallback" href="#"></a></p>
      </body>
    </html>
    ```
    

---

## 3) Enable GitHub Pages for the repo

Repo → **Settings → Pages**:

1. **Source:** Deploy from a branch
2. **Branch:** `main`
3. Save

Your bridge URL will be:

- `https://<ORG>.github.io/zotero-bridge/`

This is the stable URL you should bake into the Zotero “Notion link” action.

---

## 4) User side setup

See:

User guide on setting up Zotero links in notion below

---

## 5) Maintenance tips

- Keep the bridge repo in the **lab org** so the URL doesn’t break when people leave.
- If you ever move the bridge:
    - update `bridgePrefix` in the action script
    - optionally keep the old bridge URL alive with a redirect page during transition
- Put the action script and this doc in the same `zotero-bridge` repo (README or `/docs/`) so onboarding is easy.

# User guide on setting up Zotero links in notion

This setup lets you paste a link into Notion that, when clicked, opens **Zotero Desktop** at the **exact PDF highlight/annotation** (or at the PDF/item).

## Prerequisites

1. **Zotero Desktop installed**
    - Recommended: Zotero 7 (current major version).
2. **Access to the lab Zotero Group Library**
    - You must be added to the lab’s Zotero group, and the shared papers must be in that group library.
3. **(Recommended) Group PDF file syncing enabled**
    - In Zotero: `Zotero → Settings/Preferences → Sync`
    - Ensure **“Sync attachment files in group libraries using Zotero storage”** is enabled.
    - This helps ensure the PDF is available locally so the deep link opens smoothly.
4. **You must use Zotero Desktop to open links**
    - The links ultimately redirect to `zotero://...`, which opens the Zotero desktop app.

---

## Install Zotero “Actions & Tags” add-on

1. Download the **Actions & Tags** add-on (.xpi).
2. In Zotero Desktop, go to:
    - `Tools → Plugins`
    - Click the **gear icon** → **Install Plugin From File…**
    - Select the `.xpi` file
3. Restart Zotero.

---

## Add the lab’s “Copy Notion Link” action + set a hotkey

### Steps

1. In Zotero:
    - `Zotero → Settings → Actions & Tags`
2. Click + to create a **new Action**
3. Name the Action as “CopyNotionLink”
4. Choose “Operation” to be Script
5. Paste the following scripts in “Data”
    - Scripts in here
        
        ```jsx
        // @author windingwind, garth74 + customized for Notion bridge
        // @link https://github.com/windingwind/zotero-actions-tags/discussions/115
        // @usage Select an item in the library (or an annotation) and press the assigned shortcut keys
        
        // =====================
        // EDIT THESE SETTINGS
        // =====================
        
        /** Name of the field to use as link text for non-annotation items (title is fine). */
        let linkTextField = "title";
        
        /**
         * Output type for "text/unicode" clipboard payload.
         * - "md" is best if you want [label](url) for annotations.
         * - "plain" outputs just the URL (but we still add parentheses for non-annotation below).
         */
        let linkType = "md";
        
        /** If true, make the link specific to the currently selected collection. */
        let useColl = false;
        
        /** If true, use Better Notes zotero://note link when the selected item is a note. */
        let useNoteLink = false;
        
        /** Action of link. auto = open-pdf for PDFs/annotations (and regular items with a PDF), select otherwise */
        let linkAction = "auto";
        
        /**
         * Your Notion bridge prefix (GitHub Pages). Must end with "/#"
         * Example: "https://yourname.github.io/zotero-bridge/#"
         */
        const bridgePrefix = "https://endreslabe3.github.io/zotero-bridge/#";
        
        // =====================
        // END OF EDITABLE SETTINGS
        // =====================
        
        // For efficiency, only execute once for all selected items
        if (item) return;
        item = items[0];
        if (!item && !collection) return "[Copy Zotero Notion Link] item is empty";
        
        if (collection) {
          linkAction = "select";
          useColl = true;
        }
        
        // Decide action automatically:
        // - annotations + PDF attachments => open-pdf
        // - regular items => open-pdf if they have a PDF attachment, else select
        if (linkAction === "auto") {
          if (item.isAnnotation() || item.isPDFAttachment()) {
            linkAction = "open-pdf";
          } else if (item.isRegularItem()) {
            const bestPDF = (await item.getBestAttachments()).find((att) => att.isPDFAttachment());
            linkAction = bestPDF ? "open-pdf" : "select";
          } else {
            linkAction = "select";
          }
        }
        
        const uriParts = [];
        let uriParams = "";
        
        let targetItem = item;
        if (linkAction === "open-pdf") {
          uriParts.push("zotero://open-pdf");
          if (item.isRegularItem()) {
            targetItem = (await item.getBestAttachments()).find((att) => att.isPDFAttachment());
          } else if (item.isAnnotation()) {
            targetItem = item.parentItem;
        
            // Open the PDF at the page of the annotation + point to annotation key
            let pageIndex = 1;
            try {
              pageIndex = JSON.parse(item.annotationPosition).pageIndex + 1;
            } catch (e) {
              Zotero.warn(e);
            }
            uriParams = `?page=${pageIndex}&annotation=${item.key}`;
          }
        } else {
          uriParts.push("zotero://select");
          if (item?.isAnnotation()) targetItem = item.parentItem;
        }
        
        if (!targetItem && !collection) return "[Copy Zotero Notion Link] item is invalid";
        
        // Link label text
        let linkText;
        if (collection) {
          linkText = collection.name;
        } else if (item.isAttachment()) {
          // Use top-level item title for attachments
          linkText = Zotero.Items.getTopLevel([item])[0].getField(linkTextField);
        } else if (item.isAnnotation()) {
          // For annotations: ONLY use highlight text/comment as label (no "Full Text PDF(...)")
          linkText = item.annotationText || item.annotationComment || "annotation";
        } else {
          linkText = item.getField(linkTextField);
        }
        
        // Add library/group prefix
        let libraryType = (collection || item).library.libraryType;
        if (libraryType === "user") {
          uriParts.push("library");
        } else {
          uriParts.push(`groups/${Zotero.Libraries.get((collection || item).libraryID).groupID}`);
        }
        
        // Optional: make collection-specific
        if (useColl) {
          let coll = collection || Zotero.getActiveZoteroPane().getSelectedCollection();
          if (!!coll) uriParts.push(`collections/${coll.key}`);
        }
        
        if (!collection) {
          uriParts.push(`items/${targetItem.key}`);
        }
        
        let uri = uriParts.join("/");
        if (uriParams) uri += uriParams;
        
        if (useNoteLink && item?.isNote() && Zotero.BetterNotes) {
          uri = Zotero.BetterNotes.api.convert.note2link(item);
        }
        
        // Build Notion-friendly https URL (fragment form, no encoding needed)
        const notionUri = bridgePrefix + uri;
        
        // Helpers: keep label neat (one line, not huge)
        function compact(s, maxLen = 120) {
          if (!s) return "";
          const oneLine = String(s).replace(/\s+/g, " ").trim();
          return oneLine.length > maxLen ? oneLine.slice(0, maxLen - 1) + "…" : oneLine;
        }
        function mdEscape(s) {
          // minimal escapes for Markdown link label
          return String(s)
            .replace(/\[/g, "\\[")
            .replace(/\]/g, "\\]")
            .replace(/\(/g, "\\(")
            .replace(/\)/g, "\\)");
        }
        
        const clipboard = new Zotero.ActionsTags.api.utils.ClipboardHelper();
        
        const isAnno = item.isAnnotation();
        // const label = mdEscape(compact(linkText));
        const label = mdEscape(String(linkText).replace(/\s+/g, " ").trim());
        
        if (linkType === "md") {
          // Your requested behavior:
          // - annotation => [highlight sentence](https://.../#zotero://...)
          // - else       => (https://.../#zotero://...)
          if (isAnno) {
            clipboard.addText(`[${label}](${notionUri})`, "text/unicode");
          } else {
            clipboard.addText(`(${notionUri})`, "text/unicode");
          }
        } else if (linkType === "plain") {
          // Plain mode: still keep your parentheses preference for non-annotations
          if (isAnno) {
            clipboard.addText(`${compact(linkText)}\n${notionUri}`, "text/unicode");
          } else {
            clipboard.addText(`(${notionUri})`, "text/unicode");
          }
        } else {
          // Fallback
          clipboard.addText(notionUri, "text/unicode");
        }
        
        // Also provide HTML flavor (some apps prefer it)
        // clipboard.addText(`<a href="${notionUri}">${compact(linkText)}</a>`, "text/html");
        clipboard.addText(`<a href="${notionUri}">${String(linkText).replace(/\s+/g, " ").trim()}</a>`, "text/html");
        
        clipboard.copy();
        return `[Copy Zotero Notion Link] link ${notionUri} copied.`;
        ```
        
6. Assign a shortcut hotkey
    
    ### Recommended hotkey
    
    Use: **⌘ + ⌥ + ~**
    
    (You can use any shortcut you like, as long as it does not conflict with Zotero’s built-in shortcuts. If a shortcut accidentally creates a new empty item or triggers another Zotero action, pick a different combo.)
    
7. Save the action, then restart Zotero once (recommended).

---

## How to use it (daily workflow)

### A) Link to a specific highlight/annotation

1. Open the PDF in Zotero.
2. Highlight the sentence/paragraph (creates an annotation).
3. Click the highlight sentence/paragraph
4. Press your **Copy Notion Link** hotkey.
5. Paste into Notion.

What you’ll paste looks like a normal `https://...` link, but it will redirect to Zotero and jump to that highlight.

### B) Link to the PDF/item (not a specific highlight)

1. Click the paper item in Zotero (or its PDF attachment).
2. Press the **Copy Notion Link** hotkey.
3. Paste into Notion.

This will open Zotero at the PDF (or at least select the item), depending on whether a PDF is attached.

---

## Notes / Troubleshooting

- **A browser tab opens first:** That’s expected. Notion can’t directly open `zotero://...`, so we use an `https://` bridge page that redirects to Zotero.
- **Link opens but Zotero can’t find the PDF:** Wait for Zotero sync to finish, or open the item once to trigger PDF download (if syncing is enabled).
- **Nothing gets copied / weird behavior happens:** Your hotkey is likely conflicting with a Zotero shortcut. Change the hotkey to a different combination (e.g., `⌘⌥⇧Z`).
