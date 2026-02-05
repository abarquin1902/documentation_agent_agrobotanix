# Agrobotanix Agent - Visual Architecture

## High-Level Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         WHATSAPP MESSAGE                            │
│                    (Text, Audio, or Image)                          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         KOMMO WEBHOOK                               │
│                    Receives incoming message                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DUPLICATE CHECK (Redis)                          │
│          message_id_{id} with 5-minute TTL                          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ├──► Duplicate? → STOP
                             │
                             ▼ New Message
┌─────────────────────────────────────────────────────────────────────┐
│                        MESSAGE TYPE CHECK                           │
│                                                                     │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│   │    AUDIO     │    │     TEXT     │    │    IMAGE     │          │
│   │  (Voice msg) │    │   (Direct)   │    │  (Future)    │          │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
│          │                   │                   │                  │
│          ▼                   │                   ▼                  │
│   ┌──────────────┐           │            ┌──────────────┐          │
│   │   WHISPER    │           │            │ CLAUDE VISION│          │
│   │Transcription │           │            │ (In Progress)│          │
│   └──────┬───────┘           │            └──────────────┘          │
│          │                   │                                      │
│          └───────────────────┴──────────────────────────────────────┤
│                              │                                      │
│                              ▼                                      │
│                      ┌──────────────┐                               │
│                      │  TEXT READY  │                               │
│                      └──────┬───────┘                               │
└─────────────────────────────┼───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INTENT CLASSIFIER (OpenAI)                       │
│                    Uses function calling to categorize              │
│                                                                     │
│   Possible intents:                                                 │
│   • saludo (greeting)                                               │
│   • producto_only (product inquiry)                                 │
│   • cultivo_producto (crop + product) (needs knowledge base)        │
│   • cultivo_only (crop consultation) (needs knowledge base)         │
│   • compra (purchase)                                               │
│   • seguimiento (order tracking)                                    │
│   • atencion_especializada (human needed)                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AGENT ROUTER (Switch)                          │
│             Directs to appropriate specialized agent                │
└─────┬────────┬────────┬────────┬────────┬────────┬──────────────────┘
      │        │        │        │        │        │
      │        │        │        │        │        │
      ▼        ▼        ▼        ▼        ▼        ▼
   ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
   │ A1 │  │ A2 │  │ A3 │  │ A4 │  │ A5 │  │ A6 │
   └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘
     │       │       │       │       │       │
     │       │       │       │       │       │
     └───────┴───────┴───────┴───────┴───────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AGENT PROCESSING LAYER                         │
│                                                                     │
│         ┌─────────────────┐         ┌─────────────────┐             │
│         │   TOOL CALL     │◄───────►│     QDRANT      │             │
│         │     AGENT       │         │   (Vector DB)   │             │
│         │  (OpenAI GPT-4o)│         │   RAG Search    │             │
│         └────────┬────────┘         └─────────────────┘             │
│                  │                                                  │
│                  ▼                                                  │
│         ┌─────────────────┐         ┌─────────────────┐             │
│         │     REDIS       │         │    CONTEXT      │             │
│         │  Chat History   │◄───────►│   INJECTION     │             │
│         │   + Metadata    │         │                 │             │
│         └─────────────────┘         └─────────────────┘             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    RESPONSE GENERATION                              │
│   • Synthesize agent output                                         │
│   • Extract metadata (crop, problem, products)                      │
│   • Format for WhatsApp                                             │
│   • Save to Redis for next conversation                             │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INTEGRATION LAYER                                │
│                                                                     │
│   If purchase → Shopify API (create order)                          │
│   If tracking → Shopify API (get order status)                      │
│   Send message → Kommo API (WhatsApp)                               │
│   Save metadata → Redis (persistent storage)                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    WHATSAPP MESSAGE SENT                            │
│                   User receives response                            │
└─────────────────────────────────────────────────────────────────────┘
```

## Agents Explained

```
A1: GREETING AGENT
   Purpose: Welcome new users, gather context
   Tools: None (conversational)
   Output: Stores crop type + initial problem in Redis if the user provides it at the beginning

A2: PRODUCT AGENT
   Purpose: Direct product inquiries
   Tools: Qdrant (product search)
   Output: Product recommendations + specs

A3: CROP + PRODUCT AGENT (Most Complex)
   Purpose: Disease diagnosis + product recommendation
   Tools: Qdrant (disease DB + products), Redis (metadata)
   Output: Diagnosis + personalized treatment plan

A4: CROP-ONLY AGENT
   Purpose: Agricultural consultation without sales
   Tools: Qdrant (knowledge base)
   Output: Educational advice

A5: PURCHASE AGENT
   Purpose: Order creation and payment
   Tools: Shopify API, Redis (cart management)
   Output: Order number + payment link

A6: ORDER TRACKING AGENT
   Purpose: Post-purchase support
   Tools: Shopify API (order lookup)
   Output: Delivery status + tracking number
```

## Data Flow Architecture

```
INPUT LAYER
    │
    ├─ Text Message
    ├─ Voice Message (→ Whisper → Text)
    └─ Image (Future: → CLAUDE VISION → Description)
    │
    ▼
PREPROCESSING LAYER
    │
    ├─ Duplicate Detection (Redis: message_id)
    ├─ Audio Transcription (Whisper API)
    └─ Text Extraction
    │
    ▼
CLASSIFICATION LAYER
    │
    ├─ Intent Detection (OpenAI Function Calling)
    ├─ Entity Extraction (crop, problem, urgency)
    └─ Sentiment Analysis
    │
    ▼
ROUTING LAYER
    │
    ├─ Select Specialized Agent (Switch/Router)
    ├─ Load Conversation History (Redis)
    └─ Fetch User Metadata (Redis)
    │
    ▼
EXECUTION LAYER
    │
    ├─ Agent Reasoning (Tool call + GPT-4)
    ├─ Knowledge Retrieval (Qdrant RAG)
    ├─ Context Integration (Redis Memory)
    └─ External API Calls (Shopify if needed)
    │
    ▼
RESPONSE LAYER
    │
    ├─ Synthesize Output
    ├─ Extract New Metadata (crop, products, stage)
    ├─ Format for WhatsApp
    │
    ▼
PERSISTENCE LAYER
    │
    ├─ Save Chat History (Redis)
    ├─ Update User Metadata (Redis)
    ├─ Create Order (Shopify if purchase)
    │
    ▼
OUTPUT LAYER
    │
    └─ Send via Kommo API → WhatsApp
```

## Redis Data Structure

```
Redis Database Structure:
│
├── chat_history:{contact_id}
│   │
│   ├── Type: List
│   ├── Content: Last 12 messages (alternating user/assistant)
│   ├── TTL: 7 days
│   └── Example:
│       [
│         {"role": "user", "content": "Tengo plagas en mi maíz"},
│         {"role": "assistant", "content": "¿Qué tipo de plagas..."},
│         ...
│       ]
│
├── metadata:{contact_id}
│   │
│   ├── Type: Hash
│   ├── Content: User context and problem info
│   ├── TTL: 7 days
│   └── Fields:
│       • cultivo: "maíz"
│       • problema: "plagas"
│       • etapa: "floración"
│       • productos_recomendados: ["prod_1", "prod_2"]
│       • ubicacion: "Michoacán"
│
├── message_dedup:message_{id}
    │
    ├── Type: String
    ├── Content: "1" (flag)
    ├── TTL: 5 minutes
    └── Purpose: Prevent duplicate webhook processing
```

## Qdrant Vector Database

```
Collection: "agrobotanix_products"
│
├── Vector Configuration
│   ├── Dimensions: 1024
│   ├── Similarity: Cosine
│   └── Embedding Model: OpenAI text-embedding-large-3
│
├── Document Types
│   ├── Products (agricultural inputs)
│   ├── Diseases (pest/disease encyclopedia)
│   ├── Treatments (application guidelines)
    ├── Dosis for specified crop
│   └── Best Practices (cultivation tips)
│
├── Metadata Schema
│   ├── product_name: string
│   ├── category: ["fungicida", "insecticida", "fertilizante", ...]
│   ├── provided_crop: string[]
│   ├── dosage_info: string
│   ├── application_method: string
│   ├── price_mxn: number
│   └── technical_sheet_url: string
│
└── Search Strategy
    ├── 1. Semantic similarity search (user query → vectors)
    ├── 2. Metadata filtering (crop type, category)
    ├── 3. Score threshold: > 0.7
    ├── 4. Top-k retrieval: k=3-5 results
    └── 5. Post-retrieval validation (product availability)
```

## Edge Cases Handled

```
PROBLEM: Duplicate Webhooks
SOLUTION: Redis deduplication
IMPLEMENTATION:
    ├─ Store message_id in Redis
    ├─ TTL: 5 minutes
    ├─ Check before processing
    └─ Skip if exists

PROBLEM: RAG Hallucination (recommending non-existent products)
SOLUTION: Post-retrieval validation
IMPLEMENTATION:
    ├─ RAG returns product candidates
    ├─ Validate against Shopify inventory
    ├─ Remove out-of-stock items
    └─ Fallback to human if no matches

PROBLEM: Context Loss Across Days
SOLUTION: Persistent Redis memory
IMPLEMENTATION:
    ├─ Session key: contact_id (expires at 7 days)
    ├─ Store metadata after each conversation
    ├─ Load metadata before agent execution
    └─ Progressive accumulation of context

PROBLEM: Multi-turn Coherence
SOLUTION: Chat history injection
IMPLEMENTATION:
    ├─ Load last 10 messages from Redis
    ├─ Inject into agent prompt
    ├─ Maintain conversation thread
    └─ Reference previous recommendations
```

## Key Performance Optimizations

```
1. CACHING STRATEGY
   ├─ Frequent product queries cached in Redis (1 hour TTL)
   ├─ Common disease searches cached (24 hours TTL)
   └─ Reduces Qdrant calls by ~40%

2. BATCH EMBEDDING
   ├─ Product catalog embedded in batches (nightly)
   ├─ Not real-time (reduces API costs)
   └─ ~$5/month vs. $50/month for real-time

3. CONNECTION POOLING
   ├─ Redis connection pool (max 10 connections)
   ├─ Prevents connection overhead
   └─ Reduces latency by ~30%

4. SMART CONTEXT WINDOW
   ├─ Only last 10-12 messages in context
   ├─ Metadata summary instead of full history
   └─ Reduces token usage by ~60%
```