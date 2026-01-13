# Database Headers (Copy/Paste)

## audit_log

```
audit_id	occurred_at	actor_type	actor_id	action	entity_type	entity_id	before_json	after_json	source_workflow	source_message_id	trace_id	notes
```

## counters

```
date	seq	entity
```

## customers

```
customer_id	created_at	updated_at	status	name	phone	email	telegram_username	default_channel	default_thread_id	address_line	kelurahan	kecamatan	kabupaten	provinsi	country_code	postal_code	notes	profile_json	preferences_json	active_ticket_id	last_intent	last_seen_at
```

## errors

```
error_id	occurred_at	workflow_name	workflow_id	execution_id	node_name	ticket_id	source_message_id	trace_id	channel	direction	severity	error_message	error_stack	payload_json	status	last_node	provider	http_status	retry_count
```

## faqs

```
faq_id	question	answer	tags	updated_at	category	question_patterns	is_active	source_url	rag_text	created_at	checksum	last_indexed_at
```

## idempotency_keys

```
idem_key	scope	ref_type	ref_id	status	created_at	expires_at	last_seen_at	result_json	error_message
```

## inventory_ledger

```
ledger_id	created_at	type	ref_type	ref_id	product_id	qty_change	stock_before	stock_after	actor	notes	sku	warehouse	idempotency_key	raw_json	actor_type
```

## jobs

```
job_id	type	status	priority	run_at	created_at	updated_at	attempt_count	last_attempt_at	next_retry_at	payload_json	result_json	error_message
```

## kb_chunks

```
chunk_id	doc_id	chunk_index	chunk_text	checksum	tokens	created_at	updated_at	index_status	last_indexed_at	vector_id	error_message	metadata_json
```

## kb_documents

```
doc_id	source_type	source_id	title	content	tags	category	is_active	checksum	created_at	updated_at	index_status	last_indexed_at	vector_id	error_message	metadata_json
```

## kb_search_cache

```
cache_key	query_text	topk	result_json	created_at	expires_at	hit_count	last_hit_at
```

## messages

```
message_id	provider_message_id	ticket_id	thread_id	direction	channel	sender	sent_at	content	raw_json	provider	provider_event_id	dedupe_key	intent	latency_ms	ai_json
```

## order_items

```
order_item_id	order_id	product_id	sku	product_name	qty	unit_price	line_total	created_at	uom	notes	meta_json
```

## orders

```
order_id	created_at	updated_at	status	channel	ticket_id	customer_id	customer_name	customer_contact	shipping_address	subtotal	discount	tax	shipping_fee	grand_total	currency	notes	invoice_id	invoice_due_at	invoice_url	payment_status	payment_due_at	idempotency_key	source_message_id	meta_json
```

## outbox

```
outbox_id	created_at	updated_at	status	channel	provider	thread_id	ticket_id	message_id	idempotency_key	payload_json	response_json	last_attempt_at	attempt_count	next_retry_at	error_message
```

## payments

```
payment_id	order_id	created_at	status	method	amount	provider_ref	paid_at	proof_url	notes	provider	provider_event_id	idempotency_key	confirmed_at	confirmed_by	raw_json
```

## products

```
product_id	sku	name	category	price	cost	stock_qty	stock_min	is_active	updated_at	created_at	description	keywords	brand	uom	image_url	product_url	rag_text	checksum	last_indexed_at
```

## settings

```
key	value	type	is_active	updated_at	notes
```

## thread_locks

```
thread_id	active_ticket_id	lock_owner	locked_until	last_inbound_message_id	last_inbound_at	last_ticket_status	last_seen_at	created_at	updated_at	notes
```

## tickets

```
ticket_id	created_at	updated_at	status	priority	channel	customer_ref	requester_name	requester_contact	thread_id	subject	description	category	assignee	sla_due_at	last_inbound_at	last_outbound_at	last_message_snippet	unread_flag	source_message_id	dedupe_key	tags	meta_json	conversation_summary	facts_json	stage	handoff_flag	handoff_reason	reopened_at	closed_at	last_ai_at
```

