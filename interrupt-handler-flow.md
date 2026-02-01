# Interrupt Handler Flow Diagram

This document visualizes the flow of the interrupt handler system used in the voice agent application.

## Overview

The interrupt handler manages user interruptions during agent speech, distinguishing between meaningful interruptions and backchannel responses (like "yeah", "ok", "hmm").

## Architecture Overview

```mermaid
graph TB
    subgraph "System Components"
        IC[InterruptController]
        SH[Session Handlers]
        AG[Voice Agent]
    end
    
    subgraph "Configuration"
        IW[Ignore Words<br/>yeah, ok, hmm, uh]
        IRW[Interrupt Words<br/>stop, wait, no, cancel]
        GP[Grace Period<br/>250ms]
    end
    
    subgraph "State"
        AS[agent_is_speaking<br/>TRUE/FALSE]
        PI[pending_interrupt<br/>TRUE/FALSE]
        PS[pending_since<br/>timestamp]
    end
    
    IW --> IC
    IRW --> IC
    GP --> IC
    IC --> AS
    IC --> PI
    IC --> PS
    SH --> IC
    AG --> SH
```

## Event Flow

```mermaid
sequenceDiagram
    participant User
    participant VAD as Voice Activity<br/>Detection
    participant STT as Speech-to-Text
    participant IC as Interrupt<br/>Controller
    participant Agent
    
    Agent->>IC: State Changed (Speaking)
    IC->>IC: agent_is_speaking = TRUE
    
    User->>VAD: Starts Speaking
    VAD->>IC: User Speech Detected
    IC->>IC: pending_interrupt = TRUE<br/>pending_since = now()
    
    User->>STT: Continues Speaking
    STT->>IC: Transcript (Final)
    
    IC->>IC: Evaluate should_interrupt()
    
    alt Explicit Interrupt Word
        IC->>Agent: INTERRUPT ❌
    else Grace Period Expired
        IC->>Agent: INTERRUPT ❌
    else Backchannel Only
        IC->>IC: IGNORE ✓
    else Mixed/Semantic Input
        IC->>Agent: INTERRUPT ❌
    else No Tokens
        IC->>IC: IGNORE ✓
    end
```

## Interrupt Decision Logic

```mermaid
graph TD
    Start[User Input Transcribed<br/>Final Transcript] --> Tokenize[Tokenize Transcript<br/>Extract Words]
    
    Tokenize --> Check1{Step 1<br/>Contains Interrupt Words?<br/>stop, wait, no, cancel, hold}
    
    Check1 -->|YES| Int1[🔴 INTERRUPT<br/>Priority: HIGHEST]
    
    Check1 -->|NO| Check2{Step 2<br/>Grace Period Expired?<br/>> 250ms since speech start}
    
    Check2 -->|YES| Int2[🔴 INTERRUPT<br/>Priority: HIGH]
    
    Check2 -->|NO| Check3{Step 3<br/>Agent Speaking &<br/>All Tokens are Ignore Words?<br/>yeah, ok, hmm, uh}
    
    Check3 -->|YES| Ignore1[🟢 IGNORE<br/>Backchannel Response]
    
    Check3 -->|NO| Check4{Step 4<br/>Agent Speaking &<br/>Has Meaningful Tokens?}
    
    Check4 -->|YES| Int3[🔴 INTERRUPT<br/>Priority: NORMAL]
    
    Check4 -->|NO| Ignore2[🟢 IGNORE<br/>Empty/No Content]
    
    Int1 --> Execute[Execute Interruption<br/>session.interrupt]
    Int2 --> Execute
    Int3 --> Execute
    
    Ignore1 --> Continue[Continue<br/>Agent Speaking]
    Ignore2 --> Continue
    
    style Int1 fill:#ff4444,stroke:#cc0000,color:#fff
    style Int2 fill:#ff6666,stroke:#cc0000,color:#fff
    style Int3 fill:#ff8888,stroke:#cc0000,color:#fff
    style Ignore1 fill:#44ff44,stroke:#00cc00,color:#000
    style Ignore2 fill:#66ff66,stroke:#00cc00,color:#000
    style Check1 fill:#ffeb3b,stroke:#f57f17
    style Check2 fill:#ffeb3b,stroke:#f57f17
    style Check3 fill:#ffeb3b,stroke:#f57f17
    style Check4 fill:#ffeb3b,stroke:#f57f17
```

## State Machine

```mermaid
stateDiagram-v2
    [*] --> AgentIdle
    
    AgentIdle --> AgentSpeaking: agent_state_changed(speaking)
    AgentSpeaking --> AgentIdle: agent_state_changed(not speaking)
    
    AgentIdle --> UserSpeaking: user_speech_detected
    UserSpeaking --> AgentIdle: transcript processed
    
    AgentSpeaking --> PendingInterrupt: user_speech_detected
    
    state PendingInterrupt {
        [*] --> EvaluatingTranscript
        EvaluatingTranscript --> CheckingInterruptWords
        CheckingInterruptWords --> Interrupt: Explicit words found
        CheckingInterruptWords --> CheckingGracePeriod: Not found
        CheckingGracePeriod --> Interrupt: > 250ms elapsed
        CheckingGracePeriod --> CheckingBackchannel: Within grace period
        CheckingBackchannel --> Ignore: All ignore words
        CheckingBackchannel --> CheckingMixed: Has other words
        CheckingMixed --> Interrupt: Meaningful tokens
        CheckingMixed --> Ignore: No meaningful tokens
    }
    
    PendingInterrupt --> AgentIdle: Interrupt → Agent stops
    PendingInterrupt --> AgentSpeaking: Ignore → Agent continues
    
    note right of AgentSpeaking
        agent_is_speaking = TRUE
        Can receive interruptions
    end note
    
    note right of AgentIdle
        agent_is_speaking = FALSE
        User input commits normally
    end note
    
    note right of PendingInterrupt
        pending_interrupt = TRUE
        pending_since = timestamp
        Evaluating transcript
    end note
```

## Component Breakdown

### 1. System Components

| Component | Responsibility |
|-----------|---------------|
| **InterruptController** | Core logic for evaluating interruptions |
| **Session Handlers** | Connects events to interrupt controller |
| **Voice Agent** | Main agent orchestrating the conversation |

### 2. Configuration Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| **ignore_words** | `yeah`, `ok`, `okay`, `hmm`, `uh`, `uh-huh`, `aha`, `right`, `correct` | Backchannel words that don't interrupt |
| **interrupt_words** | `stop`, `wait`, `no`, `cancel`, `hold` | Explicit commands that force interruption |
| **grace_ms** | `250ms` | Time window before treating speech as interruption |

### 3. State Variables

| Variable | Type | Description |
|----------|------|-------------|
| **agent_is_speaking** | Boolean | Tracks if agent is currently speaking |
| **pending_interrupt** | Boolean | Marks when user speaks during agent speech |
| **pending_since** | Timestamp | Records when user started speaking |

### 4. Decision Priority Levels

The system evaluates interruptions in this cascading order:

```
Priority 1: EXPLICIT INTERRUPT WORDS (stop, wait, no, cancel, hold)
    ↓ If not found
Priority 2: GRACE PERIOD CHECK (> 250ms elapsed)
    ↓ If within grace period
Priority 3: BACKCHANNEL DETECTION (all words are yeah/ok/hmm)
    ↓ If has other words
Priority 4: MIXED/SEMANTIC INPUT (meaningful conversation)
    ↓ If no meaningful tokens
Result: IGNORE (empty or silence)
```

### 5. Event Processing Flow

**Step 1**: Agent State Monitoring
- Listens to `agent_state_changed` and `speech_created` events
- Updates `agent_is_speaking` flag

**Step 2**: User Speech Detection
- Captures `user_input_transcribed` events
- Sets `pending_interrupt` if agent is speaking

**Step 3**: Transcript Evaluation
- Tokenizes the user's transcript
- Applies decision logic cascade

**Step 4**: Action Execution
- **Interrupt**: Calls `session.interrupt()` to stop agent
- **Ignore**: Allows agent to continue speaking

## Visual Summary

### Interrupt vs Ignore Scenarios

```mermaid
graph LR
    subgraph "🔴 INTERRUPT Scenarios"
        I1[User says: STOP]
        I2[User speaks > 250ms]
        I3[User: Tell me about Paris]
    end
    
    subgraph "🟢 IGNORE Scenarios"
        G1[User says: yeah]
        G2[User says: ok hmm]
        G3[User: silence]
    end
    
    style I1 fill:#ff4444,color:#fff
    style I2 fill:#ff4444,color:#fff
    style I3 fill:#ff4444,color:#fff
    style G1 fill:#44ff44,color:#000
    style G2 fill:#44ff44,color:#000
    style G3 fill:#44ff44,color:#000
```

## Color Legend

- 🔴 **Red**: Interrupt actions - agent speech is canceled
- 🟢 **Green**: Ignore actions - agent continues speaking
- 🟡 **Yellow**: Decision points - conditional logic
- 🔵 **Blue**: System components and state

## Implementation Files

- **interrupt_controller.py**: Core interrupt logic and decision-making
- **session_handlers.py**: Event handler registration and coordination
- **history_agent.py**: Main agent implementation using the interrupt controller

## Usage Example

```python
# Initialize the interrupt controller
interrupt_controller = InterruptController(
    ignore_words={"yeah", "ok", "hmm"},
    interrupt_words={"stop", "wait", "no"},
    grace_ms=250
)

# Register event handlers
register_interrupt_handlers(session, interrupt_controller, logger)
```
