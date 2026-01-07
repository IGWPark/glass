# Glass Application - AI State and Memory Management Analysis

**Analysis Date:** 2026-01-07
**Repository:** IGWPark/glass
**Branch:** claude/investigate-memory-management-li18C

---

## Executive Summary

This document analyzes how the Glass application manages **AI conversational memory, user state, and context persistence** - NOT system RAM.

**Key Finding:** Glass uses a **stateless AI architecture** with minimal long-term memory. Context is ephemeral and session-scoped.

---

## 1. User Profile & State Management

### 1.1 User Profile Schema

**Location:** `src/features/common/config/schema.js:2-11`

```javascript
users: {
    columns: [
        { name: 'uid', type: 'TEXT PRIMARY KEY' },
        { name: 'display_name', type: 'TEXT NOT NULL' },
        { name: 'email', type: 'TEXT NOT NULL' },
        { name: 'created_at', type: 'INTEGER' },
        { name: 'auto_update_enabled', type: 'INTEGER DEFAULT 1' },
        { name: 'has_migrated_to_firebase', type: 'INTEGER DEFAULT 0' }
    ]
}
```

**Profile Contents:**
- ✅ **Basic Identity:** UID, display name, email
- ❌ **No Personality Data:** No user preferences, conversation style, tone preferences
- ❌ **No Learning:** No stored facts about user's work, interests, or history
- ❌ **No Context Memory:** AI doesn't remember previous conversations across sessions

**User Management** (`src/features/common/repositories/user/sqlite.repository.js:3-36`):

```javascript
function findOrCreate(user) {
    const query = `
        INSERT INTO users (uid, display_name, email, created_at)
        VALUES (?, ?, ?, ?)
        ON CONFLICT(uid) DO UPDATE SET
            display_name=excluded.display_name,
            email=excluded.email
    `;
    // No personality fields, no memory fields
}
```

### 1.2 What Glass DOES NOT Store About Users

- ❌ Conversation preferences (formal vs casual)
- ❌ Domain expertise (engineer, salesperson, student)
- ❌ Topics of interest
- ❌ Historical context from past sessions
- ❌ User's background (company, role, industry)
- ❌ Learned facts about user's projects/work
- ❌ Relationship context (names mentioned, recurring people)

---

## 2. AI Memory Architecture

### 2.1 Stateless Design Pattern

**Core Principle:** Each AI request is independent with NO persistent memory between sessions.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI Memory Lifecycle                          │
└─────────────────────────────────────────────────────────────────┘

Session Start
    │
    ├─→ Load user profile (uid, email, name only)
    ├─→ Initialize empty conversationHistory array
    ├─→ No previous context loaded
    │
    └─→ AI has ZERO knowledge of:
        - Previous conversations
        - User's background
        - Past topics discussed
        - User preferences

During Active Session (Listen Mode)
    │
    ├─→ conversationHistory array accumulates
    │   └─→ ["me: hello", "them: hi", "me: how are you"]
    │
    ├─→ Previous analysis context (only for continuity)
    │   └─→ { topic, summary, actions } from last analysis
    │
    └─→ Passed to AI in prompt:
        - Last 30 conversation turns only
        - Previous analysis (if exists)
        - Current screenshot (per-request)

Session End
    │
    ├─→ conversationHistory array DELETED
    ├─→ previousAnalysisResult DELETED
    ├─→ Database saves transcripts (for user review only)
    │
    └─→ AI Memory Reset to ZERO

Next Session
    │
    └─→ AI starts fresh with no memory of previous session
```

### 2.2 Conversation History Management

**Location:** `src/features/listen/summary/summaryService.js:38-57`

```javascript
class SummaryService {
    constructor() {
        this.previousAnalysisResult = null;  // Only for current session
        this.analysisHistory = [];           // Max 10 recent analyses
        this.conversationHistory = [];       // ← THE ONLY "MEMORY"
        this.currentSessionId = null;
    }

    addConversationTurn(speaker, text) {
        const conversationText = `${speaker.toLowerCase()}: ${text.trim()}`;
        this.conversationHistory.push(conversationText);
        // NO SIZE LIMIT - grows unbounded during session
    }

    resetConversationHistory() {
        this.conversationHistory = [];       // ← MEMORY WIPE
        this.previousAnalysisResult = null;
        this.analysisHistory = [];
    }
}
```

**Memory Characteristics:**

| Aspect | Implementation | Persistence |
|--------|---------------|-------------|
| **Scope** | Single active session only | Session-scoped |
| **Size Limit** | None (unbounded array) | Until session ends |
| **Cross-Session** | ❌ Not carried over | Deleted on close |
| **Database Link** | ❌ Not loaded from DB | One-way storage |
| **AI Access** | Last 30 turns in prompt | Ephemeral |

### 2.3 Context Window Strategy

**Ask Mode** (`src/features/ask/askService.js:206-210`):

```javascript
_formatConversationForPrompt(conversationTexts) {
    if (!conversationTexts || conversationTexts.length === 0) {
        return 'No conversation history available.';
    }
    return conversationTexts.slice(-30).join('\n');  // ← Last 30 turns only
}
```

**Listen Mode** (`src/features/listen/summary/summaryService.js:65-68`):

```javascript
formatConversationForPrompt(conversationTexts, maxTurns = 30) {
    if (conversationTexts.length === 0) return '';
    return conversationTexts.slice(-maxTurns).join('\n');  // ← Last 30 turns
}
```

**Implications:**
- AI can only "remember" last ~15 exchanges (30 turns = 15 back-and-forth)
- Conversations beyond 30 turns lose early context
- No long-term memory of facts mentioned earlier

---

## 3. Prompt Preset System (AI Personalities)

### 3.1 Preset Architecture

**Location:** `src/features/common/repositories/preset/sqlite.repository.js`

**Database Schema:**

```javascript
prompt_presets: {
    columns: [
        { name: 'id', type: 'TEXT PRIMARY KEY' },
        { name: 'uid', type: 'TEXT NOT NULL' },
        { name: 'title', type: 'TEXT NOT NULL' },
        { name: 'prompt', type: 'TEXT NOT NULL' },     // ← The "personality"
        { name: 'is_default', type: 'INTEGER NOT NULL' },
        { name: 'created_at', type: 'INTEGER' }
    ]
}
```

**Default Presets** (`src/features/common/services/sqliteClient.js:223-229`):

```javascript
const defaultPresets = [
    ['school', 'School', 'You are a school and lecture assistant...'],
    ['meetings', 'Meetings', 'You are a meeting assistant...'],
    ['sales', 'Sales', 'You are a real-time AI sales assistant...'],
    ['recruiting', 'Recruiting', 'You are a recruiting assistant...'],
    ['customer-support', 'Customer Support', 'You are a customer support assistant...'],
];
```

### 3.2 How Presets Work

**Prompt Building** (`src/features/common/prompts/promptBuilder.js:3-18`):

```javascript
function buildSystemPrompt(promptParts, customPrompt = '', googleSearchEnabled = true) {
    const sections = [
        promptParts.intro,               // ← AI personality/role
        '\n\n',
        promptParts.formatRequirements,  // ← Response structure rules
    ];

    if (googleSearchEnabled) {
        sections.push('\n\n', promptParts.searchUsage);
    }

    sections.push(
        '\n\n',
        promptParts.content,             // ← Domain-specific instructions
        '\n\nUser-provided context\n-----\n',
        customPrompt,                    // ← User's custom additions
        '\n-----\n\n',
        promptParts.outputInstructions   // ← Final formatting rules
    );

    return sections.join('');
}
```

**Usage in Ask Service** (`src/features/ask/askService.js:257`):

```javascript
const conversationHistory = this._formatConversationForPrompt(conversationHistoryRaw);
const systemPrompt = getSystemPrompt('pickle_glass_analysis', conversationHistory, false);

const messages = [
    { role: 'system', content: systemPrompt },  // ← Preset + context
    {
        role: 'user',
        content: [
            { type: 'text', text: `User Request: ${userPrompt.trim()}` },
            { type: 'image_url', image_url: { url: `data:image/jpeg;base64,${screenshotBase64}` } }
        ]
    }
];
```

**Preset Characteristics:**

| Feature | Implementation | Notes |
|---------|---------------|-------|
| **Customization** | User can create/edit presets | Stored per-user in DB |
| **Scope** | System prompt only | Doesn't persist across messages |
| **State** | Stateless | No learning or adaptation |
| **Context Injection** | `{{CONVERSATION_HISTORY}}` placeholder | Last 30 turns injected |

---

## 4. Context Sources for AI

### 4.1 What the AI Sees Per Request

**Ask Mode** (`src/features/ask/askService.js:259-274`):

```javascript
const messages = [
    {
        role: 'system',
        content: systemPrompt  // Contains:
                               // - Preset personality
                               // - Last 30 conversation turns
                               // - Output format instructions
    },
    {
        role: 'user',
        content: [
            { type: 'text', text: `User Request: ${userPrompt}` },
            { type: 'image_url', image_url: { url: screenshot } }  // ← Per-request screenshot
        ]
    }
];
```

**Listen Mode Analysis** (`src/features/listen/summary/summaryService.js:82-91`):

```javascript
// Include previous analysis for continuity
if (this.previousAnalysisResult) {
    contextualPrompt = `
Previous Analysis Context:
- Main Topic: ${this.previousAnalysisResult.topic.header}
- Key Points: ${this.previousAnalysisResult.summary.slice(0, 3).join(', ')}
- Last Actions: ${this.previousAnalysisResult.actions.slice(0, 2).join(', ')}

Please build upon this context while analyzing the new conversation segments.
`;
}
```

**Context Summary:**

| Source | Persistence | Scope | Access Method |
|--------|------------|-------|---------------|
| **User Prompt** | Request-only | Single message | Direct input |
| **Screenshot** | Request-only | Current screen state | Base64 image |
| **Conversation History** | Session-only | Last 30 turns | Injected in system prompt |
| **Previous Analysis** | Session-only | Last analysis result | Injected in prompt |
| **Database Transcripts** | ❌ Not used | Archived only | User-facing only |
| **User Profile** | ❌ Not used | Stored but not passed to AI | N/A |

---

## 5. Database Storage vs AI Memory

### 5.1 What Gets Stored in Database

**Transcripts** (`src/features/common/config/schema.js:24-36`):

```javascript
transcripts: {
    columns: [
        { name: 'id', type: 'TEXT PRIMARY KEY' },
        { name: 'session_id', type: 'TEXT NOT NULL' },
        { name: 'speaker', type: 'TEXT' },
        { name: 'text', type: 'TEXT' },
        { name: 'created_at', type: 'INTEGER' }
    ]
}
```

**AI Messages** (`src/features/common/config/schema.js:37-49`):

```javascript
ai_messages: {
    columns: [
        { name: 'id', type: 'TEXT PRIMARY KEY' },
        { name: 'session_id', type: 'TEXT NOT NULL' },
        { name: 'role', type: 'TEXT' },           // user/assistant
        { name: 'content', type: 'TEXT' },        // Full message
        { name: 'model', type: 'TEXT' },
        { name: 'created_at', type: 'INTEGER' }
    ]
}
```

**Summaries** (`src/features/common/config/schema.js:50-63`):

```javascript
summaries: {
    columns: [
        { name: 'session_id', type: 'TEXT PRIMARY KEY' },
        { name: 'text', type: 'TEXT' },           // Full analysis
        { name: 'tldr', type: 'TEXT' },           // Key points
        { name: 'bullet_json', type: 'TEXT' },    // Structured topics
        { name: 'action_json', type: 'TEXT' },    // Action items
        { name: 'model', type: 'TEXT' }
    ]
}
```

### 5.2 Critical Gap: No Retrieval System

**Current Flow:**

```
User speaks → STT → Save to DB (transcripts table)
                            ↓
                      [STORED BUT NEVER RETRIEVED]
                            ↓
Next AI request → NO ACCESS to past transcripts
                → Only sees last 30 turns from current session
```

**What's Missing:**

```
❌ NO RAG (Retrieval-Augmented Generation)
   - Database has full conversation history
   - AI cannot access it
   - No semantic search over past conversations

❌ NO Long-Term Memory
   - User said "I work at Microsoft" 3 weeks ago
   - AI doesn't remember in current session
   - No facts stored about user

❌ NO Context Continuity
   - Session ends → all context lost
   - Next session starts fresh
   - User must re-explain background every time
```

---

## 6. Prompt Template Analysis

### 6.1 "pickle_glass_analysis" Prompt (Main AI Personality)

**Location:** `src/features/common/prompts/promptTemplates.js:238-403`

**Priority Hierarchy:**

```javascript
pickle_glass_analysis: {
    intro: `You are Pickle, developed and created by Pickle, and you are the user's live-meeting co-pilot.`,

    formatRequirements: `
    Execute in the following priority order:

    1. QUESTION_ANSWERING_PRIORITY:
       - If a question is presented at the end of transcript → answer it
       - Infer intent from garbled/unclear text (STT errors)
       - Confidence threshold: 50%+ → treat as question

    2. TERM_DEFINITION_PRIORITY:
       - Define proper nouns in last 10-15 words of transcript
       - Company names, technical terms, domain-specific terms

    3. CONVERSATION_ADVANCEMENT_PRIORITY:
       - Suggest 1-3 follow-up questions to drive conversation
       - If project description given → ask clarifying questions

    4. OBJECTION_HANDLING_PRIORITY:
       - If sales/negotiation context → handle objections

    5. SCREEN_PROBLEM_SOLVING_PRIORITY:
       - Solve visible problems on screen (e.g., LeetCode)

    6. PASSIVE_ACKNOWLEDGMENT:
       - Fallback: "Not sure what you need help with"
       - Brief summary of last 1-2 events
    `
}
```

**Key Features:**

| Aspect | Implementation | Notes |
|--------|---------------|-------|
| **Context Awareness** | `{{CONVERSATION_HISTORY}}` in output | Last 30 turns only |
| **Intent Detection** | 50% confidence threshold | Handles STT errors |
| **Multimodal** | Screenshot + audio transcript | Per-request only |
| **Adaptive Behavior** | Priority hierarchy for different scenarios | Rule-based, not learned |

### 6.2 Specialized Prompt Presets

**Sales Assistant** (`promptTemplates.js:113-142`):

```javascript
sales: {
    intro: `You are a sales call assistant. Your job is to provide the exact words the salesperson should say to prospects during sales calls.`,

    content: `Examples:

    Prospect: "Tell me about your product"
    You: "Our platform helps companies like yours reduce operational costs by 30%..."

    Prospect: "I need to think about it"
    You: "I completely understand. What specific concerns can I address for you today?"
    `
}
```

**Meeting Assistant** (`promptTemplates.js:144-173`):

```javascript
meeting: {
    intro: `You are a meeting assistant. Your job is to provide the exact words to say during professional meetings.`,

    content: `Examples:

    Participant: "What's the status on the project?"
    You: "We're currently on track. We've completed 75% of deliverables..."
    `
}
```

**Characteristics:**

- ✅ **Domain-Specific Prompts:** Tailored for different use cases
- ✅ **Example-Driven:** Few-shot learning via examples
- ❌ **No User Customization Variables:** Can't inject "{user_name}" or "{company}"
- ❌ **No Dynamic Context:** Same prompt for all users in same preset

---

## 7. State Management During Sessions

### 7.1 Ask Service State

**Location:** `src/features/ask/askService.js:129-136`

```javascript
this.state = {
    isVisible: false,        // UI state
    isLoading: false,        // Request in progress
    isStreaming: false,      // Streaming response
    currentQuestion: '',     // Last user input
    currentResponse: '',     // Current AI response
    showTextInput: true,     // UI toggle
};
```

**State Lifecycle:**

```
User asks question
    ↓
state.currentQuestion = "user input"
state.isLoading = true
    ↓
LLM responds (streaming)
    ↓
state.isStreaming = true
state.currentResponse += token  // Accumulate response
    ↓
Stream complete
    ↓
Save to database (ai_messages table)
    ↓
Window closes
    ↓
state.currentQuestion = ''
state.currentResponse = ''
    ↓
MEMORY WIPED
```

**Critical:** `currentResponse` is NOT reused in next request. Each request is independent.

### 7.2 Summary Service State

**Location:** `src/features/listen/summary/summaryService.js:9-17`

```javascript
class SummaryService {
    constructor() {
        this.previousAnalysisResult = null;  // ← Only for session continuity
        this.analysisHistory = [];           // ← Ring buffer (max 10)
        this.conversationHistory = [];       // ← Unbounded during session
        this.currentSessionId = null;
    }
}
```

**Analysis Trigger** (`summaryService.js:305-320`):

```javascript
async triggerAnalysisIfNeeded() {
    // Trigger every 5 conversation turns
    if (this.conversationHistory.length >= 5 && this.conversationHistory.length % 5 === 0) {
        const data = await this.makeOutlineAndRequests(this.conversationHistory);

        // Save to database
        summaryRepository.saveSummary({
            sessionId: this.currentSessionId,
            text: responseText,
            tldr: structuredData.summary.join('\n'),
            bullet_json: JSON.stringify(structuredData.topic.bullets),
            action_json: JSON.stringify(structuredData.actions)
        });

        // Update in-memory state for next analysis
        this.previousAnalysisResult = structuredData;
    }
}
```

**State Continuity Pattern:**

```
Turn 1-4: Accumulate conversation
Turn 5: Generate analysis A
    └─→ previousAnalysisResult = A
Turn 6-9: Accumulate more conversation
Turn 10: Generate analysis B
    └─→ Use previousAnalysisResult (A) for context
    └─→ previousAnalysisResult = B
```

**Benefit:** Analysis builds on previous analysis within same session
**Limitation:** Resets to null when session ends

---

## 8. What Glass DOES NOT Have

### 8.1 No Long-Term Memory System

❌ **No RAG (Retrieval-Augmented Generation)**
- Database stores all transcripts but AI cannot query them
- No semantic search over past conversations
- No fact extraction and storage

❌ **No User Modeling**
- Doesn't learn user preferences over time
- Can't adapt tone/style to individual users
- No personalization beyond preset selection

❌ **No Cross-Session Context**
- Each session starts with zero context
- User must re-explain background every time
- No continuity between meetings/calls

❌ **No Entity Memory**
- Doesn't remember people mentioned
- No tracking of projects, companies, or relationships
- No fact database about user's work

### 8.2 No Advanced Context Management

❌ **No Context Prioritization**
- Last 30 turns treated equally (no importance weighting)
- No distinction between critical facts vs small talk
- No automatic summarization of older context

❌ **No External Knowledge Integration**
- Google Search capability exists in prompts but implementation unclear
- No real-time web search for current information
- No integration with user's calendar, email, CRM

❌ **No Multi-Modal Memory**
- Screenshots are per-request only
- No visual memory of past screens
- No tracking of document edits or code changes

---

## 9. Comparison with Modern AI Memory Systems

### 9.1 Glass vs. ChatGPT/Claude/Gemini Memory

| Feature | Glass | Modern AI Assistants |
|---------|-------|---------------------|
| **Session Memory** | ✅ Last 30 turns | ✅ Full conversation |
| **Cross-Session Memory** | ❌ None | ✅ Persistent memory |
| **User Facts** | ❌ Not stored | ✅ "User is a Python developer" |
| **Preferences** | ❌ None | ✅ "User prefers concise answers" |
| **Context Window** | 30 turns (~1500 tokens) | 128K-2M tokens |
| **RAG** | ❌ None | ✅ Knowledge retrieval |
| **Learning** | ❌ Stateless | ✅ Adapts over time |

### 9.2 Glass vs. Enterprise AI Assistants (Microsoft Copilot, etc.)

| Feature | Glass | Enterprise AI |
|---------|-------|--------------|
| **Integration** | ❌ Standalone | ✅ Email, calendar, files |
| **Context Sources** | Audio + screenshot | Email, docs, chat, code |
| **Personalization** | Preset selection only | ✅ Per-user adaptation |
| **Team Memory** | ❌ Individual only | ✅ Shared team context |
| **Security** | Local SQLite | Enterprise auth + encryption |

---

## 10. Architectural Implications

### 10.1 Why Stateless?

**Potential Reasons for Current Design:**

1. **Privacy:**
   - No persistent AI memory = no long-term data retention risk
   - User controls what's stored (database only)
   - AI provider doesn't accumulate user data

2. **Simplicity:**
   - No need for complex retrieval systems
   - No vector databases or embeddings
   - Easier to debug and maintain

3. **Cost:**
   - No expensive RAG infrastructure
   - Minimal token usage (only last 30 turns)
   - No embedding generation costs

4. **Transparency:**
   - User knows exactly what AI sees
   - No "hidden" context or learned biases
   - Predictable behavior

### 10.2 Trade-Offs

**Advantages:**

✅ **Privacy-Preserving:** AI can't leak information from past sessions
✅ **Predictable:** Same input → same output (no personalization drift)
✅ **Low Cost:** Minimal token usage and infrastructure
✅ **Fast:** No retrieval latency

**Disadvantages:**

❌ **Poor UX:** User must repeat context in every session
❌ **Limited Utility:** Can't help with long-term projects
❌ **No Learning:** Doesn't improve for individual users
❌ **Context Loss:** Important information from early in session gets lost

---

## 11. Potential Improvements

### 11.1 Short-Term Enhancements (No Architecture Change)

1. **Increase Context Window:**
   ```javascript
   // Current: Last 30 turns
   return conversationTexts.slice(-30).join('\n');

   // Improved: Last 100 turns
   return conversationTexts.slice(-100).join('\n');
   ```

2. **Smart Context Summarization:**
   ```javascript
   // If conversation > 30 turns, summarize older context
   if (conversationTexts.length > 30) {
       const oldContext = summarize(conversationTexts.slice(0, -30));
       const recentContext = conversationTexts.slice(-30).join('\n');
       return oldContext + '\n---\n' + recentContext;
   }
   ```

3. **User Context Injection:**
   ```javascript
   // Add user profile to system prompt
   const userContext = `User Profile: ${user.display_name} (${user.email})`;
   // Could be extended with: role, company, preferences
   ```

### 11.2 Long-Term Enhancements (Architecture Changes)

1. **RAG Implementation:**
   ```
   User query
       ↓
   Search past transcripts (vector similarity)
       ↓
   Retrieve top 5 relevant conversations
       ↓
   Inject into prompt alongside last 30 turns
   ```

2. **Entity Memory System:**
   ```javascript
   entities: {
       columns: [
           { name: 'id', type: 'TEXT PRIMARY KEY' },
           { name: 'uid', type: 'TEXT NOT NULL' },
           { name: 'entity_type', type: 'TEXT' },  // person, company, project
           { name: 'name', type: 'TEXT' },
           { name: 'facts', type: 'TEXT' },        // JSON array of facts
           { name: 'last_mentioned', type: 'INTEGER' }
       ]
   }
   ```

3. **Session Continuity:**
   ```javascript
   // On new session start, check for related past sessions
   const relatedSessions = findSimilarSessions(currentTopic);
   const context = summarizeRelatedSessions(relatedSessions);
   // Inject summary into system prompt
   ```

4. **User Preference Learning:**
   ```javascript
   user_preferences: {
       columns: [
           { name: 'uid', type: 'TEXT PRIMARY KEY' },
           { name: 'response_style', type: 'TEXT' },   // concise, detailed, casual
           { name: 'domain_expertise', type: 'TEXT' }, // engineering, sales, etc.
           { name: 'language_preference', type: 'TEXT' }
       ]
   }
   ```

---

## 12. Conclusion

### Key Findings

1. **Stateless AI Architecture:**
   - Glass uses a **completely stateless AI design**
   - No persistent memory between sessions
   - Context limited to last 30 conversation turns

2. **Minimal User Profiling:**
   - User profiles contain only basic identity (uid, email, name)
   - No personality data, preferences, or learned facts
   - Preset system provides domain-specific prompts but no personalization

3. **Database Storage ≠ AI Memory:**
   - Full conversation history stored in SQLite
   - **Not used by AI** - stored for user review only
   - No retrieval or search mechanism

4. **Session-Scoped Context:**
   - Conversation history accumulated during active session
   - Previous analysis result used for continuity
   - **All context wiped when session ends**

### Architectural Philosophy

Glass prioritizes:
- ✅ **Privacy:** No long-term AI memory
- ✅ **Simplicity:** Stateless, predictable behavior
- ✅ **Cost-Efficiency:** Minimal token usage
- ❌ **User Experience:** Poor cross-session continuity
- ❌ **Personalization:** No learning or adaptation

### Recommended Next Steps

**If goal is to add AI memory:**

1. **Phase 1 (Low Effort):**
   - Increase context window to 100+ turns
   - Add user context fields to database
   - Inject user profile into system prompts

2. **Phase 2 (Medium Effort):**
   - Implement conversation summarization
   - Add entity extraction and storage
   - Create "session notes" for continuity

3. **Phase 3 (High Effort):**
   - Build RAG system with vector search
   - Implement long-term user memory
   - Add cross-session context retrieval

**If goal is to maintain privacy:**
- Current architecture is well-designed
- Consider minor UX improvements (context summarization)
- Document the stateless design as a feature, not a bug

---

## Appendix: File Reference Index

**User State Management:**
- `src/features/common/repositories/user/sqlite.repository.js` - User CRUD operations
- `src/features/common/config/schema.js` - Database schema (users table)

**AI Context Management:**
- `src/features/listen/summary/summaryService.js` - Conversation history accumulation
- `src/features/ask/askService.js` - Ask mode state and context
- `src/features/common/prompts/promptBuilder.js` - System prompt construction

**Prompt Presets:**
- `src/features/common/prompts/promptTemplates.js` - All AI personalities
- `src/features/common/repositories/preset/sqlite.repository.js` - Preset storage

**Database Schema:**
- `src/features/common/config/schema.js` - Complete schema (transcripts, ai_messages, summaries)

---

**Analysis Complete**
Generated by Claude Code on branch `claude/investigate-memory-management-li18C`
