# Klevar Hash Chain Anchors

**Public, append-only, externally-verifiable RFC 3161 TSA anchor receipts for the Klevar Group document hash chain.**

Klevar Docs (`docs.klevar.ai`) maintains a SHA-256 hash chain of every finalised document (invoices, credit notes, board resolutions, etc.) per legal entity (FZE / LLC / Ltd). Once a month the chain head is anchored against multiple time-stamping authorities (FreeTSA + OpenTimestamps + DigiCert QTSP when configured) — and a receipt is published here.

Anyone can clone this repo and re-verify any historical anchor without touching Klevar's infrastructure.

## Layout

```
anchors/
  fze/
    2026-05/
      0000142-freetsa.json
      0000142-opentimestamps.json
      0000142-digicert-qtsp.json
    2026-06/
      ...
  llc/
    ...
  ltd/
    ...
```

Path: `anchors/{entity_id}/{YYYY-MM}/{chain_index}-{tsa_provider_slug}.json`

## Receipt schema

```json
{
  "entity_id": "fze",
  "anchored_chain_index": 142,
  "anchored_chain_link_hash": "<64-char hex SHA-256>",
  "tsa_provider": "freetsa",
  "provider_is_qualified": false,
  "tsa_timestamp": "2026-05-01T06:00:14.123Z",
  "tsa_serial": "<TSA-assigned serial>",
  "tsa_response_b64": "<base64 RFC 3161 TimeStampResp DER bytes>",
  "anchor_trigger": "monthly_cron",
  "published_at": "2026-05-01T06:00:18.456Z"
}
```

`tsa_response_b64` is the full DER-encoded `TimeStampResp` returned by the TSA — ASN.1-parseable with any RFC 3161 library (Bouncy Castle, node-forge, OpenSSL).

## Verifying a receipt

Given a receipt at `anchors/<entity>/<month>/<index>-<provider>.json`:

1. **Reproduce the chain link hash up to that index.** Klevar Docs's verifier exposes
   `GET https://docs.klevar.ai/api/hash-chain/:entity_id?up_to_index=<index>` (public-readable for verification audits).
   The response includes the deterministic chain link hash at that index — should match
   `anchored_chain_link_hash` in the receipt.

2. **Decode the TSA response.** Base64-decode `tsa_response_b64`, parse the ASN.1 `TimeStampResp` →
   `TimeStampToken` → `TSTInfo`. The `messageImprint.hashedMessage` field MUST equal
   `anchored_chain_link_hash` (the hash that was anchored).

3. **Verify the TSA signature.** The `TimeStampToken` is a CMS SignedData over the `TSTInfo`.
   Extract the TSA's signing certificate from the `TimeStampToken.certificates` field, validate the
   chain against the TSA's published root, and verify the CMS signature.

4. **Confirm `tsa_timestamp` matches `TSTInfo.genTime`.** The receipt's timestamp is just metadata —
   the cryptographic truth lives inside the TSA response bytes.

If steps 1-4 all hold, the chain at index N existed at time `tsa_timestamp` and has not been
altered since.

## What this proves

- Klevar Docs cannot retroactively alter document history — any change downstream of an anchored
  chain index breaks the hash chain, and the TSA receipts here are independent witnesses.
- Multiple anchors per index (3 providers per month) means even if one TSA is later compromised
  or shuts down, the others remain verifiable.

## What this does NOT prove

- The CONTENT of any individual document (the chain anchors hashes, not files).
- Klevar's internal access controls.
- Anything that happened BEFORE the first anchor in this repo.

## Append-only invariant

This repo is append-only. Every commit adds new receipt files and never modifies or deletes
existing ones. Branch protection on `main` rejects force-pushes and history-rewriting commits.
The Git history of this repo is itself a tamper-evident audit trail; combined with the anchored
TSA receipts inside each commit, it gives anyone two independent ways to detect tampering.

## Generating receipts

Receipts are written by Klevar Docs's hash-chain TSA anchor cron (`src/hash-chain/anchor.ts`),
which runs monthly at `0 6 1 * *` UTC. The cron path is:

1. Compute the current chain head per entity
2. Submit it to every configured TSA (FreeTSA + OpenTimestamps + DigiCert QTSP)
3. Persist the response to the Klevar Docs DB (`hash_chain_anchors` table) for internal audit
4. **Publish a copy of the response here** (this repo) for external audit

Step 4 is fire-and-forget — failure to publish does NOT block the anchor write to step 3.

## License

Receipts are released under [CC0-1.0](https://creativecommons.org/publicdomain/zero/1.0/) — no
restrictions on copying, redistribution, or use for verification.
