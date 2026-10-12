This PR changes EIP-8250 (Keyed Nonces) so that binary single-use nonces, such as nullifiers, can be told apart from general sequence-number nonces, both in the transaction and in state.

Background: <ethresear.ch post on Bloom filters and keyed nonces>. If binary nonces are recognisable in state, (weak) stateless nodes can keep much less nonce state as VOPS state, for example by putting spent binary keys in a Bloom filter. Fixing the distinction now means such optimizations will also apply to every binary nonce consumed before they ship.

## Changes

**New `nonce_type` field**
- Adds `nonce_type` (`uint8`) and orders the nonce fields as `nonce_seq`, `nonce_type`, `nonce_keys`:
  `[chain_id, nonce_seq, nonce_type, nonce_keys, sender, ...]`
- Valid values are `NONCETYPE_GENERAL = 0` and `NONCETYPE_BINARY = 1`. Other values are rejected at decoding and reserved for future EIPs, for example nonces valid only for a limited time.
- One type applies to all keys in the transaction. Its encoding is priced as calldata like the other nonce fields, and the signature hash covers it.

**Type in the storage slot**
- The slot is now `keccak256(A || T || K)`, so the same key has a separate slot under each type.

**Binary nonce semantics**
- A binary-type transaction is valid only if `nonce_seq == 0` and every selected slot is unused.
- Consuming a binary key writes `MAX_NONCE_SEQ`.
- Clients MAY already reject a binary-type transaction with `nonce_seq != 0` at decoding, since it can never be valid.
- Decoding now requires `nonce_seq < MAX_NONCE_SEQ`, so no transaction can match a slot holding `MAX_NONCE_SEQ`. A stored `MAX_NONCE_SEQ` therefore means a spent binary key or an exhausted general key. Every stored value below it belongs to a live general key, and binary slots are written exactly once.

**Legacy nonce encoding**
- The legacy account nonce is now selected by `nonce_keys == []` instead of `[0]`.
- A key of `0` is invalid, so every listed key is a `NONCE_MANAGER` key.
- `nonce_keys == []` requires `NONCETYPE_GENERAL`.
- `TXPARAM(0x0E)` is `0` for legacy transactions. `TXPARAM(0x10)` returns `0` when the list is empty.
- Verifier code that assumes `TXPARAM(0x01)` is the legacy account nonce now guards with `TXPARAM(0x0E) == 0` (Backwards Compatibility).

**Introspection**
- Adds `TXPARAM(0x11)`, which returns `tx.nonce_type`.

**Mempool**
- The pending transaction identity becomes `(sender, nonce_seq, nonce_type, nonce_keys)`.
- Keyed dependencies are looked up per `(nonce_type, nonce_key)`.
- Legacy transactions keep their dependency on the account nonce.

**Rationale and Security Considerations**
- Rationale explains why the type is a separate field rather than a byte of the key: it keeps the full 32-byte key space for hash outputs and field elements.
- Rationale also explains why one type is shared by all keys.
- Single-use applications MUST now pin `nonce_type == NONCETYPE_BINARY` (via `TXPARAM(0x11)`) in `VERIFY`, in addition to authenticating `nonce_keys_hash`.
- Security Considerations note that the same key can be consumed once under each type, so a general-type transaction does not consume a key's binary slot.

## Out of scope

- Incentives to prefer binary nonces over general ones, such as cheaper state gas.
- Separate system contracts per nonce type.

## History

- The first version encoded the type in the first byte of the nonce key and capped binary nonces at sequence 1.
- Following review (@ilitteri, @soispoke, @nerolation), it moved to a separate `nonce_type` field and to writing `MAX_NONCE_SEQ` for spent binary keys. This folds in the closed #12441.
- The latest changes encode the legacy nonce as an empty `nonce_keys` list and reorder the fields to `nonce_seq, nonce_type, nonce_keys` (slightly better for compression).
- The matching execution-specs changes will need to follow the new encoding.
