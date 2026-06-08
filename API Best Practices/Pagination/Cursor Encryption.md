---
tags: [api-design, pagination, security, cryptography, typescript]
created: 2026-06-03
---

# Cursor Encryption

> Summary: Cursors in [[Cursor-based Pagination]] range from plain Base64 (obfuscation only) to fully encrypted blobs (AES-256-GCM). The right level depends on what the cursor contains and what you're protecting against. At minimum, prevent clients from constructing cursors manually; at maximum, prevent them from reading cursor contents or reusing another user's cursor. Always return the same error for any cursor failure — never leak *why* it failed.

## The spectrum

| Level | Method | Client can read? | Client can tamper? | Use when |
|---|---|---|---|---|
| 1 — Raw | Plain string | Yes | Yes | Never |
| 2 — Base64 | Encode JSON | Yes | Yes (easy) | Dev/internal only |
| 3 — HMAC | Sign the payload | Yes | No | Public API, non-sensitive data |
| 4 — AES-256-GCM | Encrypt the payload | No | No | Sensitive data, multi-tenant APIs |

---

## Level 2 — Base64 (obfuscation only)

```typescript
const cursor = Buffer.from(JSON.stringify({
  created_at: record.created_at,
  id:         record.id,
})).toString('base64url');
```

Anyone can decode this with `atob()`. It signals "don't build on this" but doesn't enforce it. Fine for internal APIs. Not suitable when the cursor contains sensitive values or when you need to prevent manual cursor construction.

---

## Level 3 — HMAC signing (integrity, no secrecy)

Payload is readable but tampered cursors are rejected. Prevents clients from forging arbitrary positions.

```typescript
import { createHmac, timingSafeEqual } from 'crypto';

const SECRET = process.env.CURSOR_SECRET!; // openssl rand -hex 32

function encodeCursor(payload: object): string {
  const data = Buffer.from(JSON.stringify(payload)).toString('base64url');
  const sig  = createHmac('sha256', SECRET).update(data).digest('base64url');
  return `${data}.${sig}`;
}

function decodeCursor(cursor: string): object {
  const dot = cursor.lastIndexOf('.');
  if (dot === -1) throw new Error('Invalid cursor');

  const data = cursor.slice(0, dot);
  const sig  = cursor.slice(dot + 1);

  const expected = createHmac('sha256', SECRET).update(data).digest('base64url');

  // Constant-time compare — prevents timing attacks
  const a = Buffer.from(sig,      'base64url');
  const b = Buffer.from(expected, 'base64url');
  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    throw new Error('Invalid cursor');
  }

  return JSON.parse(Buffer.from(data, 'base64url').toString());
}
```

The client can still decode `data` and see the payload. Use Level 4 if that's a concern.

---

## Level 4 — AES-256-GCM (full secrecy + integrity)

Payload is encrypted. Clients see an opaque blob; they cannot read or tamper with it. The GCM auth tag catches any modification, even a single flipped bit.

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

// 32 bytes (256 bits) — generate with: openssl rand -hex 32
const KEY = Buffer.from(process.env.CURSOR_KEY!, 'hex');

function encodeCursor(payload: object): string {
  const iv     = randomBytes(12);                         // 96-bit IV for GCM
  const cipher = createCipheriv('aes-256-gcm', KEY, iv);

  const encrypted = Buffer.concat([
    cipher.update(JSON.stringify(payload), 'utf8'),
    cipher.final(),
  ]);
  const authTag = cipher.getAuthTag();                    // 16-byte integrity tag

  // Pack: IV (12 bytes) + authTag (16 bytes) + ciphertext
  return Buffer.concat([iv, authTag, encrypted]).toString('base64url');
}

function decodeCursor(cursor: string): object {
  let buf: Buffer;
  try {
    buf = Buffer.from(cursor, 'base64url');
  } catch {
    throw new Error('Invalid cursor');
  }

  if (buf.length < 29) throw new Error('Invalid cursor'); // 12 + 16 + 1 minimum

  const iv        = buf.subarray(0, 12);
  const authTag   = buf.subarray(12, 28);
  const encrypted = buf.subarray(28);

  try {
    const decipher = createDecipheriv('aes-256-gcm', KEY, iv);
    decipher.setAuthTag(authTag);

    const decrypted = Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),             // throws if auth tag doesn't match
    ]);

    return JSON.parse(decrypted.toString('utf8'));
  } catch {
    throw new Error('Invalid cursor'); // same error regardless of why
  }
}
```

---

## What to put in the cursor payload

```typescript
interface CursorPayload {
  // Sort columns — required for the keyset query
  created_at: string;    // ISO 8601
  id:         string;    // unique tiebreaker

  // Optional security fields
  user_id?: string;      // reject cursors from other users
  exp?:     number;      // Unix timestamp — cursor expiry
  version?: number;      // bump to invalidate all existing cursors
}
```

### `user_id` — prevent cross-user cursor reuse

```typescript
function decodeCursor(cursor: string, requestingUserId: string): CursorPayload {
  const payload = decrypt(cursor);
  if (payload.user_id && payload.user_id !== requestingUserId) {
    throw new Error('Invalid cursor'); // same error — don't leak why
  }
  return payload;
}
```

### `exp` — cursor expiry

Useful for live feeds where old cursors point to archived data:

```typescript
function encodeCursor(payload: Omit<CursorPayload, 'exp'>): string {
  return encrypt({ ...payload, exp: Math.floor(Date.now() / 1000) + 3600 });
}

function decodeCursor(cursor: string): CursorPayload {
  const payload = decrypt(cursor);
  if (payload.exp && payload.exp < Math.floor(Date.now() / 1000)) {
    throw new Error('Cursor expired');
  }
  return payload;
}
```

### `version` — instant invalidation of all cursors

```typescript
const CURSOR_VERSION = 2; // bump when sort order or schema changes

// Include in encode; check in decode:
if (payload.version !== CURSOR_VERSION) throw new Error('Invalid cursor');
```

---

## Error handling

Always return the same error regardless of why the cursor failed:

```typescript
app.get('/v1/orders', async (req, res) => {
  if (req.query.after) {
    try {
      cursorPayload = decodeCursor(req.query.after as string, req.user.id);
    } catch {
      return res.status(400).json({
        type:   'https://api.example.com/errors/invalid-cursor',
        title:  'Invalid or expired pagination cursor.',
        status: 400,
        detail: 'Restart pagination from the first page.',
      });
    }
  }
});
```

> [!warning] Same error for all failures
> Don't distinguish "tampered" vs "expired" vs "wrong user" in the response. Any difference leaks information to an attacker probing your cursor validation.

---

## Key management

```bash
# Generate a key
openssl rand -hex 32
```

- Store in environment variables or a secret manager (AWS Secrets Manager, HashiCorp Vault). Never hardcode.
- Rotate via the `version` field — add a new key for version N+1, keep the old key for N until cursors expire.

```typescript
const CURSOR_KEYS: Record<number, Buffer> = {
  1: Buffer.from(process.env.CURSOR_KEY_V1!, 'hex'),  // retiring
  2: Buffer.from(process.env.CURSOR_KEY_V2!, 'hex'),  // current
};
const CURRENT_VERSION = 2;
```

---

## Which level do you need?

| Situation | Level |
|---|---|
| Internal API, non-sensitive data | 2 (Base64) |
| Public API, prevent manual cursor construction | 3 (HMAC) |
| Cursor contains sensitive values (prices, private timestamps) | 4 (AES-256-GCM) |
| Multi-tenant — cursor leak = another user's data | 4 + `user_id` binding |
| Live feed with time-limited validity | 4 + `exp` |

Most public production APIs land at Level 3 or 4. Level 2 is the "we'll fix it later" tier that tends to stay in production.

## References

- Node.js `crypto` module: https://nodejs.org/api/crypto.html
- AES-GCM on MDN: https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/encrypt#aes-gcm
- OWASP cryptographic storage cheat sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html

## Related

- [[Cursor-based Pagination]]
- [[Pagination Introduction]]
- [[JSON Web Tokens]]
- [[Authentication and Authorization patterns]]
