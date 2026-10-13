# 📐 AI Requirement Generation: Backend Architecture & Workflow Diagrams

This document illustrates the end-to-end architecture, data flows, state machines, and code paths powering the **AI Requirement Generation** system in `ux_requirement_studio`.

---

## 1. End-to-End System Architecture

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Product Manager / Engineer)
    participant UI as WorkshopAssistant (RequirementDraftViews.tsx)
    participant Service as GeminiService (geminiService.ts)
    participant Config as Config Manager (/api/config)
    participant Aura as Aura AI Proxy (/api-proxy/aura/v1)
    participant Model as LLM (Gemini Model)
    participant Editor as RequirementDraftEditor (Editor State)

    User->>UI: Types request & clicks Send
    UI->>UI: Append user message to chatHistory
    UI->>Service: refineRequirementDraft(draftText, userInput, chatHistory, selectedType)
    
    rect rgb(240, 245, 255)
        note over Service: 1. Compute Context
        Service->>Service: Count CLARIFICATION_QUESTIONS_ASKED from history (0 to 3)
        Service->>Service: Slice last 6 messages to prevent context drift
        Service->>Config: getFeatureRequestPromptTitle() ("USER_FEATURE_WRITER_HELPER")
        Service->>Config: getAuraModel() (resolves active model, e.g. gemini-flash-latest)
        Service->>Service: Construct payload with prompt, draft, checklist rules & JSON schema
    end

    Service->>Aura: POST /chat/completions (OpenAI SDK client)
    Aura->>Model: Forward prompt + system instruction
    Model-->>Aura: Raw response string
    Aura-->>Service: Raw response string

    rect rgb(245, 255, 240)
        note over Service: 2. Parse & Invariant Guard
        Service->>Service: parseJsonResponse(): Strip markdown fences (```json ... ```)
        Service->>Service: Extract { state, reply, suggestedDraft, requirements }
        Service->>Service: Enforce Guard Invariants (suggestedDraft null => requirements strictly null)
    end

    Service-->>UI: Return { state, reply, suggestedDraft, requirements }

    alt state == "CLARIFICATION_REQUIRED"
        UI->>UI: Append AI reply with A/B/C options to chatHistory
        note over Editor: Left editor untouched (draftText preserved)
    else state == "MESSAGE_ONLY"
        UI->>UI: Append AI advisory reply to chatHistory
        note over Editor: Left editor untouched (draftText preserved)
    else state == "REQUIREMENT_READY"
        UI->>UI: Append confirmation reply to chatHistory
        UI->>Editor: setDraftText(suggestedDraft) -> Updates editor markdown
        alt Multiple requirements detected
            UI->>Editor: setBrokenDownReqs() & setIsReviewingBreakdown(true)
        end
    end
```

---

## 2. Clarification State Machine & Decision Tree

This diagram outlines how the model processes user input against the **6-Point Checklist** and the **3-Question Cap**:

```mermaid
flowchart TD
    Start([User sends chat message]) --> ParseInput[Analyze CURRENT DRAFT + USER INPUT + History]
    
    ParseInput --> CheckFunc{Are Points 1 to 5 satisfied?\n- Actor & Access\n- Context\n- Trigger\n- Action & Behavior\n- Functional Outcome}

    CheckFunc -- YES --> ReqReady[State: REQUIREMENT_READY]
    ReqReady --> GenDraft[Generate complete markdown in suggestedDraft]
    GenDraft --> GenJSON[Populate requirements array with discrete items]
    GenJSON --> ShortReply[Set reply to short confirmation: 'Requirement draft ready.']
    GenJSON --> PopulateEditor([Editor populates on left side])

    CheckFunc -- NO --> CountCheck{CLARIFICATION_QUESTIONS_ASKED < 3?}

    CountCheck -- YES --> ClarifyState[State: CLARIFICATION_REQUIRED]
    ClarifyState --> NullDraft1[Set suggestedDraft: null\nSet requirements: null]
    NullDraft1 --> AskQuestion[Formulate 1 targeted question on missing point]
    AskQuestion --> ReplyFormat[Format reply with A/B/C options template]
    ReplyFormat --> WaitUser([Wait for user answer in chat])
    WaitUser --> Start

    CountCheck -- NO --> MsgOnlyState[State: MESSAGE_ONLY]
    MsgOnlyState --> NullDraft2[Set suggestedDraft: null\nSet requirements: null]
    MsgOnlyState --> ExplainGaps[In reply: summarize what is understood & state missing functional items]
    ExplainGaps --> Recovery[User provides missing details in follow-up]
    Recovery --> ParseInput

    classDef ready fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef clarify fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    classDef msg fill:#e2e3e5,stroke:#6c757d,stroke-width:2px;

    class ReqReady,GenDraft,GenJSON,PopulateEditor ready;
    class ClarifyState,AskQuestion,ReplyFormat clarify;
    class MsgOnlyState,ExplainGaps msg;
```

---

## 3. Two-Stage Delivery Pipeline (Writer AI ➔ Validator Agent ➔ Human Review)

This diagram shows how the generated output transitions across database entities and approval quality gates:

```mermaid
stateDiagram-v2
    [*] --> DraftMode: User opens RequirementDraftEditor

    state "Stage 1: Requirement Writer AI" as Stage1 {
        DraftMode --> ChatClarification: Clarification rounds (1-3)
        ChatClarification --> ReadyState: Points 1-5 satisfied
        ReadyState --> SavedDatabase: User clicks 'Save'
    }

    state "Stage 2: Requirement Validator Agent" as Stage2 {
        SavedDatabase --> StatusNew: Saved to DB with status = 'New'
        StatusNew --> FetchMCP: Validator Agent calls getProjectRequirementById(id)
        
        state QualityGates <<choice>>
        FetchMCP --> QualityGates: Audit against 6 Quality Gates
        
        QualityGates --> FailsGates: Ambiguous words / Missing criteria / No tangible output
        FailsGates --> NeedsClarification: Status remains 'New'\nComment logged with deficiencies
        
        QualityGates --> PassesGates: All gates pass (Yes/No testable, 1-4 ACs, Zero ambiguity)
        PassesGates --> StatusAIReviewed: Update status to 'AI Reviewed' via API
    }

    state "Stage 3: Team Review & Execution" as Stage3 {
        StatusAIReviewed --> HumanReviewUnlocked: Review actions unlocked in UI
        HumanReviewUnlocked --> InReview: Lead/Dev approves (status = 'Review')
        InReview --> Approved: Status = 'Approved'
        Approved --> InProgress: Status = 'In Progress'
    }

    NeedsClarification --> DraftMode: Author refines requirement
```

---

## 4. Acceptance Criteria Validation Architecture

How the Writer AI's criteria are checked by the Validator Agent's Quality Gate:

```mermaid
graph TD
    subgraph WriterAI ["Stage 1: Writer AI (new_user_feature_writer_helper.md)"]
        W1[Bounded Count: 1 to 4 criteria]
        W2[WHAT, Not HOW: No code/APIs/DB]
        W3[Yes/No Assertions]
        W4[Exhaustive Field Names: No 'data' or 'details']
        W5[Coverage: Happy Path + State Change + Empty State]
    end

    subgraph ValidatorAgent ["Stage 2: Validator Agent (requirement_validator_prompt.md)"]
        V1{Criteria Count <= 4?}
        V2{Any tech implementation details?}
        V3{Can QA answer YES/NO without guessing?}
        V4{Contains banned words: any, relevant, appropriate, etc.?}
        V5{Is there tangible observable output?}
    end

    subgraph Verdict ["Verdict"]
        Pass[Pass Quality Gate\nStatus -> 'AI Reviewed'\nScore: 9-10]
        Fail[Fail Quality Gate\nRecommendation -> 'Needs Clarification'\nStatus remains 'New']
    end

    W1 --> V1
    W2 --> V2
    W3 --> V3
    W4 --> V4
    W5 --> V5

    V1 -- No --> Fail
    V2 -- Yes --> Fail
    V3 -- No --> Fail
    V4 -- Yes --> Fail
    V5 -- No --> Fail

    V1 -- Yes --> Pass
    V2 -- No --> Pass
    V3 -- Yes --> Pass
    V4 -- No --> Pass
    V5 -- Yes --> Pass

    classDef pass fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef fail fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    class Pass pass;
    class Fail fail;
```

---

## 5. Code Directory Reference

| Component / Service | File Path | Role in Architecture |
|---|---|---|
| **Chat Panel UI** | [`components/RequirementDraftViews.tsx`](file:///d:/HIQ/ux_requirement_studio/components/RequirementDraftViews.tsx) | `WorkshopAssistant`: captures chat input, renders chat bubbles, sends turns. |
| **Editor Container** | [`components/RequirementDraftEditor.tsx`](file:///d:/HIQ/ux_requirement_studio/components/RequirementDraftEditor.tsx) | Holds `draftText`, handles multi-requirement breakdown, attachment uploads, and save handler. |
| **AI Service Engine** | [`services/geminiService.ts`](file:///d:/HIQ/ux_requirement_studio/services/geminiService.ts) | Implements `refineRequirementDraft`, manages Aura Proxy connection, JSON parsing, and guard invariants. |
| **System Instruction** | [`new_user_feature_writer_helper.md`](file:///d:/HIQ/new_user_feature_writer_helper.md) | The production prompt template with XML tags, 6-point checklist, and validator compliance. |
| **Validator Prompt** | [`requirement_validator_prompt.md`](file:///d:/HIQ/requirement_validator_prompt.md) | System prompt for the downstream quality auditor. |
| **API Config & Endpoints**| [`services/appBuilder/config.ts`](file:///d:/HIQ/ux_requirement_studio/services/appBuilder/config.ts) | Base URLs for Proxy, Config Manager, and AI Studio Manager. |
