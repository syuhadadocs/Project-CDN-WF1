# Project CDN-WF1

Repo ini berisi dokumentasi lengkap (GitHub-ready) dan workflow n8n untuk implementasi **CDN WF1 Hybrid v3**.

## Mulai dari sini
- Baca: `docs/blueprint.md`
- Import workflow n8n: `workflows/CDN_WF1_HYBRID_WORKFLOWS_v3_all_in_one.json`
- Import single workflow (untuk paste ke editor n8n): 
  - `workflows/cdn_wf1_inbound_orchestrator_v3_hybrid.json`
  - `workflows/cdn_wf_send_outbox_sender_v1.json`
  - `workflows/cdn_wf_kb_hybrid_knowledge_sync_v1.json`
  - `workflows/cdn_wf_pay_payment_callback_v1.json`
- Template DB: `CDN_WF1_Master_Template.xlsx`

## Catatan Keamanan
Jangan commit API key/credential. Gunakan `n8n Credentials` dan/atau env vars.
