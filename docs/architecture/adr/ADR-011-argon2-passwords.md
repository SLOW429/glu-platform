# ADR-011: Argon2id for Password Hashing

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU stores user passwords. Passwords must be hashed with an algorithm that is slow by design (to resist brute-force attacks), memory-hard (to resist GPU/ASIC attacks), and recommended by current security standards.

---

## Decision

**Use Argon2id for password hashing** with parameters: memory=65536 (64MB), iterations=3, parallelism=4.

---

## Alternatives Considered

| Algorithm | Description |
|---|---|
| **bcrypt** | De-facto standard for decades, CPU-intensive, max 72-byte input |
| **scrypt** | Memory-hard, used by many cloud providers |
| **PBKDF2** | NIST-recommended, minimal hardware requirements |
| **Argon2id** | Password Hashing Competition winner (2015), memory-hard, side-channel resistant |

---

## Reasons for Argon2id

### 1. Winner of the Password Hashing Competition

Argon2 was designed specifically to replace bcrypt/scrypt. It won the Password Hashing Competition in 2015 — a rigorous open competition with expert review. It is now recommended by OWASP, NIST SP 800-63B, and RFC 9106.

### 2. Memory-hard by design

Argon2 requires a configurable amount of memory (we use 64MB) during hashing. GPU clusters — the attacker's primary tool — cannot amortize the memory requirement across thousands of parallel cores. A GPU that runs 10,000 bcrypt hashes in parallel can only run ~100 Argon2 hashes in parallel with the same cost.

### 3. Three variants for different threat models

- **Argon2d**: maximizes GPU resistance, susceptible to side-channel attacks
- **Argon2i**: side-channel resistant, less GPU resistant
- **Argon2id**: hybrid — first pass uses Argon2i (side-channel safe), subsequent passes use Argon2d (GPU resistant). Best of both. OWASP recommends Argon2id.

### 4. bcrypt limitations

bcrypt has a 72-byte input limit. A user with a password longer than 72 bytes gets no additional security (bcrypt silently truncates). Argon2 has no such limit.

bcrypt uses a cost factor (2^N iterations) that grows slowly. Modern hardware (2024) can compute ~100,000 bcrypt hashes per second on a GPU. Argon2's memory requirement makes this infeasible.

---

## Parameters Chosen

```
memory:      65536 KB (64 MB)
iterations:  3
parallelism: 4
```

These parameters take ~100-300ms to hash on a typical server CPU. This is intentional — slow enough to frustrate brute-force attacks, fast enough for legitimate login flows.

OWASP minimum recommendation: memory=65536, iterations=2, parallelism=1. GLU exceeds the minimum.

---

## Pros

- Best current recommendation from OWASP, NIST, RFC 9106
- Memory-hard (GPU/ASIC resistant)
- Side-channel resistant (Argon2id variant)
- No 72-byte truncation issue (unlike bcrypt)
- Configurable parameters (can increase cost as hardware improves)

## Cons

- Higher memory usage per login request vs bcrypt (mitigated: login is low-frequency)
- Slightly less ecosystem maturity than bcrypt in Node.js (`argon2` npm package is well-maintained but not a Node.js builtin)

---

## Consequences

- `argon2` npm package used (`npm install argon2`)
- Password hashing isolated in `AuthService` — no other module touches raw passwords
- Parameters stored as constants in `auth.config.ts` (not hardcoded in the service)
- Parameters can be increased in future without breaking existing passwords (Argon2 output includes parameters for verification)
- On login, verify hash → if parameters are outdated, re-hash and update in DB transparently
