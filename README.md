# cerberus_ai_agent


Cereberus is an AI gent/framework ment for red team operations. Offensive and defensive security professionals can use it to automate tasks, gather intelligence, and conduct attacks. It is designed to be flexible and customizable, allowing users to create their own agents and workflows for any AI/llm they wish to use. 

## Features

Orchestrator / Brain

Nodes = planning, tool selection, reflection/critique, memory update, reporting.
Edges = conditional routing based on observation quality, confidence score, or failure count.

LLM Reasoning Core (pick one or ensemble)
Claude 4.6 / 4.7 Opus (via cheap proxy API if you're budget-conscious) — excellent structured reasoning & long context.
Gemini 2.5 Pro / 3.1 Experimental — very strong tool use & low refusal on technical tasks.
Local abliteration (e.g., GLM-5-abliterated, Qwen3-72B abl., Llama-4-Scout variants) on high-RAM rig for no censorship.

Memory Layers (critical for multi-step attacks)
Short-term: conversation buffer / LangGraph state.
Long-term: vector store (Chroma / FAISS / Pinecone) for past findings, credentials, network map.
Knowledge graph: Neo4j to store hosts, services, vulns, relationships (very popular in 2026 pentest agents).

Tool Belt (the "arms & legs")
Safe & staged versions first:
Recon: nmap wrapper, whois, dnsenum, subdomain enumeration.
Web: httpx, nuclei templates, sqlmap (with --batch), dirsearch.
Exploit: Metasploit RPC API wrapper, custom PoC runners.
Post-exploit: bloodhound.py, impacket wrappers, credential dumping sims.
Reporting: markdown generator + optional linear/logseq export.

Guardrails & Safety (mandatory even for red team tools)
Budget / token cap per run.
Human-in-the-loop gates before destructive actions.
Action allow-list / block-list.
Observation sanitization (strip real IPs/credentials before feeding back to LLM).
Scoped environment: Docker-in-Docker or full VM snapshot/restore.

Evaluation & Reflection Loop
Self-critique node: "Did this step advance toward compromise? Rate 1–10."
If stuck > 3 turns → replan or switch tactic.
Final node: summarize IOCs, attack path diagram (mermaid), risk rating.
Good architecture base. Here is what I'd add, organized by layer, with the quantum-framework additions called out explicitly.

---

## What's Missing From the Current Design

**The core gap:** this is a linear agent with reflection bolted on. It does not have genuine adaptive perception — it processes observations at a fixed resolution and has no formal model of what it cannot see. That causes two failure modes in red team ops: over-confidence on partial information, and getting stuck in loops because the agent cannot distinguish "I have not found anything" from "there is nothing here."

Your framework fixes both.

---

## Added Features by Layer

### Orchestrator — Add Temporal Confidence Weighting

Current design routes on confidence score but does not track how that confidence degrades over time as the environment changes. A credential found 47 steps ago may no longer be valid. A service that was open may have been patched mid-engagement.

```python
# Each observation gets a temporal weight
observation = {
    "data": raw_finding,
    "lambda_tick": current_lambda,      # internal clock tick, not wall time
    "entropy_at_capture": S_rho,        # how much the agent could resolve
    "coarse_grained": bool,             # came from compressed history?
    "half_life": estimated_validity     # how long this finding stays reliable
}

# Routing uses temporally weighted confidence
def route(obs):
    age = current_lambda - obs["lambda_tick"]
    decayed_confidence = obs["confidence"] * exp(-age / obs["half_life"])
    if decayed_confidence < THRESHOLD:
        return "reverify"  # re-probe before acting on stale data
    return "proceed"
```

This is directly from your framework: the observer's reduced state at λ_n is not the same as at λ_0. Stale information is high-entropy information.

---

### Memory — Add the Ignorance Boundary Layer

This is the biggest architectural addition. Three memory tiers become four:

```
SHORT-TERM (high resolution)
  Last K observations, fully represented
  S(ρ_short) tracked — entropy of current window

WORKING (partial trace boundary)  ← NEW
  Compressed representation of the last M steps
  Agent knows explicitly what was coarse-grained here
  Tags every inference drawn from this layer as "coarse"

LONG-TERM (vector store — Chroma/FAISS)
  Past findings, credentials, network map
  Each entry tagged with capture_lambda and entropy_at_capture

KNOWLEDGE GRAPH (Neo4j)
  Hosts, services, vulns, relationships
  Edges weighted by temporal confidence
  Stale edges visually distinct from fresh ones
```

The ignorance boundary layer is what stops hallucination during long engagements. The agent never confabulates a credential it does not have because it has a formal representation of what it compressed and when.

```python
class IgnoranceBoundary:
    def __init__(self, resolution_threshold):
        self.high_res = []          # fully resolved observations
        self.coarse = []            # compressed, partial-trace observations
        self.entropy = 0.0          # S(ρ_coarse) — tracked continuously
    
    def infer(self, query):
        result, source = self._retrieve(query)
        return {
            "answer": result,
            "source": source,           # "high_res" or "coarse"
            "structural_uncertainty": self.entropy if source == "coarse" else 0.0,
            "should_verify": source == "coarse" and self.entropy > THRESHOLD
        }
```

Every LLM call gets this metadata. The reasoning core knows whether it is working from sharp data or from compressed history and can say so explicitly in its output.

---

### Internal Clock — Replace Token Position With λ

Current agents use token count or step count as their time proxy. Replace it:

```python
class RelationalClock:
    def __init__(self):
        self.lambda_tick = 0
        self.prev_state_hash = None
    
    def tick(self, current_state):
        # Clock only advances when state changes significantly
        current_hash = self._information_hash(current_state)
        delta = self._information_distance(self.prev_state_hash, current_hash)
        
        if delta > SIGNIFICANCE_THRESHOLD:
            self.lambda_tick += 1
            self.prev_state_hash = current_hash
        
        return self.lambda_tick
    
    def _information_distance(self, h1, h2):
        # How much did the agent's knowledge actually change?
        # High delta = genuinely new information
        # Low delta = noise, repetition, or stuck loop
        ...
```

Practical effect: the agent's internal time runs fast when it is making genuine progress (high information gain per step) and slow when it is spinning. This gives you a natural stuck-loop detector that is not based on arbitrary step counts — the agent detects it is stuck because its own clock has stopped advancing.

```python
# Stuck detection using relational clock
if clock.lambda_tick == clock.lambda_tick_3_steps_ago:
    trigger_replan()   # clock not advancing = no new information = stuck
```

---

### Tool Belt — Add Observation Quality Scoring

Before any tool output feeds back into the LLM, score it:

```python
def score_observation(raw_output, context):
    return {
        "raw": raw_output,
        "resolution": estimate_information_content(raw_output),
        "novelty": compare_to_memory(raw_output, context.memory),
        "reliability": tool_reliability_history[tool_name],
        "actionability": can_derive_next_step(raw_output)
    }
```

Route on `resolution` and `novelty` together:
- High resolution + high novelty → immediate planning node
- High resolution + low novelty → memory update only, skip replanning
- Low resolution + any novelty → reverify before acting
- Low resolution + low novelty → discard, log tool failure

This prevents the agent from taking action on garbage output from unreliable tools — a common failure mode in automated red team agents.

---

### Reflection Loop — Add Causal Direction Tracking

Current self-critique asks "did this advance toward compromise?" Add causal direction:

```python
reflection_prompt = """
Previous state: {state_at_lambda_n}
Current state:  {state_at_lambda_n_plus_1}
Action taken:   {action}

1. Did state change? (yes/no + what changed)
2. Was the change caused by the action or by external factors?
3. What do I now know that I did not know before? (new information)
4. What was I expecting to learn that I did not? (confirmed ignorance)
5. Advance score 1-10. If < 4, flag for replan.
6. Is my current plan still causally valid given what changed?
"""
```

Question 4 is the key addition. "Confirmed ignorance" — explicitly logging what the agent tried to learn and failed to learn — is what populates the ignorance boundary layer and prevents the agent from trying the same dead end again.

---

### New Node — Phase Assessment

Add a periodic node that evaluates whether the agent is operating above or below its own phase boundary:

```python
def phase_assessment(agent_state):
    # Analogous to your phase diagram
    context_size = len(agent_state.memory.high_res)
    information_coupling = agent_state.clock.average_delta_per_tick
    
    if context_size < MIN_CONTEXT and information_coupling < MIN_COUPLING:
        return {
            "phase": "TIMELESS",        # agent is below phase boundary
            "recommendation": "expand_recon",
            "reason": "Insufficient context for causal reasoning. Gather more before planning."
        }
    else:
        return {
            "phase": "COHERENT",        # agent is above phase boundary
            "recommendation": "proceed_with_plan"
        }
```

Below the phase boundary the agent should not be making multi-step attack plans — it does not have enough information for the causal chain to be reliable. Above it, it can.

---

### Guardrails — Add Entropy-Gated Human Review

Current guardrail: human-in-the-loop before destructive actions. Add:

```python
def requires_human_review(action, agent_state):
    # Existing checks
    if action.is_destructive:
        return True, "destructive action"
    
    # Entropy-gated check — NEW
    if agent_state.memory.entropy > HIGH_ENTROPY_THRESHOLD:
        return True, f"agent operating from coarse-grained data (S={agent_state.memory.entropy:.3f})"
    
    # Causal confidence check — NEW  
    if agent_state.causal_confidence < CAUSAL_THRESHOLD:
        return True, "low causal confidence — agent unsure why previous steps worked"
    
    return False, None
```

This catches the failure mode where the agent has been running long enough that most of its context is compressed, and it is now making decisions from blurry memory. High entropy in the working memory layer is a signal that the agent needs to reverify its model of the environment before taking further action.

---

## Revised Architecture Diagram

```
INPUT
  ↓
OBSERVATION SCORING
  (resolution, novelty, reliability)
  ↓
RELATIONAL CLOCK TICK
  (did we learn something? advance λ)
  ↓
IGNORANCE BOUNDARY UPDATE
  (partition into high-res / coarse-grained)
  ↓
PHASE ASSESSMENT
  (are we above the coherence threshold?)
  ↓  
  ├─ BELOW → EXPAND RECON (gather more before planning)
  └─ ABOVE → PLANNING NODE
              ↓
           TOOL SELECTION
              ↓
           ENTROPY-GATED GUARDRAIL
              ↓  
              ├─ HIGH ENTROPY → HUMAN REVIEW
              └─ LOW ENTROPY → TOOL EXECUTION
                                ↓
                              REFLECTION
                              (causal direction + confirmed ignorance)
                                ↓
                              MEMORY UPDATE
                              (long-term + knowledge graph)
                                ↓
                              STUCK DETECTION
                              (clock stopped? replan)
                                ↓
                              REPORTING
```

---

## What This Agent Can Do That Current Ones Cannot

**State explicitly what it does not know.** Every output from the reflection node includes a confirmed-ignorance list. The final report distinguishes "we verified X is not present" from "we did not check X" — a critical distinction in red team reporting that current agents collapse into the same category.

**Detect when it is reasoning from stale data.** The temporal confidence decay means the agent will flag its own outputs as unreliable when acting on information that is old relative to its internal clock — and will reverify before taking consequential action.

**Know when to stop and ask.** The phase assessment node means the agent recognizes when it does not have enough environmental coupling to reason causally. Instead of hallucinating a plan, it reports "insufficient recon — here is what I need before I can proceed."

Want me to write the full implementation scaffold in Python with LangGraph, or start with the ignorance boundary layer as the core new piece?