# screenmine_io — Solana-based Anonymous Messenger (MVP)

**screenmine_io** is a crypto-native messenger where anonymity is the foundation, not a feature.
Messages live off-chain (encrypted), while **on-chain signals on Solana** notify recipients without
revealing identities or content.

> “They watch the screen — we mine the silence.”

---

## ✨ Key Properties
- **Anonymity by default:** no phone, no email, no KYC — only keys (Ed25519/X25519).
- **Encrypted off-chain storage:** IPFS/DHT stores ciphertext only.
- **Solana signals:** cheap, fast, censorship-resistant notifications.
- **Zero-knowledge friendly:** future-proof for ZK proofs of membership/trust.
- **User control:** local self-destruct timers; keys never leave device.

---

## 🏗 Architecture (High Level)
- **Client (Sender):** generate ephemeral keys → encrypt (X25519 + ChaCha20-Poly1305) → upload ciphertext → emit Solana signal (Memo or program log) with a short marker.
- **Solana (Signals Layer):** transaction with Memo OR custom Anchor program emitting a stable log (`recipient_tag`, `nonce`, `mac`).
- **P2P Indexer:** listens Solana slots/logs, maps `recipient_tag` to recipients (no plaintext storage), notifies over p2p.
- **Encrypted Storage:** IPFS/Filecoin/DHT storing only ciphertext (no metadata). Receiver fetches CID and decrypts locally.

See the architecture diagram: **screenmine_sol_arch.png**

---

## 🔐 Signal Format (V1)
Fields carried either in **Memo** (hex/base58) or **program log**:

~~~
version: u8 = 1
flags:   u8            // msg/file/handshake
recipient_tag: [u8;16] // H(IdentityPub || session_salt)[0..16]
nonce:   [u8;12]
mac:     [u8;16]       // HMAC_k(CID || nonce), k from X25519 shared secret
~~~

- **CID/link never appears on-chain.**
- `recipient_tag` is discoverable only by the intended recipient (derivation with local salts).

---

## 🧩 On-chain Options
**A) MVP — Memo Program**
- Put the marker into Memo and (optionally) attach a micro-transfer to a PDA.
- Pros: zero contract work, ultra-cheap, simple indexing.
- Cons: fewer invariants (logic enforced client-side).

**B) Custom Anchor Program (stateless)**
- Instruction `emit_signal(recipient_tag, nonce, mac)` emits a stable log event.
- Optional SMT fee collection via CPI into an SPL-associated treasury.
- Pros: predictable event format, built-in fee policy.
- Cons: requires audit.

---

## 💠 SMT (SPL Token)
- **Anti-spam fee:** tiny SMT amount per signal (e.g., $0.001).
- **Node incentives:** indexer/storage nodes receive SMT rewards.
- **Staking for QoS:** stake SMT to prioritize reliable relays.
- UX: accept SOL/USDC; swap to SMT with Jupiter/Orca under the hood.

---

## 🛡 Security & Privacy
- Tor/I2P routing for clients and indexers; no server logs.
- Ephemeral keys per session/conversation; identity key in Secure Enclave/Keystore.
- Time-window batching + jitter to resist timing correlation.
- Local encrypted storage with auto-wipe timers.

---

## 🧪 Getting Started (Dev)
### Prereqs
- Node 18+, pnpm, Rust toolchain, Solana CLI, Anchor (if using option B).
- IPFS node or a pinning service (for tests you can use a local IPFS daemon).

### Install (client prototype)
~~~bash
pnpm i
pnpm dev
~~~

### Send a Signal via Memo (TypeScript)
~~~ts
import { Connection, Keypair, PublicKey, SystemProgram, Transaction } from '@solana/web3.js';
import { MemoProgram } from '@solana/spl-memo'; // or construct a raw memo instruction
import bs58 from 'bs58';

const connection = new Connection('https://api.devnet.solana.com'); // or mainnet
const payer = Keypair.generate();

// Build marker bytes off-chain (version|recipient_tag|nonce|mac)
const marker = new Uint8Array([/* ... */]); // 1 + 1 + 16 + 12 + 16
const memoIx = MemoProgram.memo({ memo: bs58.encode(marker) });

const tx = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey: payer.publicKey,
    toPubkey: new PublicKey('RecipientPDA111111111111111111111111111111'), // optional
    lamports: 5000 // micro-transfer (optional)
  }),
  memoIx
);

await connection.sendTransaction(tx, [payer]);
~~~

### Indexer Sketch (Rust/Go/Python)
- Subscribe to slots and filter txs with Memo or program logs.
- Parse marker bytes; match `recipient_tag` against a local per-user table.
- Notify clients over p2p (Noise/WebRTC data channel/QUIC).

---

## 🗺 Roadmap (MVP → v1)
1. Crypto core (Rust + TS bindings): X25519, ChaCha20-Poly1305, HKDF.
2. IPFS integration + ciphertext packing (CID obfuscation).
3. Memo-based signals + lightweight indexer.
4. RN desktop/mobile client: keygen, QR exchange, chat, timers.
5. SPL token (SMT) + client-side fee policy.
6. Closed beta; telemetry-free profiling; threat modeling.
7. Optional Anchor `signal` program + external audit.
8. Public launch, docs & SDK.

---

## 📄 License
TBD (Apache-2.0 or MIT for SDK; brand assets restricted).
