# MucAtlas — Municipal Knowledge Agent

Competition submission: Munich Innovation Challenge 2026, Challenge 4.

Five-agent RAG knowledge assistant for Munich city administration (Kreisverwaltungsreferat). Connects fragmented internal knowledge sources, validates legal document currency, detects source contradictions, and delivers precise, cited answers to caseworkers in seconds.

> **Note**: This repository contains architecture and design documentation only. MucAtlas was submitted as a competition design proposal; implementation code is not included.

---

## The Problem

Munich city administration handles thousands of regulatory queries per day. Caseworkers must manually navigate fragmented internal sources — file servers, SharePoint wikis, email archives, and scanned PDFs — to find current, authoritative answers. This creates three compounding problems:

1. **Speed**: Cross-referencing multiple sources for a single query takes significant time
2. **Accuracy**: Outdated documents remain in circulation without automated validation
3. **Consistency**: Different caseworkers may reach different conclusions from the same sources

## Architecture

MucAtlas is structured in four layers, with five specialised agents in the Agent Layer.

### Layer 1 — Knowledge Layer
Ingests all internal sources on a nightly schedule: Windows file servers (SMB), SharePoint wiki, Exchange email archives, and scanned PDFs (via multimodal model).

### Layer 2 — Retrieval Layer
Hybrid search combining BM25 keyword search and ANN semantic vector search simultaneously. Returns ranked document chunks by relevance score. A cross-encoder reranker is provisioned as a sidecar for further precision.

**Embedding model**: Qwen3-Embedding 4B (32k token context window)

### Layer 3 — Agent Layer

Five agents run in sequence. The system re-queries retrieval if confidence is low.

```
Research Agent → Prüf-Agent → Conflict Agent → Synthesis Agent → Risk Agent
```

| Agent | Role |
|---|---|
| **Research Agent** | Searches all connected sources via hybrid retrieval; returns top-ranked document chunks |
| **Prüf-Agent** | Validates document currency and quality; flags docs older than a configurable threshold or from periods of known legal change; links directly to Bundesrecht.de and Bayerisches Landesrecht |
| **Conflict Agent** | Detects contradictions between retrieved sources; surfaces conflicts explicitly to the caseworker rather than silently favouring one version |
| **Synthesis Agent** | Generates a structured answer with full inline citations — source name, origin system, date, and specific passage; caseworker can click through to the original document |
| **Risk Agent** | Flags low-confidence answers and explicitly recommends human verification; prevents false certainty when underlying evidence is weak |

A sixth Validation Agent (automated output quality control / DeepEval-style) is deliberately deferred — to be evaluated during the pilot based on real-world performance data.

### Layer 4 — Governance Layer
Full source citations, confidence indicators, audit log of all queries and responses, GDPR-compliant design with no personal data in the index. LDAP/AD authentication. Nightly re-index cadence.

## Model Stack — Privatemode (Edgeless Systems)

All models run on [Privatemode](https://edgeless.systems/privatemode/), a confidential computing platform under German/EU law. No data leaves the city's infrastructure. All inference runs in encrypted enclaves.

| Model | Role |
|---|---|
| gpt-oss-120b | Primary reasoning and Synthesis Agent (128k context, Anthropic-compatible API) |
| Qwen3-Embedding 4B | Vector embeddings for the Retrieval Layer |
| Gemma 3 27B | Multimodal document handling — processes scanned PDFs and images directly |
| Qwen3-Coder-30b-a3b | Structured data extraction from complex documents (optional) |
| Whisper-large-v3 | Voice input transcription — optional accessibility feature |

## MucGPT Integration

MucAtlas is architected as a modular MCP (Model Context Protocol) server from day one.

- **Phase 1**: Standalone browser application, no client installation required, LDAP/AD authentication
- **Phase 2**: Exposed as an MCP tool within MucGPT — caseworkers already using MucGPT access MucAtlas without leaving their existing interface; no changes to MucGPT core required

## Key Differentiators

**Prüf-Agent document validation**: Automatically flags outdated sources and links to current legal references. Not present in any known municipal solution.

**Conflict detection**: Surfaces contradictions between sources explicitly rather than silently synthesising a potentially incorrect answer. Critical in legal/administrative contexts.

**Risk flagging**: Low-confidence answers are labelled and human review recommended. The system never generates false certainty.

**Full data sovereignty**: Confidential Computing via Privatemode. All processing in encrypted enclaves. No external API calls. GDPR-compliant by architecture.

**MucGPT-native via MCP**: No parallel infrastructure. Plugs into the city's existing AI strategy.

## Open Assumptions (to validate in scoping)

- Primary data sources: Windows file servers (SMB), SharePoint wiki, Exchange email archives, scanned PDFs
- No structured Bavarian legal change calendar exists in machine-readable form — to be built manually for pilot legal domains
- Browser-based access is available from all relevant KVR workstations without firewall constraints
- Nightly re-index cadence is sufficient — adjustable per source if needed
- Document-level access permissions sufficient for prototype; paragraph-level PII controls evaluated in co-creation phase
- MCP integration follows standard protocol as per publicly available MucGPT documentation
- Reranker model (cross-encoder) may need to be added as a sidecar — not currently in Privatemode model catalogue

## Co-Creation Roadmap

| Phase | Content |
|---|---|
| Scoping | Joint review of KVR data sources with caseworkers and IT-Referat; requirements workshops; define pilot legal domains |
| MVP Build | Implement Knowledge Layer and Agent Layer; test with real case questions from KVR day-to-day work |
| Integration | Connect to MucGPT via MCP protocol; extended test operation with KVR staff |
| Handover | Documentation, final evaluation, joint planning of scaling to further departments |

---

*Submitted to Munich Innovation Challenge 2026, Challenge 4 — Kayvon Salari*  
*[kayvonsalari.github.io](https://kayvonsalari.github.io)*
