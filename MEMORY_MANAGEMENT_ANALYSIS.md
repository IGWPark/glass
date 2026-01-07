# Glass Application - Memory Management Analysis

**Analysis Date:** 2026-01-07
**Repository:** IGWPark/glass
**Branch:** claude/investigate-memory-management-li18C

---

## Executive Summary

This document provides a comprehensive reverse engineering analysis of the Glass application's memory management system, including data sources, memory management algorithms, input/output flows, and key logic.

---

## 1. Data Sources & Storage Architecture

### 1.1 Primary Data Store: SQLite Database

**Location:** `~/Library/Application Support/Glass/pickleglass.db` (macOS) or `%APPDATA%\Glass\pickleglass.db` (Windows)

**Implementation:** `src/features/common/services/sqliteClient.js`

**Key Features:**
- **Database Library:** `better-sqlite3` - synchronous SQLite3 bindings
- **Write-Ahead Logging (WAL):** Enabled via `PRAGMA journal_mode = WAL` for better concurrency
- **Schema Management:** Dynamic schema synchronization on startup
- **Memory Optimization:** Single database connection reused throughout application lifecycle

**Database Schema** (`src/features/common/config/schema.js`):

```javascript
// Core Tables:
- users: User profiles and preferences
- sessions: Recording/conversation sessions (listen/ask modes)
- transcripts: Speech-to-text transcription records
- ai_messages: LLM conversation history
- summaries: AI-generated session summaries
- prompt_presets: User-defined AI prompts
- provider_settings: API keys and model selections
- ollama_models: Local AI model management
- whisper_models: Local STT model management
- shortcuts: Keyboard shortcut configurations
- permissions: User permission states
```

### 1.2 Secondary Storage: Firebase (Optional)

**Implementation:** `src/features/common/services/firebaseClient.js`

**Purpose:** Cloud sync for cross-device data synchronization (disabled by default)

### 1.3 Temporary Storage

**Audio Buffers:**
- In-memory JavaScript arrays for audio chunking
- Ring buffer pattern with max size constraints
- WASM linear memory for AEC (Acoustic Echo Cancellation) processing

---

## 2. Memory Management Algorithms

### 2.1 Audio Processing Pipeline Memory Management

**Location:** `src/ui/listen/audioCore/listenCapture.js`

#### Audio Buffer Management Strategy

```javascript
// Fixed-size buffer with splice-based consumption
let audioBuffer = [];
const samplesPerChunk = SAMPLE_RATE * AUDIO_CHUNK_DURATION; // 2400 samples

micProcessor.onaudioprocess = (e) => {
    const inputData = e.inputBuffer.getChannelData(0);
    audioBuffer.push(...inputData);  // Accumulate samples

    while (audioBuffer.length >= samplesPerChunk) {
        let chunk = audioBuffer.splice(0, samplesPerChunk);  // Consume
        // Process chunk...
    }
};
```

**Memory Characteristics:**
- **Growth Pattern:** Linear accumulation until threshold
- **Cleanup Strategy:** Immediate consumption via `splice()` (O(n) operation)
- **Bounded Growth:** Maximum buffer size = `samplesPerChunk`
- **Memory Footprint:** ~19.2KB per buffer (2400 samples * 8 bytes/Float64)

#### System Audio Ring Buffer

```javascript
let systemAudioBuffer = [];
const MAX_SYSTEM_BUFFER_SIZE = 10;

window.api.listenCapture.onSystemAudioData((event, { data }) => {
    systemAudioBuffer.push({
        data: data,
        timestamp: Date.now(),
    });

    // Ring buffer: Keep only last 10 chunks
    if (systemAudioBuffer.length > MAX_SYSTEM_BUFFER_SIZE) {
        systemAudioBuffer = systemAudioBuffer.slice(-MAX_SYSTEM_BUFFER_SIZE);
    }
});
```

**Memory Characteristics:**
- **Fixed Size:** Maximum 10 audio chunks
- **FIFO Eviction:** Oldest chunks removed first
- **Memory Footprint:** ~10 chunks * base64-encoded PCM data (~9.6KB each) = ~96KB
- **Cleanup:** Automatic via array slicing

### 2.2 WASM Memory Management (AEC Processing)

**Location:** `src/ui/listen/audioCore/listenCapture.js:107-189`

#### Memory Allocation Pattern

```javascript
function int16PtrFromFloat32(mod, f32) {
    const len = f32.length;
    const bytes = len * 2;
    const ptr = mod._malloc(bytes);  // Allocate WASM memory

    // Convert Float32 -> Int16 in WASM heap
    const heapBuf = mod.HEAPU8.buffer;
    const i16 = new Int16Array(heapBuf, ptr, len);
    // ... conversion logic

    return { ptr, view: i16 };
}

function runAecSync(micF32, sysF32) {
    const frameSize = 160;
    const numFrames = Math.floor(micF32.length / frameSize);

    for (let i = 0; i < numFrames; i++) {
        const micPtr = int16PtrFromFloat32(aecMod, micFrame);
        const echoPtr = int16PtrFromFloat32(aecMod, echoFrame);
        const outPtr = aecMod._malloc(frameSize * 2);

        aecMod.cancel(aecPtr, micPtr.ptr, echoPtr.ptr, outPtr, frameSize);

        // Critical: Immediate deallocation
        aecMod._free(micPtr.ptr);
        aecMod._free(echoPtr.ptr);
        aecMod._free(outPtr);
    }
}
```

**Memory Characteristics:**
- **Allocation Strategy:** Frame-by-frame (160 samples per frame)
- **Lifetime:** Immediate deallocation after processing
- **Memory Footprint per Frame:** 3 allocations × 320 bytes = 960 bytes
- **Total Processing:** 15 frames per chunk = 14.4KB allocated and freed per 100ms
- **Critical Feature:** Manual memory management with explicit `_free()` calls prevents WASM heap growth

### 2.3 Database Connection Management

**Location:** `src/features/common/services/sqliteClient.js`

#### Singleton Connection Pattern

```javascript
class SQLiteClient {
    constructor() {
        this.db = null;
        this.dbPath = null;
    }

    connect(dbPath) {
        if (this.db) {
            console.log('[SQLiteClient] Already connected.');
            return;  // Reuse existing connection
        }

        this.db = new Database(this.dbPath);
        this.db.pragma('journal_mode = WAL');  // Enable WAL mode
    }

    close() {
        if (this.db) {
            this.db.close();
            this.db = null;
        }
    }
}
```

**Memory Characteristics:**
- **Connection Pooling:** Single connection, no pool overhead
- **WAL Mode:** Separate write-ahead log file reduces database lock contention
- **Prepared Statements:** Implicit caching by better-sqlite3 for repeated queries
- **Cleanup:** Explicit close on app shutdown (`src/index.js:291-296`)

### 2.4 Session Memory Management

**Location:** `src/features/common/repositories/session/sqlite.repository.js`

#### Session Lifecycle & Cleanup

```javascript
function getOrCreateActive(uid, requestedType = 'ask') {
    const db = sqliteClient.getDb();

    // Find ANY active session (ended_at IS NULL)
    const findQuery = `
        SELECT id, session_type FROM sessions
        WHERE uid = ? AND ended_at IS NULL
        ORDER BY CASE session_type WHEN 'listen' THEN 1 WHEN 'ask' THEN 2 ELSE 3 END
        LIMIT 1
    `;

    const activeSession = db.prepare(findQuery).get(uid);

    if (activeSession) {
        touch(activeSession.id);  // Update timestamp
        return activeSession.id;
    } else {
        return create(uid, requestedType);  // Create new
    }
}

function endAllActiveSessions(uid) {
    const now = Math.floor(Date.now() / 1000);
    const query = `UPDATE sessions SET ended_at = ?, updated_at = ? WHERE ended_at IS NULL AND uid = ?`;
    return db.prepare(query).run(now, now, uid);
}
```

**Memory Characteristics:**
- **Orphan Prevention:** Zombie sessions cleaned up on app restart (`src/index.js:266`)
- **Session Reuse:** Single active session per user reduces database rows
- **Explicit Lifecycle:** Sessions marked with start/end timestamps for garbage collection

#### Empty Session Cleanup

```javascript
// Location: src/features/common/services/sqliteClient.js:183-207
cleanupEmptySessions() {
    const query = `
        SELECT s.id FROM sessions s
        LEFT JOIN transcripts t ON s.id = t.session_id
        LEFT JOIN ai_messages a ON s.id = a.session_id
        LEFT JOIN summaries su ON s.id = su.session_id
        WHERE t.id IS NULL AND a.id IS NULL AND su.session_id IS NULL
    `;

    const rows = this.db.prepare(query).all();
    const idsToDelete = rows.map(r => r.id);

    if (idsToDelete.length > 0) {
        const placeholders = idsToDelete.map(() => '?').join(',');
        const deleteQuery = `DELETE FROM sessions WHERE id IN (${placeholders})`;
        this.db.prepare(deleteQuery).run(idsToDelete);
    }
}
```

**Cleanup Strategy:**
- **Trigger:** On application startup after database initialization
- **Logic:** Delete sessions with no associated transcripts, AI messages, or summaries
- **Memory Impact:** Prevents database bloat from abandoned sessions

---

## 3. Key Process Input/Output Flows

### 3.1 Audio Capture → Transcription Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Audio Capture Pipeline                          │
└─────────────────────────────────────────────────────────────────────┘

[Microphone]
    │
    ├─→ navigator.mediaDevices.getUserMedia()
    │   └─→ MediaStream (24kHz, mono, PCM)
    │
    ├─→ AudioContext.createScriptProcessor(4096)
    │   └─→ onaudioprocess event (every 170ms)
    │       └─→ Float32Array input buffer (4096 samples)
    │
    ├─→ Accumulation Buffer (audioBuffer: [])
    │   └─→ Collect until 2400 samples (100ms chunk)
    │
    ├─→ AEC Processing (if system audio available)
    │   ├─→ WASM module allocation (micPtr, echoPtr, outPtr)
    │   ├─→ Frame-by-frame processing (160 samples/frame)
    │   └─→ Immediate deallocation (_free)
    │
    ├─→ Float32 → Int16 conversion
    │   └─→ Scaling: s * 0x7FFF (positive) or s * 0x8000 (negative)
    │
    ├─→ Base64 encoding
    │   └─→ arrayBufferToBase64(int16Array.buffer)
    │
    └─→ IPC: window.api.listenCapture.sendMicAudioContent()
        │
        └─→ [Main Process]
            │
            ├─→ sttService.sendMicAudioContent(base64Data)
            │   │
            │   ├─→ [OpenAI] WebSocket JSON message
            │   │   └─→ { type: 'input_audio_buffer.append', audio: base64 }
            │   │
            │   ├─→ [Gemini] Live API payload
            │   │   └─→ { audio: { data: base64, mimeType: 'audio/pcm;rate=24000' } }
            │   │
            │   └─→ [Deepgram] Raw Buffer (Buffer.from(base64, 'base64'))
            │
            └─→ [STT Response Handler]
                ├─→ Partial transcripts → UI update
                ├─→ Final transcripts → Debounce (2000ms)
                └─→ Database: transcripts table
                    └─→ INSERT (session_id, speaker, text, timestamp)
```

**Input:**
- Raw PCM audio from microphone (Float32Array, 24kHz, mono)
- System audio reference for echo cancellation (optional)

**Output:**
- Base64-encoded PCM16 chunks sent to STT provider
- Database records in `transcripts` table

**Memory Checkpoints:**
1. **Audio Context Buffer:** 4096 samples × 8 bytes = 32KB (transient)
2. **Accumulation Buffer:** 2400 samples × 8 bytes = 19.2KB (transient)
3. **WASM Processing:** 960 bytes × 15 frames = 14.4KB (immediately freed)
4. **Base64 Encoding:** 2400 samples × 2 bytes × 1.33 = 6.4KB (transient)
5. **Database Record:** ~100-500 bytes per transcript (persistent)

### 3.2 STT Session Renewal Flow (Anti-Timeout Mechanism)

```
┌─────────────────────────────────────────────────────────────────────┐
│         STT Session Lifecycle Management (30min+ sessions)          │
└─────────────────────────────────────────────────────────────────────┘

[initializeSttSessions]
    │
    ├─→ Create WebSocket connections (mySttSession, theirSttSession)
    │   └─→ OpenAI/Gemini/Deepgram STT sessions
    │
    ├─→ Start Keep-Alive Heartbeat (every 60s)
    │   └─→ setInterval(() => { ws.ping() }, 60000)
    │       └─→ Prevents idle disconnection (< 2-5min timeout)
    │
    └─→ Schedule Session Renewal (after 20min)
        └─→ setTimeout(async () => {
            │
            ├─→ [renewSessions]
            │   ├─→ Create NEW WebSocket connections
            │   │   └─→ Fresh 30min timeout window
            │   │
            │   ├─→ Update session pointers (mySttSession, theirSttSession)
            │   │   └─→ Audio pipeline continues uninterrupted
            │   │
            │   └─→ Close OLD connections after 2s overlap
            │       └─→ setTimeout(() => { oldSession.close() }, 2000)
            │
            └─→ Reset timers for new 20min cycle
        }, 1200000)  // 20 minutes

[Audio Flow During Renewal]
    │
    ├─→ T=0-2s: Both OLD and NEW sockets active
    │   └─→ Prevents packet loss during handoff
    │
    └─→ T=2s+: Only NEW socket active
        └─→ Old socket gracefully closed
```

**Key Logic** (`src/features/listen/stt/sttService.js:475-544`):

```javascript
const KEEP_ALIVE_INTERVAL_MS = 60 * 1000;         // 1 minute
const SESSION_RENEW_INTERVAL_MS = 20 * 60 * 1000; // 20 minutes
const SOCKET_OVERLAP_MS = 2 * 1000;               // 2 seconds

// Keep-alive prevents idle timeout
this.keepAliveInterval = setInterval(() => {
    this._sendKeepAlive();
}, KEEP_ALIVE_INTERVAL_MS);

// Proactive renewal before 30min hard timeout
this.sessionRenewTimeout = setTimeout(async () => {
    await this.renewSessions(language);
}, SESSION_RENEW_INTERVAL_MS);
```

**Memory Impact:**
- **Overlap Period:** 2 WebSocket connections × ~10KB = 20KB (transient)
- **Prevents:** Session loss and reconnection overhead

### 3.3 Conversation Analysis Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│              Real-time Conversation Analysis Pipeline               │
└─────────────────────────────────────────────────────────────────────┘

[Transcription Complete]
    │
    ├─→ sttService.flushMyCompletion() / flushTheirCompletion()
    │   └─→ Final text: "me: hello world" or "them: hi there"
    │
    ├─→ listenService.handleTranscriptionComplete(speaker, text)
    │   │
    │   ├─→ Database: transcripts table
    │   │   └─→ INSERT (session_id, speaker, text, timestamp)
    │   │
    │   └─→ summaryService.addConversationTurn(speaker, text)
    │       │
    │       ├─→ conversationHistory.push(`${speaker.toLowerCase()}: ${text}`)
    │       │   └─→ In-memory array accumulation
    │       │
    │       └─→ triggerAnalysisIfNeeded()
    │           │
    │           └─→ [Trigger Condition: Every N turns or time interval]
    │
    └─→ summaryService.makeOutlineAndRequests()
        │
        ├─→ Format conversation for LLM
        │   └─→ Last 30 turns joined with newlines
        │
        ├─→ Build contextual prompt
        │   ├─→ Base system prompt from promptTemplates
        │   └─→ Previous analysis context (topic, key points)
        │
        ├─→ Call LLM API
        │   ├─→ modelStateService.getCurrentModelInfo('llm')
        │   ├─→ createLLM(provider, { apiKey, model, temperature: 0.7 })
        │   └─→ llm.chat([{ role: 'system', content: prompt }, ...])
        │
        ├─→ Parse structured response
        │   └─→ { summary: [], topic: {}, actions: [], followUps: [] }
        │
        ├─→ Database: summaries table
        │   └─→ UPSERT (session_id, text, tldr, bullet_json, action_json)
        │
        └─→ Update UI via IPC
            └─→ listenWindow.webContents.send('summary-update', data)
```

**Input:**
- Conversation history (array of "speaker: text" strings)
- Previous analysis context

**Output:**
- Structured summary (JSON)
- Database record in `summaries` table

**Memory Management** (`src/features/listen/summary/summaryService.js:38-56`):

```javascript
addConversationTurn(speaker, text) {
    const conversationText = `${speaker.toLowerCase()}: ${text.trim()}`;
    this.conversationHistory.push(conversationText);
    // No explicit limit - grows unbounded during session
}

resetConversationHistory() {
    this.conversationHistory = [];
    this.previousAnalysisResult = null;
    this.analysisHistory = [];
}
```

**Memory Characteristics:**
- **Unbounded Growth:** Conversation history accumulates during active session
- **Typical Size:** ~100 bytes per turn × N turns
- **Cleanup:** Only on session close via `resetConversationHistory()`

**Analysis History Ring Buffer:**

```javascript
this.analysisHistory.push({
    timestamp: Date.now(),
    data: structuredData,
    conversationLength: conversationTexts.length,
});

// Keep last 10 analyses only
if (this.analysisHistory.length > 10) {
    this.analysisHistory.shift();
}
```

---

## 4. Critical Memory Management Issues & Patterns

### 4.1 Identified Memory Growth Vectors

#### ⚠️ High Risk: Unbounded Conversation History

**Location:** `src/features/listen/summary/summaryService.js:38-42`

```javascript
addConversationTurn(speaker, text) {
    const conversationText = `${speaker.toLowerCase()}: ${text.trim()}`;
    this.conversationHistory.push(conversationText);  // ← No size limit
}
```

**Risk:**
- Long-running sessions (>1 hour) could accumulate 100+ turns
- Memory footprint: ~10KB - 100KB depending on verbosity
- **Impact:** Low-Medium (strings are GC'd when session ends)

**Mitigation Strategy:**
- Implement ring buffer with max size (e.g., 100 most recent turns)
- Suggestion: `if (this.conversationHistory.length > 100) this.conversationHistory.shift();`

#### ⚠️ Medium Risk: Audio Context Accumulation

**Location:** `src/ui/listen/audioCore/listenCapture.js:303-336`

```javascript
let audioBuffer = [];

micProcessor.onaudioprocess = (e) => {
    const inputData = e.inputBuffer.getChannelData(0);
    audioBuffer.push(...inputData);  // ← Unbounded if processing stalls

    while (audioBuffer.length >= samplesPerChunk) {
        let chunk = audioBuffer.splice(0, samplesPerChunk);
        // Process...
    }
};
```

**Risk:**
- If processing (AEC, encoding, IPC) takes longer than audio accumulation rate
- Buffer grows indefinitely
- **Impact:** Medium (could grow to several MB in extreme cases)

**Mitigation Strategy:**
- Add max buffer size check: `if (audioBuffer.length > samplesPerChunk * 10) { audioBuffer.splice(0, samplesPerChunk); }`
- Drop oldest audio to prevent memory explosion

#### ✅ Well-Managed: WASM Memory

**Location:** `src/ui/listen/audioCore/listenCapture.js:179-182`

```javascript
// Immediate deallocation after each frame
aecMod._free(micPtr.ptr);
aecMod._free(echoPtr.ptr);
aecMod._free(outPtr);
```

**Pattern:** Explicit manual memory management prevents WASM heap growth

### 4.2 Memory Optimization Patterns

#### ✅ Pattern 1: Ring Buffer for Fixed-Size History

```javascript
// System audio ring buffer
if (systemAudioBuffer.length > MAX_SYSTEM_BUFFER_SIZE) {
    systemAudioBuffer = systemAudioBuffer.slice(-MAX_SYSTEM_BUFFER_SIZE);
}
```

**Benefits:**
- Constant memory footprint regardless of session duration
- Automatic oldest-item eviction

#### ✅ Pattern 2: Database-Backed Persistence

```javascript
// Move conversation history to database, clear memory
await sttRepository.addTranscript({
    sessionId: this.currentSessionId,
    speaker: speaker,
    text: transcription.trim(),
});
```

**Benefits:**
- SQLite handles memory management
- Can query historical data without loading entire session into RAM

#### ✅ Pattern 3: Singleton Connection Pooling

```javascript
class SQLiteClient {
    constructor() {
        this.db = null;  // Single connection reused
    }
}
```

**Benefits:**
- No connection pool overhead
- better-sqlite3 uses memory-mapped I/O for efficiency

---

## 5. Data Flow Diagrams

### 5.1 High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Glass Application                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────┐      ┌──────────────────┐      ┌───────────────┐ │
│  │  Renderer Process│      │   Main Process   │      │   SQLite DB   │ │
│  │  (Chromium)      │◄────►│   (Electron)     │◄────►│  (Persistent) │ │
│  └──────────────────┘      └──────────────────┘      └───────────────┘ │
│         │                           │                                    │
│         │                           │                                    │
│  ┌──────▼──────────────────────────▼──────────┐                        │
│  │         IPC Bridge (contextBridge)          │                        │
│  │  - listenCapture API                        │                        │
│  │  - askService API                           │                        │
│  │  - settingsService API                      │                        │
│  └─────────────────────────────────────────────┘                        │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              External Services (WebSocket/HTTP)                 │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  OpenAI STT    │  Gemini STT   │  Deepgram STT  │  Anthropic  │   │
│  │  (wss://)      │  (wss://)     │  (wss://)      │  (https://) │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Memory Lifecycle During Active Session

```
Application Start
    │
    ├─→ SQLiteClient.connect()
    │   └─→ Single DB connection (persistent)
    │
    ├─→ modelStateService.initialize()
    │   └─→ Load provider settings from DB
    │
    └─→ authService.initialize()
        └─→ Load user profile

User Starts "Listen" Session
    │
    ├─→ sessionRepository.getOrCreateActive()
    │   └─→ DB: Find or create active session row
    │
    ├─→ sttService.initializeSttSessions()
    │   ├─→ WebSocket connections (2x) to STT provider
    │   ├─→ Start keep-alive heartbeat (60s interval)
    │   └─→ Schedule session renewal (20min)
    │
    ├─→ startCapture()
    │   ├─→ navigator.mediaDevices.getUserMedia()
    │   ├─→ AudioContext.createScriptProcessor()
    │   └─→ WASM AEC module load (one-time)
    │
    └─→ Memory Footprint:
        ├─→ WebSockets: ~20KB (2 connections)
        ├─→ Audio buffers: ~100KB (accumulators + ring buffers)
        ├─→ WASM module: ~500KB (loaded once)
        └─→ DB connection: ~10KB

Audio Processing (Continuous)
    │
    ├─→ Every 170ms: audioprocess event
    │   ├─→ Input: 4096 samples (32KB)
    │   ├─→ Accumulate until 2400 samples
    │   ├─→ AEC processing (14.4KB alloc/free)
    │   ├─→ Send to STT (6.4KB base64 chunk)
    │   └─→ Memory freed after send
    │
    └─→ STT Response
        ├─→ Partial: Update UI only (transient)
        └─→ Final: Write to DB (persistent)

Session Close
    │
    ├─→ stopCapture()
    │   ├─→ mediaStream.getTracks().forEach(t => t.stop())
    │   ├─→ audioContext.close()
    │   └─→ Free: ~100KB audio buffers
    │
    ├─→ sttService.closeSessions()
    │   ├─→ ws.close() for both STT sessions
    │   ├─→ clearInterval(keepAliveInterval)
    │   └─→ Free: ~20KB WebSocket buffers
    │
    ├─→ sessionRepository.end(sessionId)
    │   └─→ DB: UPDATE sessions SET ended_at = NOW()
    │
    └─→ summaryService.resetConversationHistory()
        └─→ Free: conversation history array

Application Shutdown
    │
    ├─→ listenService.closeSession()
    ├─→ sessionRepository.endAllActiveSessions()
    ├─→ ollamaService.shutdown()
    └─→ sqliteClient.close()
        └─→ DB connection closed
```

---

## 6. Key Algorithms & Logic

### 6.1 Debounced Transcription Completion

**Purpose:** Prevent fragmented transcripts from rapid partial results

**Location:** `src/features/listen/stt/sttService.js:130-150`

```javascript
debounceMyCompletion(text) {
    // Accumulate text chunks
    if (this.modelInfo?.provider === 'gemini') {
        this.myCompletionBuffer += text;  // Gemini: direct concat
    } else {
        this.myCompletionBuffer += (this.myCompletionBuffer ? ' ' : '') + text;  // Others: space-separated
    }

    // Reset timer on each chunk
    if (this.myCompletionTimer) clearTimeout(this.myCompletionTimer);

    // Flush after 2 seconds of silence
    this.myCompletionTimer = setTimeout(() => this.flushMyCompletion(), 2000);
}

flushMyCompletion() {
    const finalText = (this.myCompletionBuffer + this.myCurrentUtterance).trim();

    // Save to database
    if (this.onTranscriptionComplete) {
        this.onTranscriptionComplete('Me', finalText);
    }

    // Clear buffers
    this.myCompletionBuffer = '';
    this.myCurrentUtterance = '';
}
```

**Logic:**
1. Accumulate text chunks in buffer
2. Reset 2-second timer on each new chunk
3. After 2 seconds of no new chunks → flush to database
4. Prevents database spam from interim results

### 6.2 Token-Based Rate Limiting

**Purpose:** Prevent exceeding API rate limits (multimodal tokens)

**Location:** `src/ui/listen/audioCore/listenCapture.js:208-282`

```javascript
let tokenTracker = {
    tokens: [],  // [{timestamp, count, type}, ...]

    addTokens(count, type = 'image') {
        this.tokens.push({ timestamp: Date.now(), count, type });
        this.cleanOldTokens();  // Remove tokens older than 1 minute
    },

    trackAudioTokens() {
        const elapsedSeconds = (Date.now() - this.audioStartTime) / 1000;
        const audioTokens = Math.floor(elapsedSeconds * 16);  // 16 tokens/sec
        this.addTokens(audioTokens, 'audio');
    },

    shouldThrottle() {
        const maxTokensPerMin = parseInt(localStorage.getItem('maxTokensPerMin') || '500000');
        const throttleAtPercent = parseInt(localStorage.getItem('throttleAtPercent') || '75');

        const currentTokens = this.getTokensInLastMinute();
        const throttleThreshold = Math.floor((maxTokensPerMin * throttleAtPercent) / 100);

        return currentTokens >= throttleThreshold;  // Throttle at 75% of limit
    }
};
```

**Algorithm:**
1. Track audio tokens: 16 tokens/second (OpenAI multimodal rate)
2. Track image tokens: Variable based on resolution (85-1700 tokens)
3. Sliding window: Only count tokens from last 60 seconds
4. Throttle when reaching 75% of configured limit (default: 375K/min)

### 6.3 Session Promotion Logic

**Purpose:** Seamlessly upgrade "ask" sessions to "listen" sessions

**Location:** `src/features/common/repositories/session/sqlite.repository.js:77-109`

```javascript
function getOrCreateActive(uid, requestedType = 'ask') {
    // Find ANY active session (ended_at IS NULL)
    // Prefer 'listen' over 'ask' (CASE sorting)
    const activeSession = db.prepare(`
        SELECT id, session_type FROM sessions
        WHERE uid = ? AND ended_at IS NULL
        ORDER BY CASE session_type WHEN 'listen' THEN 1 WHEN 'ask' THEN 2 ELSE 3 END
        LIMIT 1
    `).get(uid);

    if (activeSession) {
        // Promotion: Upgrade 'ask' → 'listen' if needed
        if (activeSession.session_type === 'ask' && requestedType === 'listen') {
            updateType(activeSession.id, 'listen');
        }

        touch(activeSession.id);  // Update timestamp
        return activeSession.id;
    } else {
        return create(uid, requestedType);  // Create new session
    }
}
```

**Logic:**
1. Search for any active session (no end timestamp)
2. If found and type mismatch → promote "ask" to "listen"
3. If not found → create new session
4. **Benefit:** Maintains conversation continuity, prevents duplicate sessions

---

## 7. Recommendations & Observations

### 7.1 Strengths

✅ **Efficient WASM Memory Management:** Immediate deallocation prevents heap growth
✅ **Ring Buffer Pattern:** System audio buffer has fixed size (96KB max)
✅ **Database-Backed Persistence:** Offloads memory management to SQLite
✅ **Session Renewal Mechanism:** Prevents 30-minute timeout issues with 20-minute proactive renewal
✅ **Singleton Connection Pattern:** No connection pool overhead

### 7.2 Areas for Improvement

⚠️ **Unbounded Conversation History:**
- Current: `conversationHistory` grows indefinitely during session
- Recommendation: Implement max size (100 turns) with FIFO eviction

⚠️ **Audio Buffer Overflow Risk:**
- Current: `audioBuffer.push(...inputData)` has no upper bound
- Recommendation: Add max size check and drop oldest samples if exceeded

⚠️ **Analysis History Memory:**
- Current: Limited to 10 entries (good)
- Observation: Each entry stores full structured data (~5-10KB)
- Impact: Low risk (~100KB max)

### 7.3 Memory Usage Profile

**Idle State:**
- SQLite connection: ~10KB
- Application overhead: ~50MB (Electron/Chromium)

**Active Listen Session:**
- Audio buffers: ~100KB (bounded)
- WASM module: ~500KB (loaded once)
- WebSocket connections: ~20KB (2 connections)
- Conversation history: ~10KB → unbounded (grows ~100 bytes/turn)
- Database: Minimal memory impact (disk-backed)

**Estimated Peak Usage:** ~680KB + conversation history growth

---

## 8. Conclusion

The Glass application demonstrates **mature memory management practices** with several key patterns:

1. **Explicit Resource Cleanup:** WASM memory, WebSocket connections, and audio contexts are properly disposed
2. **Fixed-Size Buffers:** Ring buffers prevent unbounded growth in critical paths
3. **Database Offloading:** SQLite handles long-term storage, keeping in-memory footprint small
4. **Proactive Session Management:** 20-minute renewal prevents hard timeouts on 30+ minute sessions

**Primary Memory Growth Vector:** The conversation history array in `summaryService` accumulates unbounded during active sessions. For typical 30-minute sessions (~100 turns), this represents ~10KB of memory—acceptable but could be optimized with a ring buffer.

**Overall Assessment:** The memory management architecture is **production-ready** with minor optimization opportunities identified above.

---

## Appendix A: File Reference Index

**Core Memory Management Files:**
- `src/features/common/services/sqliteClient.js` - Database connection singleton
- `src/features/common/services/databaseInitializer.js` - DB lifecycle management
- `src/ui/listen/audioCore/listenCapture.js` - Audio buffer management
- `src/features/listen/stt/sttService.js` - STT session lifecycle & renewal
- `src/features/listen/summary/summaryService.js` - Conversation history accumulation
- `src/features/common/repositories/session/sqlite.repository.js` - Session cleanup logic

**Database Schema:**
- `src/features/common/config/schema.js` - Complete table definitions

**AI Provider Implementations:**
- `src/features/common/ai/providers/openai.js` - OpenAI WebSocket STT
- `src/features/common/ai/providers/gemini.js` - Gemini Live API
- `src/features/common/ai/providers/deepgram.js` - Deepgram WebSocket STT
- `src/features/common/ai/providers/anthropic.js` - Anthropic Messages API

---

**Analysis Complete**
Generated by Claude Code on branch `claude/investigate-memory-management-li18C`
