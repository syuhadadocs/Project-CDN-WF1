# Project CDN-WF1 — Blueprint Final (GitHub-Ready)

Versi: **Hybrid v3**  
Tanggal: **2026-01-13**  

## 1) Ringkasan
Dokumen ini adalah blueprint final untuk **CDN WF1**: orkestrasi pesan multi-channel → ticket thread-centric → AI (Gemini) → outbox delivery → (opsional) order/payment/inventory.  
Dokumen ini dibuat supaya bisa langsung di-upload ke GitHub dan dibaca tim secara nyaman.

## 2) Problem & Fix (Temuan)
- Respons kurang cepat → **early ACK + outbox async**
- Chatbot tidak mengingat → **history last N + summary + facts_json**
- Ticket numpuk → **reuse/reopen window + thread lock**
- Butuh memory FAQ + produk → **Hybrid KB (RAG + Sheets facts)**
- Prompt kurang bagus → **guardrails + JSON contract + parsing/validation**

## 3) Tools (Final)
| Tool | Peran | Detail Implementasi |
|---|---|---|
| n8n | Orkestrasi workflow | WF1 (Inbound Orchestrator), WF-SEND (Outbox Sender), WF-KB (Knowledge Sync Hybrid), WF-PAY (Payment Callback opsional) |
| Google Sheets | Database operasional | Menyimpan customers/tickets/messages + reliability tables (idempotency, locks, outbox) + KB (products, faqs, kb_*) + commerce (orders, payments, inventory) |
| AppSheet | UI operasional | UI read-heavy untuk CS/Ops: ticket list, customer profile, order/payment view, inventory ledger, handoff queue; tidak memegang logic kritikal |
| Gemini (LLM) | Model AI | Dipanggil via HTTP node (langsung Google AI API atau via proxy OpenAI-compatible). Output wajib JSON contract agar deterministik |
| WAHA | WhatsApp gateway | Outbound WhatsApp via API sendText. Inbound bisa dari WAHA/webhook provider lain; WF1 menormalisasi |
| Vector DB | Retrieval RAG | Menyimpan embeddings `kb_chunks`; WF-KB melakukan upsert; WF1 melakukan retrieval top-k |


## 4) Struktur Arsitektur
**WF1 (Inbound)** menerima pesan → menulis `messages`, `tickets`, `customers`, dan membuat task `outbox`.  
**WF-SEND** mengambil `outbox` → mengirim via WAHA → retry/backoff jika gagal.  
**WF-KB** sinkron knowledge dari `products/faqs` → `kb_documents/kb_chunks` → Vector DB.  
**WF-PAY** (opsional) menerima payment callback → update `payments/orders/inventory_ledger`.

## 5) Flowchart
```mermaid
flowchart TD
  A[Pesan Masuk] --> B[WF1 Normalize + Log + Dedupe]
  B --> C[Resolve Customer]
  C --> D[Resolve Ticket (reuse/reopen)]
  D --> E[Hybrid Retrieve: RAG + Sheets Facts]
  E --> F[Gemini/LLM JSON Contract]
  F -->|Reply| G[Write Outbox PENDING]
  G --> H[WF-SEND Cron]
  H --> I[Send + Retry/Backoff]
  I --> J[Log OUT + Update Ticket]
```

## 6) Database Schema (Nama Sheet + Header)
### `audit_log`
Header:

```
audit_id	occurred_at	actor_type	actor_id	action	entity_type	entity_id	before_json	after_json	source_workflow	source_message_id	trace_id	notes
```

### `counters`
Header:

```
date	seq	entity
```

### `customers`
Header:

```
customer_id	created_at	updated_at	status	name	phone	email	telegram_username	default_channel	default_thread_id	address_line	kelurahan	kecamatan	kabupaten	provinsi	country_code	postal_code	notes	profile_json	preferences_json	active_ticket_id	last_intent	last_seen_at
```

### `errors`
Header:

```
error_id	occurred_at	workflow_name	workflow_id	execution_id	node_name	ticket_id	source_message_id	trace_id	channel	direction	severity	error_message	error_stack	payload_json	status	last_node	provider	http_status	retry_count
```

### `faqs`
Header:

```
faq_id	question	answer	tags	updated_at	category	question_patterns	is_active	source_url	rag_text	created_at	checksum	last_indexed_at
```

### `idempotency_keys`
Header:

```
idem_key	scope	ref_type	ref_id	status	created_at	expires_at	last_seen_at	result_json	error_message
```

### `inventory_ledger`
Header:

```
ledger_id	created_at	type	ref_type	ref_id	product_id	qty_change	stock_before	stock_after	actor	notes	sku	warehouse	idempotency_key	raw_json	actor_type
```

### `jobs`
Header:

```
job_id	type	status	priority	run_at	created_at	updated_at	attempt_count	last_attempt_at	next_retry_at	payload_json	result_json	error_message
```

### `kb_chunks`
Header:

```
chunk_id	doc_id	chunk_index	chunk_text	checksum	tokens	created_at	updated_at	index_status	last_indexed_at	vector_id	error_message	metadata_json
```

### `kb_documents`
Header:

```
doc_id	source_type	source_id	title	content	tags	category	is_active	checksum	created_at	updated_at	index_status	last_indexed_at	vector_id	error_message	metadata_json
```

### `kb_search_cache`
Header:

```
cache_key	query_text	topk	result_json	created_at	expires_at	hit_count	last_hit_at
```

### `messages`
Header:

```
message_id	provider_message_id	ticket_id	thread_id	direction	channel	sender	sent_at	content	raw_json	provider	provider_event_id	dedupe_key	intent	latency_ms	ai_json
```

### `order_items`
Header:

```
order_item_id	order_id	product_id	sku	product_name	qty	unit_price	line_total	created_at	uom	notes	meta_json
```

### `orders`
Header:

```
order_id	created_at	updated_at	status	channel	ticket_id	customer_id	customer_name	customer_contact	shipping_address	subtotal	discount	tax	shipping_fee	grand_total	currency	notes	invoice_id	invoice_due_at	invoice_url	payment_status	payment_due_at	idempotency_key	source_message_id	meta_json
```

### `outbox`
Header:

```
outbox_id	created_at	updated_at	status	channel	provider	thread_id	ticket_id	message_id	idempotency_key	payload_json	response_json	last_attempt_at	attempt_count	next_retry_at	error_message
```

### `payments`
Header:

```
payment_id	order_id	created_at	status	method	amount	provider_ref	paid_at	proof_url	notes	provider	provider_event_id	idempotency_key	confirmed_at	confirmed_by	raw_json
```

### `products`
Header:

```
product_id	sku	name	category	price	cost	stock_qty	stock_min	is_active	updated_at	created_at	description	keywords	brand	uom	image_url	product_url	rag_text	checksum	last_indexed_at
```

### `settings`
Header:

```
key	value	type	is_active	updated_at	notes
```

### `thread_locks`
Header:

```
thread_id	active_ticket_id	lock_owner	locked_until	last_inbound_message_id	last_inbound_at	last_ticket_status	last_seen_at	created_at	updated_at	notes
```

### `tickets`
Header:

```
ticket_id	created_at	updated_at	status	priority	channel	customer_ref	requester_name	requester_contact	thread_id	subject	description	category	assignee	sla_due_at	last_inbound_at	last_outbound_at	last_message_snippet	unread_flag	source_message_id	dedupe_key	tags	meta_json	conversation_summary	facts_json	stage	handoff_flag	handoff_reason	reopened_at	closed_at	last_ai_at
```


## 6) AppSheet (UI Operasional) — Struktur View yang Disarankan
AppSheet diposisikan sebagai **UI read-heavy** untuk tim CS/Ops, bukan tempat logic kritikal. Semua perubahan penting tetap lewat workflow/aturan terkontrol.

View minimal yang disarankan:
- **Dashboard**: KPI harian (ticket open, handoff pending, outbox retry, payment pending)
- **Tickets** (table: `tickets`): filter `status=open`, `handoff_needed=true`, `unread_flag=true`
- **Ticket Detail**: relasi ke `messages` (IN/OUT), customer, order/payment (jika ada)
- **Customers** (table: `customers`): profil + active_ticket_id + riwayat
- **Outbox Monitor** (table: `outbox`): status PENDING/RETRY/DELIVERED, attempt_count, last_error
- **Knowledge Admin** (tables: `products`, `faqs`): edit konten knowledge, toggle `is_active`
- **Inventory Ledger** (table: `inventory_ledger`): audit stok (read-only / controlled edit)
- **Payments** (table: `payments`): status & raw_json (read-only)

Akses kontrol:
- CS: read + update ringan (assign ticket, notes)
- Ops: inventory ledger controlled
- Admin: products/faqs



## 7) AI Contract (Prompting) — Wajib JSON (Deterministik)
Untuk mengatasi respon kaku dan memastikan AI patuh rules, output AI harus **selalu JSON** dengan schema tetap.

Schema minimum:
```json
{
  "should_reply": true,
  "reply_text": "string",
  "handoff": { "needed": false, "to": null, "reason": null },
  "intent": "faq|order|payment|handoff|smalltalk|unknown",
  "entities": { "product_sku": null, "qty": null, "phone": null },
  "actions": [
    { "type": "NONE" }
  ],
  "ticket_update": { "summary": "string", "facts_json": {} }
}
```

Guardrails utama:
- Jangan pernah mengurangi stok tanpa payment confirmed.
- Jika confidence rendah / data tidak lengkap → ajukan pertanyaan klarifikasi atau handoff.
- Untuk harga/stok → pakai fakta dari Sheets (products), bukan halusinasi.


## 8) Environment Variables (n8n)
| ENV | Contoh | Fungsi |
|---|---|---|
| `GSHEET_ID` | `1AbC...` | Spreadsheet database |
| `WAHA_URL` | `https://waha.domain` | Base URL WAHA |
| `WAHA_API_KEY` | `xxxxx` | Auth WAHA |
| `WAHA_SESSION` | `default` | Session WAHA |
| `VECTOR_URL` | `https://vector.domain` | Endpoint vector DB |
| `VECTOR_API_KEY` | `xxxxx` | Auth vector DB |
| `VECTOR_COLLECTION` | `cdn_kb` | Collection name |
| `RAG_TOPK` | `5` | Top-k retrieval |
| `LLM_URL` | `https://...` | Endpoint Gemini/LLM (langsung/proxy) |
| `LLM_API_KEY` | `xxxxx` | Auth LLM |
| `LLM_MODEL` | `gemini-1.5-flash` | Model name |
| `THREAD_LOCK_SECONDS` | `30` | Lock TTL |
| `TICKET_REUSE_WINDOW_HOURS` | `24` | Reuse/reopen window |
| `OUTBOX_RETRY_BASE_SECONDS` | `30` | Retry base backoff |



## 9) Workflow n8n — Paket Final
File workflow import:
- `workflows/CDN_WF1_HYBRID_WORKFLOWS_v3_all_in_one.json`

Dokumentasi setting node lengkap (dump dari workflow JSON) ada di:
- `docs/n8n_node_parameters.json`

Ringkasan tujuan workflow:
- **WF1 (Inbound Orchestrator)**: normalize → dedupe → lock → resolve customer/ticket → log inbound → hybrid retrieve → LLM contract → write outbox → unlock
- **WF-SEND (Outbox Sender)**: read outbox → send via provider → retry/backoff → log OUT + update ticket
- **WF-KB (Knowledge Sync)**: products/faqs → kb_documents → chunking kb_chunks → upsert vector → mark indexed
- **WF-PAY (Payment Callback)**: payment event idempotent → update payments/orders/inventory (opsional)


## 10) Node Detail — CDN WF1 - Inbound Orchestrator v3 (Hybrid)

Jumlah node: **29**

### 1. Webhook Inbound Unified v3
- **Type**: `n8n-nodes-base.webhook`
- **httpMethod**: `POST`
- **path**: `cdn-wf1-unified-v3`
- **responseMode**: `onReceived`
- **responseCode**: `202`

### 2. Code - Normalize Inbound
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
// Normalize multi-channel inbound payload into internal schema.
// Expect req body in $json (from Webhook).
const body = $json.body ?? $json;
// --- Provider detection (edit as needed) ---
let provider = body.provider || "WAHA";
let channel  = body.channel  || "whatsapp";

// Thread id normalization (prioritize WAHA fields if present)
const thread_id = body.thread_id
  || body.remoteJidAlt
  || body.remoteJid
  || body.from
  || body.chatId
  || body.email_thread_id
  || "";

const content = body.content
  || body.text
  || body.message
  || (body.data && body.data.text)
  || "";

const provider_message_id = body.provider_message_id
  || (body.key && body.key.id)
  || body.messageId
  || bo
…(truncated; lihat docs/n8n_node_parameters.json)…
```

### 3. IF - Schema Valid?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"string": [{"value1": "={{$json.thread_id}}", "operation": "isNotEmpty"}, {"value1": "={{$json.content}}", "operation": "isNotEmpty"}, {"value1": "={{$json.provider_message_id}}", "operation": "isNotEmpty"}]}`

### 4. Code - Build Dedupe Key
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
return [{json:{...$json, dedupe_key:`in|${$json.provider}|${$json.provider_message_id}`}}];
```

### 5. GS - Lookup Idempotency (dedupe_key)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `lookup`
- **sheetName**: `idempotency_keys`
- **lookupColumn**: `idem_key`
- **lookupValue**: `={{$json.dedupe_key}}`

### 6. IF - Already Processed?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"boolean": [{"value1": "={{$json.idem_key !== undefined && ($json.status === 'DONE' || $json.status === 'IN_PROGRESS')}}", "operation": "isTrue"}]}`

### 7. GS - Upsert Idempotency IN_PROGRESS
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `idempotency_keys`

### 8. GS - Lookup Thread Lock
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `lookup`
- **sheetName**: `thread_locks`
- **lookupColumn**: `thread_id`
- **lookupValue**: `={{$json.thread_id}}`

### 9. Code - Decide Lock Acquire
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const lock = $json; // row from lookup or empty item
const now = Date.now();
const lockedUntil = lock.locked_until ? Date.parse(lock.locked_until) : 0;
const acquire = !lock.thread_id || isNaN(lockedUntil) || lockedUntil < now;
return [{json:{...$json, lock_acquire: acquire, now_iso: new Date().toISOString()}}];

```

### 10. IF - Lock Acquired?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"boolean": [{"value1": "={{$json.lock_acquire}}", "operation": "isTrue"}]}`

### 11. GS - Upsert Thread Lock (Acquire)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `appendOrUpdate`
- **sheetName**: `thread_locks`
- **keyColumn**: `thread_id`
- **keyValue**: `={{$json.thread_id}}`

### 12. GS - Lookup Customer (default_thread_id)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `lookup`
- **sheetName**: `customers`
- **lookupColumn**: `default_thread_id`
- **lookupValue**: `={{$json.thread_id}}`

### 13. Code - Resolve Customer
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
// Create deterministic-ish customer_id if missing (simple hash).
function djb2(str){ let h=5381; for (let i=0;i<str.length;i++) h=((h<<5)+h)+str.charCodeAt(i); return h>>>0; }
const existingId = $json.customer_id;
const thread = $json.thread_id || "";
const suffix = (djb2(thread) % 1000000).toString().padStart(6,'0');
const today = new Date().toISOString().slice(0,10).replaceAll('-','');
const customer_id = existingId || `CUS${today}${suffix}`;
return [{
  json: {
    ...$json,
    customer_id,
    updated_at: new Date().toISOString(),
    last_seen_at: new Date().toISOString(),
    default_thread_id: thread,
    default_channel: $json.channel
  }
}];

```

### 14. GS - Upsert Customer
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `appendOrUpdate`
- **sheetName**: `customers`
- **keyColumn**: `customer_id`
- **keyValue**: `={{$json.customer_id}}`

### 15. GS - Read Tickets by thread_id
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `read`
- **sheetName**: `tickets`

### 16. Code - Resolve Ticket (reuse/reopen)
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
// NOTE: Google Sheets read returns many rows; filter in code.
const all = items.map(i => i.json); // if connected correctly, this node should receive list of tickets rows
const thread_id = $json.thread_id;
const reuseHours = Number($env.TICKET_REUSE_WINDOW_HOURS || 24);
const now = Date.now();

const related = all.filter(t => t.thread_id === thread_id);
const isOpen = (s) => !['closed','resolved','selesai'].includes(String(s||'').toLowerCase());
let active = related.filter(t => isOpen(t.status));
active.sort((a,b)=> Date.parse(b.updated_at||b.created_at||0) - Date.parse(a.updated_at||a.created_at||0));

let chosen = active[0];
if (!chosen && related.length){
  related.sort((a,b)=> Date.pars
…(truncated; lihat docs/n8n_node_parameters.json)…
```

### 17. GS - Upsert Ticket
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `appendOrUpdate`
- **sheetName**: `tickets`
- **keyColumn**: `ticket_id`
- **keyValue**: `={{$json.ticket_id}}`

### 18. GS - Update Customer active_ticket_id
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `customers`
- **keyColumn**: `customer_id`
- **keyValue**: `={{$json.customer_id}}`

### 19. GS - Append Inbound Message
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `messages`

### 20. GS - Read Messages (ticket_id)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `read`
- **sheetName**: `messages`

### 21. Code - Build Context (filter last N)
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
// Filter messages for this ticket and build last 20 context
const all = items.map(i => i.json);
const ticket_id = $json.ticket_id;
const msg = all.filter(m => m.ticket_id === ticket_id)
  .sort((a,b)=> Date.parse(a.sent_at||0)-Date.parse(b.sent_at||0))
  .slice(-20);
return [{json:{...$json, history: msg}}];

```

### 22. HTTP - RAG Retrieve (Vector DB)
- **Type**: `n8n-nodes-base.httpRequest`
- **method**: `POST`
- **url**: `={{$env.VECTOR_URL}}/search`
- **jsonBody (excerpt)**: `={{ { collection: $env.VECTOR_COLLECTION || 'cdn_kb', query: $json.content, top_k: Number($env.RAG_TOPK||5), filters: { is_active: true } } }}`

### 23. GS - Lookup Products (dynamic facts)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `read`
- **sheetName**: `products`

### 24. Code - Compose Prompt + Guardrails
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const rag = $json.body || $json; // depending on http node output mapping
const ragSnips = (rag.results || rag.hits || []).slice(0, Number($env.RAG_TOPK||5));
const products = items.map(i=>i.json); // if connected properly: this node should receive products list
const q = ($json.content||'').toLowerCase();
const prodHits = products
  .filter(p => (p.is_active||'').toString().toLowerCase() !== 'false')
  .filter(p => [p.sku,p.name,p.keywords].some(v => (v||'').toLowerCase().includes(q.split(' ')[0]||'')))
  .slice(0,5)
  .map(p => ({sku:p.sku,name:p.name,price:p.price,stock_qty:p.stock_qty}));

const system = [
  "Kamu adalah CS/Sales chatbot. Bahasa Indonesia. Jawab singkat, natural, sopan."
…(truncated; lihat docs/n8n_node_parameters.json)…
```

### 25. HTTP - LLM Chat (OpenAI compatible)
- **Type**: `n8n-nodes-base.httpRequest`
- **method**: `POST`
- **url**: `={{$env.LLM_URL || 'https://api.openai.com/v1/chat/completions'}}`
- **jsonBody (excerpt)**: `={{ { model: ($env.LLM_MODEL || 'gpt-4o-mini'), temperature: 0.2, messages: [ {role:'system', content:$json.llm.system}, {role:'user', content: JSON.stringify($json.llm.context)} ] } }}`

### 26. Code - Parse & Validate AI Contract
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const resp = $json;
const txt = (resp.choices && resp.choices[0] && resp.choices[0].message && resp.choices[0].message.content) ? resp.choices[0].message.content : JSON.stringify(resp);
function extractJson(s){
  const m = s.match(/\{[\s\S]*\}/);
  return m ? m[0] : null;
}
let obj = null;
try { obj = JSON.parse(txt); } catch(e){
  const ex = extractJson(txt);
  if (ex) obj = JSON.parse(ex);
}
if (!obj) obj = {should_reply:false, reply_text:"", actions:[{type:"HANDOFF", reason:"AI output invalid"}], handoff:{required:true, reason:"AI output invalid"}};

const allowed = new Set(["UPDATE_TICKET","CREATE_ORDER_DRAFT","REQUEST_MORE_INFO","HANDOFF","SEND_REPLY","NO_REPLY"]);
obj.actions = Array.i
…(truncated; lihat docs/n8n_node_parameters.json)…
```

### 27. IF - Should Reply?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"boolean": [{"value1": "={{$json.ai.should_reply === true}}", "operation": "isTrue"}]}`

### 28. GS - Append Outbox (PENDING)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `outbox`

### 29. GS - Release Thread Lock
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `thread_locks`
- **keyColumn**: `thread_id`
- **keyValue**: `={{$json.thread_id}}`

## 10) Node Detail — CDN WF-SEND - Outbox Sender v1

Jumlah node: **18**

### 1. Cron - Outbox Worker
- **Type**: `n8n-nodes-base.cron`
- **triggerTimes**: `{"item": [{"mode": "everyMinute"}]}`

### 2. GS - Read Outbox
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `read`
- **sheetName**: `outbox`

### 3. Code - Filter Pending/Retry
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const now = Date.now();
const rows = items.map(i=>i.json);
const eligible = rows.filter(r => {
  const st = String(r.status||'').toUpperCase();
  if (!(st === 'PENDING' || st === 'RETRY')) return false;
  const next = r.next_retry_at ? Date.parse(r.next_retry_at) : 0;
  return isNaN(next) || next <= now;
}).slice(0, 10);
return eligible.map(r => ({json:r}));

```

### 4. Split In Batches
- **Type**: `n8n-nodes-base.splitInBatches`
- **batchSize**: `1`

### 5. Code - Build Send Idem Key
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
return [{json:{...$json, send_idem_key:`send|${$json.outbox_id}`}}];
```

### 6. GS - Lookup Idempotency (send)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `lookup`
- **sheetName**: `idempotency_keys`
- **lookupColumn**: `idem_key`
- **lookupValue**: `={{$json.send_idem_key}}`

### 7. IF - Already Sent?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"boolean": [{"value1": "={{$json.idem_key !== undefined && $json.status === 'DONE'}}", "operation": "isTrue"}]}`

### 8. GS - Append Idempotency IN_PROGRESS (send)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `idempotency_keys`

### 9. Switch - Provider
- **Type**: `n8n-nodes-base.switch`
- **value1**: `={{$json.provider}}`
- **rules**: `[{"operation": "equal", "value2": "WAHA"}, {"operation": "equal", "value2": "TELEGRAM"}, {"operation": "equal", "value2": "EMAIL"}]`

### 10. HTTP - Send WAHA
- **Type**: `n8n-nodes-base.httpRequest`
- **method**: `POST`
- **url**: `={{$env.WAHA_URL}}/api/sendText`
- **jsonBody (excerpt)**: `={{ (()=>{ const p = JSON.parse($json.payload_json||'{}'); return { session: p.session || $env.WAHA_SESSION, chatId: p.to, text: p.text }; })() }}`

### 11. Code - Normalize Send Result
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const ok = !$json.error && ($json.statusCode === undefined || ($json.statusCode >=200 && $json.statusCode < 300));
// WAHA typically returns JSON with id or key.id; keep flexible
const body = $json.body || $json;
const provider_message_id = body.id || (body.key && body.key.id) || body.messageId || '';
return [{json:{...$json, send_ok: ok, provider_message_id, send_raw: JSON.stringify(body), sent_at: new Date().toISOString()}}];

```

### 12. GS - Append Outbound Message
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `messages`

### 13. GS - Update Ticket After Outbound
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `tickets`
- **keyColumn**: `ticket_id`
- **keyValue**: `={{$json.ticket_id}}`

### 14. IF - Send OK?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"boolean": [{"value1": "={{$json.send_ok}}", "operation": "isTrue"}]}`

### 15. GS - Update Outbox DELIVERED
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `outbox`
- **keyColumn**: `outbox_id`
- **keyValue**: `={{$json.outbox_id}}`

### 16. Code - Backoff + Mark RETRY
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const attempts = Number($json.attempt_count||0)+1;
const base = Number($env.OUTBOX_RETRY_BASE_SECONDS || 30);
const backoff = Math.min(base * Math.pow(2, attempts-1), 3600);
return [{json:{...$json, attempt_count: attempts, next_retry_at: new Date(Date.now()+backoff*1000).toISOString(), error_message: ($json.error_message||'send_failed')}}];

```

### 17. GS - Update Outbox RETRY
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `outbox`
- **keyColumn**: `outbox_id`
- **keyValue**: `={{$json.outbox_id}}`

### 18. GS - Append Error
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `errors`

## 10) Node Detail — CDN WF-KB - Hybrid Knowledge Sync v1

Jumlah node: **9**

### 1. Cron - KB Sync
- **Type**: `n8n-nodes-base.cron`
- **triggerTimes**: `{"item": [{"mode": "everyHour"}]}`

### 2. GS - Read Products
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `read`
- **sheetName**: `products`

### 3. GS - Read FAQs
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `read`
- **sheetName**: `faqs`

### 4. Code - Build kb_documents
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
// Merge products + faqs into kb_documents rows.
// Input should be two branches merged manually; for simplicity, keep this node as placeholder.
// Expected output rows fields: doc_id, source_type, source_id, title, content(rag_text), tags, category, is_active, checksum, index_status, last_indexed_at, metadata_json
return items.map(i => ({json: i.json}));

```

### 5. GS - Upsert kb_documents
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `appendOrUpdate`
- **sheetName**: `kb_documents`
- **keyColumn**: `doc_id`
- **keyValue**: `={{$json.doc_id}}`

### 6. Code - Chunking to kb_chunks
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
// Chunk content into kb_chunks
const doc = $json;
const text = doc.content || doc.rag_text || '';
const chunkSize = 900;
const chunks = [];
for (let i=0;i<text.length;i+=chunkSize){
  chunks.push(text.slice(i, i+chunkSize));
}
const out = chunks.map((c, idx) => ({
  json: {
    chunk_id: `CHUNK-${doc.doc_id}-${String(idx+1).padStart(4,'0')}`,
    doc_id: doc.doc_id,
    chunk_index: idx+1,
    chunk_text: c,
    checksum: "",
    tokens: "",
    created_at: new Date().toISOString(),
    updated_at: new Date().toISOString(),
    index_status: "PENDING",
    last_indexed_at: "",
    vector_id: "",
    error_message: "",
    metadata_json: doc.metadata_json || ""
  }
}));
return out;

```

### 7. GS - Upsert kb_chunks
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `appendOrUpdate`
- **sheetName**: `kb_chunks`
- **keyColumn**: `chunk_id`
- **keyValue**: `={{$json.chunk_id}}`

### 8. HTTP - Upsert Vector (placeholder)
- **Type**: `n8n-nodes-base.httpRequest`
- **method**: `POST`
- **url**: `={{$env.VECTOR_URL}}/upsert`
- **jsonBody (excerpt)**: `={{ { collection: $env.VECTOR_COLLECTION || 'cdn_kb', id: $json.chunk_id, text: $json.chunk_text, metadata: { doc_id: $json.doc_id, chunk_index: $json.chunk_index, is_active: true } } }}`

### 9. GS - Update kb_chunks INDEXED
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `kb_chunks`
- **keyColumn**: `chunk_id`
- **keyValue**: `={{$json.chunk_id}}`

## 10) Node Detail — CDN WF-PAY - Payment Callback v1

Jumlah node: **8**

### 1. Webhook - Payment Callback v1
- **Type**: `n8n-nodes-base.webhook`
- **httpMethod**: `POST`
- **path**: `cdn-wf1-payment-v1`
- **responseMode**: `lastNode`
- **responseCode**: `None`

### 2. Code - Normalize Payment Event
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
const body = $json.body ?? $json;
return [{json:{
  provider: body.provider || 'QRIS_GATEWAY',
  provider_event_id: body.event_id || body.eventId || body.ref || '',
  order_id: body.order_id || body.orderId || '',
  amount: body.amount || 0,
  status: body.status || 'confirmed',
  paid_at: body.paid_at || new Date().toISOString(),
  raw_json: JSON.stringify(body)
}}];

```

### 3. Code - Build payment idem_key
- **Type**: `n8n-nodes-base.code`
- **jsCode**:
```javascript
return [{json:{...$json, payment_idem_key:`pay|${$json.provider}|${$json.provider_event_id}`}}];
```

### 4. GS - Lookup Idempotency (payment)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `lookup`
- **sheetName**: `idempotency_keys`
- **lookupColumn**: `idem_key`
- **lookupValue**: `={{$json.payment_idem_key}}`

### 5. IF - Payment Already Processed?
- **Type**: `n8n-nodes-base.if`
- **conditions**: `{"boolean": [{"value1": "={{$json.idem_key !== undefined && $json.status === 'DONE'}}", "operation": "isTrue"}]}`

### 6. GS - Append payments (confirmed)
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `append`
- **sheetName**: `payments`

### 7. GS - Update order paid
- **Type**: `n8n-nodes-base.googleSheets`
- **operation**: `update`
- **sheetName**: `orders`
- **keyColumn**: `order_id`
- **keyValue**: `={{$json.order_id}}`

### 8. Respond - 200 OK
- **Type**: `n8n-nodes-base.respondToWebhook`
- **responseCode**: `200`
- **responseData**: `={{ { ok: true } }}`

