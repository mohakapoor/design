# 📐 Backend Architecture: `geminiService.ts` Deep Dive

This document maps the exact, active implementation of **`services/geminiService.ts`** in `ux_requirement_studio`. It reflects the real code, functions, fallback pipelines, and invariants running in the application.

---

## 1. End-to-End Service Architecture

```mermaid
sequenceDiagram
    autonumber
    participant UI as WorkshopAssistant (RequirementDraftViews.tsx)
    participant GS as geminiService.ts (refineRequirementDraft)
    participant CFG as Config Service (config.ts / componentSpecServices)
    participant Aura as Aura AI Proxy (/api-proxy/aura/v1)
    participant LLM as Active Model (e.g. gemini-flash-latest)

    UI->>GS: refineRequirementDraft(draft, userInput, history, selectedType)

    rect rgb(240, 245, 255)
        note over GS: Step 1: Client & Config Resolution
        GS->>GS: getAuraBaseUrl(): Check localhost vs .goog vs origin
        GS->>GS: getAuraHeaders(): Add X-Component-Id (50), AppName, APP_CODE, X-Studio-Cookie
        GS->>GS: getAuraClient(): Initialize OpenAI client (baseURL: /api-proxy/aura/v1)
        GS->>CFG: getAuraModel(): GET /config/model (caches active model)
        GS->>CFG: getFeatureRequestPromptTitle(): Resolve prompt title (e.g. USER_FEATURE_WRITER_HELPER)
    end

    rect rgb(245, 250, 255)
        note over GS: Step 2: History Slicing & State Preparation
        GS->>GS: Calculate clarificationQuestionsAsked from history (checks state, phrases, & trailing '?')
        GS->>GS: history.slice(-6): Keep only last 6 messages
        GS->>GS: safeType = selectedType === 'Defect' ? 'Defect' : 'Feature'
        GS->>GS: Construct messages: system (title), history (toOpenAiRole), user payload
    end

    GS->>Aura: openai.chat.completions.create(messages, { query: { k: '0' } })
    Aura->>LLM: Forward to model with managed system instructions
    LLM-->>Aura: Completion text
    Aura-->>GS: choices[0].message.content

    rect rgb(245, 255, 240)
        note over GS: Step 3: Parse, Fallbacks & Invariants
        GS->>GS: parseJsonResponse(text, fallbackReplyKey=true)
        GS->>GS: 1. Strip markdown fences (```json ... ```)
        GS->>GS: 2. JSON.parse()
        GS->>GS: 3. Greedy bracket extraction { ... } / [ ... ]
        GS->>GS: 4. Fallback wrapping: { reply: text }
        GS->>GS: Apply Draft Recovery Fallback (detects markdown user story in reply)
        GS->>GS: Enforce State Invariant (cleanDraft != null => REQUIREMENT_READY)
        GS->>GS: Normalize requirements array or parseMultiRequirementText(cleanDraft)
    end

    GS-->>UI: Return { state, reply, suggestedDraft, requirements }
```

---

## 2. Actual State Machine in `refineRequirementDraft()`

This flowchart depicts the real branching and guard invariants in `geminiService.ts` (lines 474–615):

```mermaid
flowchart TD
    Start([User sends chat message]) --> CountHistory["Calculate clarificationQuestionsAsked<br/>Counts model messages with:<br/>- state == 'CLARIFICATION_REQUIRED'<br/>- contains 'clarification needed'<br/>- ends with '?' (excluding acceptance criteria)"]
    
    CountHistory --> SliceHistory["Take recentHistory = history.slice(-6)"]
    SliceHistory --> SendPrompt["Dispatch Prompt Payload to Aura Proxy with:<br/>CURRENT DRAFT<br/>SELECTED TYPE<br/>CLARIFICATION_QUESTIONS_ASKED<br/>USER INPUT"]
    
    SendPrompt --> ParseResponse["parseJsonResponse(text, fallbackReplyKey=true)"]
    
    ParseResponse --> CheckDraft{"Is suggestedDraft valid string?<br/>(not empty & not 'null')"}

    CheckDraft -- YES --> CleanDraftSet["cleanDraft = suggestedDraft.trim()"]
    
    CheckDraft -- NO --> ReplyFallback{"Does cleanReply contain complete draft structure?<br/>(User Story + Acceptance Criteria)<br/>AND is not a question"}
    
    ReplyFallback -- YES --> AdoptReply["cleanDraft = cleanReply.trim()<br/>cleanReply = 'Requirement draft ready.'<br/>finalState = 'REQUIREMENT_READY'"]
    
    ReplyFallback -- NO --> CleanDraftNull["cleanDraft = null"]

    CleanDraftSet --> ResolvedReady["resolvedState = 'REQUIREMENT_READY'"]
    AdoptReady --> ResolvedReady

    CleanDraftNull --> CheckFinalState{"finalState == 'CLARIFICATION_REQUIRED'?"}
    CheckFinalState -- YES --> ResolvedClarify["resolvedState = 'CLARIFICATION_REQUIRED'"]
    CheckFinalState -- NO --> ResolvedMsgOnly["resolvedState = 'MESSAGE_ONLY'"]

    ResolvedReady --> ExtractReqs{"Are parsed.requirements present?"}
    ExtractReqs -- YES --> MapReqs["Map to Requirement objects (status: 'New', safeType)"]
    ExtractReqs -- NO --> MultiCheck["parseMultiRequirementText(cleanDraft, safeType)"]
    MultiCheck -- Multiple Reqs --> SplitReqs["normalizedReqs = split requirements"]
    MultiCheck -- Single Req --> SingleReq["normalizedReqs = null (single draft)"]

    ResolvedClarify --> ForceNullReqs["normalizedReqs = null (Strict Invariant)"]
    ResolvedMsgOnly --> ForceNullReqs

    MapReqs --> ReturnPayload([Return { state, reply, suggestedDraft, requirements }])
    SplitReqs --> ReturnPayload
    SingleReq --> ReturnPayload
    ForceNullReqs --> ReturnPayload

    classDef ready fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef clarify fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    classDef msg fill:#e2e3e5,stroke:#6c757d,stroke-width:2px;

    class ResolvedReady,AdoptReply,CleanDraftSet,MapReqs ready;
    class ResolvedClarify,CheckFinalState clarify;
    class ResolvedMsgOnly,ForceNullReqs msg;
```

---

## 3. Resilient Parsing Pipeline (`parseJsonResponse`)

In `geminiService.ts` (lines 93–139), every AI response goes through a 4-tier parsing filter:

```mermaid
flowchart TD
    Input[Raw LLM Text from Choices] --> EmptyCheck{Is text null, undefined, or empty?}
    EmptyCheck -- YES --> FallbackReplyCheck{fallbackReplyKey == true?}
    FallbackReplyCheck -- YES --> EmptyReply["Return { reply: 'I'm sorry, I couldn't generate a response...' }"]
    FallbackReplyCheck -- NO --> EmptyArray["Return []"]

    EmptyCheck -- NO --> Tier1["Tier 1: Strip Code Fences<br/>Regex match ```(?:json)?\s*([\s\S]*?)```"]
    Tier1 --> Tier2{"Tier 2: Direct JSON.parse()"}
    Tier2 -- Success --> ReturnParsed([Return parsed object])
    
    Tier2 -- Fails --> Tier3{"Tier 3: Greedy Substring Extraction<br/>Find min/max of '{' and '['<br/>Find matching last '}' or ']'"}
    Tier3 -- Found Valid Substring --> ParseSub["JSON.parse(innerSubstring)"]
    ParseSub -- Success --> ReturnParsed
    
    ParseSub -- Fails / Not Found --> Tier4{fallbackReplyKey == true?}
    Tier4 -- YES --> WrapPlain["Tier 4: Wrap Plain Text<br/>Return { reply: text }"]
    Tier4 -- NO --> ThrowError([Throw Error: 'AI response is not valid JSON'])

    classDef success fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef fail fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    class ReturnParsed,WrapPlain success;
    class ThrowError fail;
```

---

## 4. Multi-Requirement Splitting (`parseMultiRequirementText`)

In `geminiService.ts` (lines 632–697), when the AI returns multiple requirements in a single draft:

```mermaid
flowchart TD
    TextIn[Markdown Draft Text] --> DividerCheck{"Contains '---' markdown dividers?"}
    
    DividerCheck -- YES --> SplitDivider["Split by /(?:^|\n)\s*---\s*(?:\n|$)/"]
    SplitDivider --> SectionsExist{Sections count > 1?}
    SectionsExist -- YES --> ProcessSections["For each section:<br/>1. Extract title from line 1 (strip markdown '#')<br/>2. resolveExplicitType(sectionText)<br/>3. status: 'New', priority: 'Medium', category: 'User'"]
    
    DividerCheck -- NO --> PatternCheck{"Matches /(?:^|\n)(?:###?\s*)?Requirement\s+\d+[:\s]/gi?"}
    SectionsExist -- NO --> PatternCheck

    PatternCheck -- YES (Count > 1) --> MatchSplit["Slice text between match start and next match start"]
    MatchSplit --> ProcessPattern["For each section:<br/>1. Clean title: strip 'Requirement N:'<br/>2. resolveExplicitType(sectionText)<br/>3. Return Requirement object"]

    PatternCheck -- NO (<= 1) --> ReturnEmpty["Return [] (Treated as single requirement)"]

    ProcessSections --> ReturnList([Return Requirement[] array])
    ProcessPattern --> ReturnList
    ReturnEmpty --> ReturnList
```

---

## 5. Complete Function Map of `geminiService.ts`

| Exported Function | Lines in Code | Prompt Key / Mechanism | Purpose |
|---|---|---|---|
| `refineRequirementDraft` | 452–619 | Dynamic via `getFeatureRequestPromptTitle()` | Interactive chat-based requirement authoring with 3-question limit and draft recovery. |
| `getRequirementBreakdown` | 199–231 | `FEATURE_PLANNER` | Decomposes a parent requirement into child sub-requirements. |
| `getComponentSpecBreakdown` | 238–296 | `getComponentSpecPromptTitle()` | Decomposes a requirement and maps each specification to an available `ProjectComponent.id`. |
| `getRequirementsResponse` | 299–344 | `REQUIREMENTS_PRODUCT_OWNER` | Full-list requirements assistant supporting cancellation via `AbortSignal`. |
| `getTaskResponse` | 346–391 | `REQUIREMENTS_PRODUCT_OWNER` | Task-level assistant supporting cancellation via `AbortSignal`. |
| `parseRequirementsFromText` | 393–423 | `REQUIREMENTS_FROM_TEXT` | Ingests unstructured meeting notes or PRD text and extracts requirement objects. |
| `generatePRD` | 425–450 | `PRD_FROM_REQUIREMENTS_GENERATOR` | Converts a structured JSON array of requirements into a complete PRD document. |
| `breakdownDraftToRequirements` | 699–737 | `getAiBreakdownPromptTitle()` | Parses an editor draft into normalized `Requirement[]` objects. |
| `parseMultiRequirementText` | 632–697 | Local regex parser (no LLM call) | Splits drafts containing `---` or `Requirement N:` into discrete requirements. |
| `fetchPromptFromManager` | 190–197 | `getPromptByTitle(title)` | Helper to fetch prompt text directly from Prompt Manager API. |
