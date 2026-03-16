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