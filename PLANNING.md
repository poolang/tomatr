# Tomatr V2 — Planning Document

## What is Tomatr?
A "Rotten Tomatoes" for Farcaster. Users reply to casts with 🍅 to register 
disapproval. Community validation via likes determines how meaningful each 
tomato is. Built as a Farcaster mini app.

## Core Mechanic
1. User replies to a cast with 🍅
2. Tomatr detects the reply via Neynar webhook
3. Bot confirms the tomato was logged
4. Other users like the 🍅 reply to validate it
5. Likes accumulate and affect both users' Splat Scores

---

## Splat Score
A community-driven reputation score between 0.00 and 1.00, similar to the 
Neynar user score but determined entirely by users, not an algorithm.

### Default score: 0.50 (neutral, like Neynar)
### Display: rounded to 2 decimal places (e.g. 0.52)

### Score mechanics:

| Event | Effect |
|---|---|
| You throw a tomato | -0.03 immediately |
| Your tomato gets its 1st like | +0.03 (break even) |
| Each additional like after 1st | +0.015 |
| You get pelted | Score down, weighted by thrower's splat score |
| 0 likes on your throw | -0.03 net (embarrassing, no validation) |
| Someone unlikes your tomato | Score adjusts back down accordingly |

### Pelt weight:
A tomato from a high splat score account hurts more than one from a low 
score account. Stops brigading from throwaway/low quality accounts.

### Score curve example:
- 0 likes → -0.03
- 1 like  → 0.00 (break even)
- 2 likes → +0.015
- 5 likes → +0.06
- 10 likes → +0.135

### Score floor/ceiling:
- Soft floor: 0.05 (score never gets permanently stuck at zero)
- Soft ceiling: 0.95 (always room to move)

### Formula:
```
raw_splat_score = (liked_tomatoes × avg_likes) /
                  (liked_tomatoes × avg_likes + pelts_received_weighted + damping)
```
- `liked_tomatoes` — count of your throws with 1+ likes
- `avg_likes` — average likes across your throws
- `pelts_received_weighted` — each pelt multiplied by thrower's splat score
- `damping` — constant of 50, prevents new accounts jumping to 1.0 instantly

---

## Throw Quota
- Every user gets 5 throws per day
- Resets at midnight EST
- Unused throws do NOT roll over

---

## Anti-Abuse Rules
- No self-tomatoes (thrower_fid cannot equal target_fid)
- Deduplication: same user cannot tomato the same cast twice
- Bot FID is whitelisted — cannot be a valid tomato target
- Brigading detection: multiple throws from low-score accounts in a short 
  window get discounted
- Like/unlike cycling is handled via UNIQUE constraint on tomato_likes — 
  one row per (tomato_id, liker_fid), unlike deletes the row and 
  decrements like_count
- Webhook idempotency: check cast_hash doesn't already exist in tomatoes 
  before inserting or replying

---

## Tech Stack
- **Frontend/Mini app:** Vercel (Next.js / React)
- **Database:** Supabase (Postgres)
- **Webhook ingestion:** n8n (local, via Cloudflare tunnel)
- **Farcaster API:** Neynar
- **Token:** $SPLAT on Base — CA: 0x61b77bd4b9979c3428aff9d4d813f53b115c6b07
- **Bot:** Neynar managed signer (confirmation replies)

---

## Database Schema

```sql
users (
  fid              INTEGER PRIMARY KEY,
  username         TEXT,
  wallet_address   TEXT,
  splat_score      FLOAT DEFAULT 0.50,
  throws_today     INTEGER DEFAULT 0,
  quota_reset_at   TIMESTAMP,
  created_at       TIMESTAMP
)

tomatoes (
  id               UUID PRIMARY KEY,
  thrower_fid      INTEGER REFERENCES users(fid),
  target_fid       INTEGER REFERENCES users(fid),
  cast_hash        TEXT UNIQUE,         -- the 🍅 reply cast hash
  parent_hash      TEXT,                -- the cast being tomato'd
  like_count       INTEGER DEFAULT 0,
  is_valid         BOOLEAN DEFAULT false, -- true when like_count >= 1
  thrown_at        TIMESTAMP
)

tomato_likes (
  id               UUID PRIMARY KEY,
  tomato_id        UUID REFERENCES tomatoes(id),
  liker_fid        INTEGER REFERENCES users(fid),
  liked_at         TIMESTAMP,
  UNIQUE(tomato_id, liker_fid)          -- prevents like/unlike gaming
)

quota_log (
  id               UUID PRIMARY KEY,
  fid              INTEGER REFERENCES users(fid),
  date             DATE,
  throws_used      INTEGER DEFAULT 0,
  UNIQUE(fid, date)
)

score_history (
  id               UUID PRIMARY KEY,
  fid              INTEGER REFERENCES users(fid),
  splat_score      FLOAT,
  recorded_at      TIMESTAMP
)
```

---

## n8n Workflows

### Workflow 1 — Tomato Ingestion
Trigger: Neynar `cast.created` webhook
1. Filter: reply contains 🍅
2. Check: cast_hash not already in tomatoes (idempotency)
3. Check: thrower_fid is not target_fid (no self-tomatoes)
4. Check: target_fid is not bot FID (whitelist)
5. Check: throws_today < 5 for thrower
6. Check: (thrower_fid, parent_hash) not already in tomatoes (dedup)
7. Upsert thrower and target into users if not exists
8. Insert into tomatoes
9. Apply -0.03 to thrower splat_score
10. Apply weighted pelt penalty to target splat_score
11. Increment throws_today for thrower
12. Insert into quota_log
13. Bot replies confirming tomato logged

### Workflow 2A — Like Ingestion
Trigger: Neynar `reaction.created` webhook
1. Check: liked cast_hash exists in tomatoes
2. Check: liker_fid is not the thrower (no self-liking)
3. Upsert into tomato_likes
4. Increment like_count on tomato
5. If like_count == 1, set is_valid = true
6. Apply like recovery to thrower score:
   - 1st like: +0.03 (break even)
   - Each subsequent like: +0.015
7. Recalculate and update splat_score for thrower

### Workflow 2B — Unlike Handling
Trigger: Neynar `reaction.deleted` webhook
1. Check: unliked cast_hash exists in tomatoes
2. Delete row from tomato_likes
3. Decrement like_count on tomato
4. If like_count drops below 1, set is_valid = false
5. Reverse like recovery from thrower score
6. Recalculate and update splat_score for thrower

### Workflow 3 — Daily Reset
Trigger: Scheduled at midnight EST
1. Reset throws_today to 0 for all users
2. Update quota_reset_at timestamp

### Workflow 4 — Score Snapshots
Trigger: Scheduled every hour
1. Insert current splat_score for all users into score_history
2. Prune score_history rows older than 30 days to keep DB clean

---

## Score Leaderboard (Live Tracker)
Treat user splat scores like tokens with real-time % changes.

### Leaderboard columns:
- Rank
- Username + avatar
- Current splat score
- 12hr % change (green/red)
- 24hr % change (green/red)
- Total tomatoes received
- Total valid throws

### Sort options:
- By current score
- By biggest movers (24hr % change)
- By most pelted
- By best throwers (highest valid throw ratio)

### % change formula:
```
change % = ((current_score - past_score) / past_score) × 100
```
- 12hr: compare latest snapshot vs snapshot from 12hrs ago
- 24hr: compare latest snapshot vs snapshot from 24hrs ago

### Design notes:
- Green = score going up, Red = score going down (like a token chart)
- "Biggest movers" tab shows who's having the worst/best day on Farcaster
- Refreshes every few minutes for live feel

---

## Score Recalculation
- Scores recalculate on every like/unlike event
- Debounce: recalculate max once every few minutes per user to 
  handle high volume without performance issues
- New users auto-created (upserted) on first throw or first pelt

---

## Mini App UI (Vercel)
- **User profile** → splat score, tomatoes thrown, tomatoes received, score history chart
- **Live leaderboard** → sortable by score, biggest movers, most pelted, best throwers — with 12hr/24hr % change displayed like token prices
- **Cast lookup** → see how many tomatoes a specific cast has
- **Score badge** → embeddable in casts

---

## Current State
- V1 source code and database unrecoverable (FC Studio)
- Clean build from scratch
- Neynar account + webhook already set up
- n8n running locally via Cloudflare tunnel
- $SPLAT token live on Base
- GitHub account ready for Vercel deployment

## Build Order
1. Create new GitHub repo (tomatr)
2. Drop PLANNING.md in repo root
3. Set up Supabase project with schema above
4. Scaffold Next.js React app, deploy to Vercel
5. Wire n8n workflows to Supabase DB
6. Build mini app UI (leaderboard, profile, cast lookup)
7. Re-register Neynar webhook pointing to n8n Cloudflare tunnel URL
