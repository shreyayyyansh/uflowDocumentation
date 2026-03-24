# UrbanConnect Walkthrough — Part 2: Server-Side Controllers, Services & API

> **Prerequisites**: Read [Part 1](file:///e:/sankalp/UrbanFlow/urbanconnect_walkthrough_part1.md) first — you need to understand the data models before diving into the logic.

---

## 1. API Route Map — Every Endpoint at a Glance

All UrbanConnect routes are mounted under `/api/urbanconnect/` via [urbanconnect.route.js](file:///e:/sankalp/UrbanFlow/server/src/routes/urbanconnect.route.js).

| Method | Route | Controller | Purpose |
|---|---|---|---|
| `GET` | `/fetchQuestion` | `fetchQuestions` | Paginated feed (cursor-based) |
| `GET` | `/fetchQuestion/:id` | `fetchQuestionById` | Single post + comments |
| `POST` | `/ask` | `createQuestion` | Create a new post |
| `POST` | `/upload` | `uploadImage` | Upload image to Cloudinary |
| `POST` | `/comment` | `postComment` | Add comment/reply |
| `GET` | `/comments` | `getComments` | Paginated top-level comments |
| `GET` | `/replies` | `getReplies` | Paginated replies to a comment |
| `PATCH` | `/votes` | `patchVote` | Vote on a comment |
| `PATCH` | `/questionVotes` | `patchQuestionVote` | Vote on a post |
| `POST` | `/search` | `searchQuestions` | Regex text search |
| `POST` | `/save` | `toggleSave` | Bookmark/unbookmark comment |
| `GET` | `/profile` | `fetchUserProfile` | Full user profile data |
| `GET` | `/authorities/:city` | `fetchAuthoritiesByCity` | List gov officials by city |
| `POST` | `/cluster-check` | `clusterCheck` | Check if a post forms a cluster |
| `GET` | `/clusters` | `getClusters` | List all emerging issue clusters |
| `POST` | `/syndicate-civic` | `civicSyndication` | Bridge a CivicConnect report |
| `POST` | `/agent/chat` | (proxy) | Forward to Python AI assistant |
| `GET` | `/test-pulse` | `fetchAndAnalyzeCityPulse` | Manual City Pulse trigger |

**Additional routes on separate routers**:

| Router | Method | Route | Controller |
|---|---|---|---|
| `announcement.routes.js` | `GET` | `/api/announcements` | `getAnnouncements` |
| `announcement.routes.js` | `POST` | `/api/announcements/search` | `searchAnnouncementsForRAG` |
| `cityPulse.routes.js` | `GET` | `/api/city-pulse/latest` | (inline) |

---

## 2. The Main Controller — `urbanconnect.controller.js`

**File**: [urbanconnect.controller.js](file:///e:/sankalp/UrbanFlow/server/src/controllers/urbanconnect.controller.js) (822 lines)

This is the largest file in the UrbanConnect feature. Let's break down every exported function.

---

### 2.1 `fetchQuestions` — The Feed Engine

**Route**: `GET /api/urbanconnect/fetchQuestion`

**What it does**: Returns a paginated list of posts for the main feed.

**Step-by-step flow**:

1. **Read query params**: `after` (cursor), `limit` (default 20), `email` (current user)
2. **Check Redis cache**: Key format `urbanconnect:questions:after_{X}:limit_{Y}:user_{Z}`
   - If cache hit → return immediately (60s TTL)
3. **Query MongoDB**: `Question.find()` with cursor-based pagination (`_id < after`)
   - `.populate('author', 'username avatar email')` — joins User data
   - `.populate('taggedAuthorities')` — joins Administration data
   - `.sort({ _id: -1 })` — newest first
   - `.limit(limit)` — page size
   - `.lean()` — returns plain objects (faster, no Mongoose document overhead)
4. **Fetch user's votes**: If authenticated, query `QuestionVote` for this user's votes on returned posts → build a vote `Map`
5. **Enrich each post**: For each question, in parallel:
   - Count comments: `Comment.countDocuments({ questionId })`
   - Build author info (name, handle, avatar)
   - If it's a CivicConnect post with no real author → spoof to "CivicConnect AI Verified"
   - Add `timeAgo`, `commentCount`, `userVote`
6. **Build response**: `{ data: [...], nextCursor, hasMore }`
7. **Store in Redis**: Cache for 60 seconds

**Interview Q: Why cursor-based pagination instead of `skip/limit`?**
> With `skip(100)`, MongoDB must scan and discard 100 documents. For page 1000, it scans 10,000 docs. Cursor-based pagination uses `_id < lastSeenId` which leverages the `_id` index for O(log n) seeks regardless of page number. This is critical for feeds with millions of posts.

**Interview Q: Why `.lean()`?**
> Mongoose documents are heavy objects with change tracking, getters/setters, and validators. `.lean()` returns plain JavaScript objects — ~5x faster for read-only operations like feed rendering.

---

### 2.2 `fetchQuestionById` — Post Detail

**Route**: `GET /api/urbanconnect/fetchQuestion/:id`

Fetches a single post with all its top-level comments. Used when you tap on a post in the feed.

**Flow**:
1. `Question.findById(id).populate('author').populate('taggedAuthorities')`
2. Fetch top-level comments: `Comment.find({ questionId, parentId: null }).sort({ createdAt: -1 })`
3. Enrich with author info, timeAgo, CivicConnect spoofing
4. Return enriched post + comments

---

### 2.3 `createQuestion` — Post Creation + AI Pipeline

**Route**: `POST /api/urbanconnect/ask`

This is one of the most important functions. It creates a post AND triggers the AI analysis pipeline.

**Flow**:

```
Client sends POST /ask → Server creates Question → Response sent to client
                                                  ↓ (fire-and-forget)
                                          Python /analyze-post
                                                  ↓
                                          Update Question with AI results
```

**Detailed steps**:

1. **Extract user info** from request body (email, name, avatar)
2. **Find or create User**: `User.findOne({ email })` — if not found, create a new user with Auth0 info. If found but missing avatar, auto-heal.
3. **Create Question document** in MongoDB
4. **Invalidate Redis cache**: Delete all keys matching `urbanconnect:questions:*`
5. **Send 201 response** to client immediately
6. **Fire-and-forget AI analysis** (async IIFE):
   - Determine city from tagged authorities (default: "Prayagraj")
   - Build payload: `{ postId, title, description, imageUrls, city }`
   - `POST` to Python server at `/analyze-post`
   - On success, update the Question with: `sentiment`, `sentimentScore`, `urgency`, `postType`, `isMisinformation`, `contextNote`, `clusterId`, `embedding`
   - On failure, log and silently swallow error

**Interview Q: What is the "fire-and-forget" pattern and why use it here?**
> The user shouldn't wait 3-5 seconds for AI analysis before seeing "Post created!" confirmation. So we immediately respond with 201, then launch an async task that updates the post later. The UI can show AI badges when they become available (on next feed refresh). This trades consistency for latency.

**Interview Q: What if the fire-and-forget fails?**
> The post still exists — it just won't have AI analysis data. The `aiAnalysis` fields will remain `null`. The system is designed to be **gracefully degraded** — the feed works fine with or without AI data.

---

### 2.4 `uploadImage` — Cloudinary Upload

**Route**: `POST /api/urbanconnect/upload`

Receives a multipart form upload (handled by `multer` middleware), uploads the buffer to Cloudinary, and returns the URL.

```javascript
const imageUrl = await uploadToCloudinary(req.file.buffer);
res.status(200).json({ url: imageUrl });
```

---

### 2.5 `postComment` — Add Comment or Reply

**Route**: `POST /api/urbanconnect/comment`

**Flow**:
1. Find or create User by email
2. Create `Comment` document (sets `parentId` if it's a reply)
3. If it's a reply, increment parent's `replyCount` atomically: `$inc: { replyCount: 1 }`
4. Invalidate Redis cache
5. Return the new comment

**Interview Q: Why `$inc` instead of reading, incrementing, and saving?**
> `$inc` is an **atomic operation** in MongoDB. If two users reply simultaneously, both `$inc` operations will correctly increment by 1 each. If you read → increment → save, you get a race condition where one update overwrites the other (lost update problem).

---

### 2.6 `getComments` — Paginated Top-Level Comments

**Route**: `GET /api/urbanconnect/comments?questionId=X&after=Y&limit=10`

Returns top-level comments (where `parentId: null`) for a question, sorted by `votes` first (hottest), then `createdAt`.

For each comment, it also enriches with:
- `userVote`: The current user's vote (from `Vote` collection)
- `saved`: Whether the current user has bookmarked this comment (from `Saved` collection)

---

### 2.7 `getReplies` — Paginated Nested Replies

**Route**: `GET /api/urbanconnect/replies?parentId=X&after=Y&limit=5`

Similar to `getComments`, but fetches replies to a specific comment. Sorted chronologically (`createdAt: 1`) since nested replies should appear in order.

---

### 2.8 `patchVote` — Comment Voting Logic

**Route**: `PATCH /api/urbanconnect/votes`

The voting logic handles 4 scenarios:

| Scenario | Action |
|---|---|
| No existing vote + `value ≠ 0` | Create new `Vote` doc, `delta = value` |
| Existing vote + `value = 0` | Delete vote doc, `delta = -existingValue` |
| Existing vote + different value | Update vote, `delta = newValue - oldValue` |
| Existing vote + same value | No-op, return immediately |

After computing `delta`, updates the comment's denormalized `votes` counter:
```javascript
Comment.findByIdAndUpdate(commentId, { $inc: { votes: delta } })
```

---

### 2.9 `patchQuestionVote` — Post Voting + Civic Syndication Proxy

**Route**: `PATCH /api/urbanconnect/questionVotes`

Same voting logic as `patchVote` but for posts. **Critical extra feature**: if the upvoted post is a CivicConnect report (`isCivicReport: true`), the vote is **proxied back** to the original CivicConnect report system.

```javascript
// If it's a civic report and user upvoted
if (updatedQuestion?.isCivicReport && updatedQuestion?.reportId && delta === 1) {
  // Proxy the upvote to CivicConnect's report update endpoint
  await axios.post(`${base}/api/reports/update${category}Reports`, {
    userId: 'urban_connect_voter',
    email: user.email,
    reportId: updatedQuestion.reportId,
    geohash: updatedQuestion.geohash || ""
  });
}
```

**Interview Q: Why proxy the vote to CivicConnect?**
> This creates a **bidirectional feedback loop**. When users upvote a syndicated civic report on UrbanConnect, it increases the priority/support count on the original CivicConnect report too. This means UrbanConnect engagement directly amplifies CivicConnect issues, creating a unified civic engagement ecosystem.

---

### 2.10 `toggleSave` — Bookmark/Unbookmark Comments

**Route**: `POST /api/urbanconnect/save`

Simple toggle: if a `Saved` doc exists for this user+comment, delete it. If not, create it.

---

### 2.11 `searchQuestions` — Text Search

**Route**: `POST /api/urbanconnect/search`

Uses MongoDB regex for case-insensitive text search across `title`, `description`, and `tags` fields:

```javascript
const regex = new RegExp(search, 'i');
Question.find({
  $or: [
    { title: { $regex: regex } },
    { description: { $regex: regex } },
    { tags: { $regex: regex } }
  ]
})
```

**Interview Q: Why not use MongoDB text index or Atlas Search?**
> For a small-to-medium dataset, regex search is simpler and sufficient. However, it won't scale well (no index utilization for regex). For production with millions of posts, you'd migrate to MongoDB Atlas Full-Text Search or Elasticsearch.

---

### 2.12 `fetchUserProfile` — Full Profile Aggregation

**Route**: `GET /api/urbanconnect/profile?email=X&name=Y&picture=Z`

The most data-heavy endpoint. Returns **four** datasets for a user:

1. **`posts`**: All questions created by this user + enriched with commentCount, userVote
2. **`replies`**: All comments written by this user
3. **`likes`**: All questions this user upvoted (via QuestionVote)
4. **`saved`**: All bookmarked comments with parent question context

**Auto-bootstrap**: If the user doesn't exist in MongoDB but `name` is provided, it auto-creates their profile. This handles first-time Auth0 logins.

**Auto-heal**: If the user exists but has no avatar and one is provided, it updates the avatar.

---

### 2.13 `fetchAuthoritiesByCity` — Authority Directory

**Route**: `GET /api/urbanconnect/authorities/:city`

Case-insensitive search for government officials in a given city:
```javascript
Administration.find({ city: { $regex: new RegExp('^' + city + '$', 'i') } })
```

---

## 3. Sub-Controllers

### 3.1 Cluster Controller — [cluster.controller.js](file:///e:/sankalp/UrbanFlow/server/src/controllers/urbanconnect/cluster.controller.js)

#### `clusterCheck` — AI Issue Clustering

**Route**: `POST /api/urbanconnect/cluster-check`

Called by the Python AI agent after analyzing a post. Determines if a new post is part of an emerging pattern.

**Algorithm**:

```
1. Receive { postId, embedding, city }
2. Fetch all posts from last 12 hours that have embeddings
3. Compute cosine similarity between new post's embedding and each recent post
4. Filter posts with similarity >= 0.75 (CLUSTER_SIMILARITY_THRESHOLD)
5. If similar posts count >= 2 (MIN_SIZE - 1):
   a. Check if any similar post already has a clusterId
   b. If yes → reuse that clusterId
   c. If no → generate new cluster_${uuid}
   d. Tag all unclustered similar posts with the clusterId
   e. If cluster hits size 3 → fire-and-forget AI summary generation
6. Return { clusterId, clusterSize } or { clusterId: null }
```

**Constants**:
```javascript
CLUSTER_SIMILARITY_THRESHOLD = 0.75  // Min cosine similarity to be "related"
CLUSTER_MIN_SIZE = 3                 // Min posts to form a cluster
CLUSTER_TIME_WINDOW_HOURS = 12       // Only look at recent posts
```

**Cosine Similarity** (helper function):
```javascript
function cosineSimilarity(a, b) {
  // dot(a,b) / (||a|| * ||b||)
  // Returns 0 if vectors are orthogonal, 1 if identical
}
```

**Interview Q: Why cosine similarity and not Euclidean distance?**
> Cosine similarity measures the **angle** between two vectors, ignoring magnitude. This is ideal for text embeddings because we care about semantic direction, not vector length. A post about "water supply problem" and another about "drinking water shortage" have similar directions even if one is longer/shorter.

#### `getClusters` — List All Clusters

**Route**: `GET /api/urbanconnect/clusters`

Returns all `Cluster` documents enriched with their `postCount` (how many Questions reference that clusterId).

---

### 3.2 City Pulse Controller — [cityPulse.controller.js](file:///e:/sankalp/UrbanFlow/server/src/controllers/urbanconnect/cityPulse.controller.js)

#### `fetchAndAnalyzeCityPulse` — The Entire Scrape-to-Analysis Pipeline

This is the most complex controller. It runs as a **daily cron job** (2 AM) or can be triggered manually.

**Pipeline**:

```
  ┌──────────────────── Step 1: Parallel Data Collection ─────────────────────┐
  │                                                                           │
  │  [Tavily API]  →  15 news/social search results                          │
  │  [Apify Twitter] → 50 tweets about city                                  │
  │  [Apify Reddit]  → 20 posts from city subreddits                        │
  │  [Apify News]    → 10 Google News articles                               │
  │                                                                           │
  └────────────────────────────┬──────────────────────────────────────────────┘
                               ↓
  ┌──────── Step 2: Merge & Normalize (up to 95 items) ──────┐
  │  Normalize all sources to: { source, text, url }          │
  └────────────────────────────┬──────────────────────────────┘
                               ↓
  ┌──────── Step 3: Fetch Historical Context ────────────────┐
  │  Get yesterday's SocialPulse from MongoDB                 │
  │  Extract: previous_trust_score, previous_summary          │
  └────────────────────────────┬──────────────────────────────┘
                               ↓
  ┌──────── Step 4: AI Analysis (Python /analyze-pulse) ─────┐
  │  Send to Python: { city, posts[], previous_data }         │
  │  Receive: executive_summary, sentiment, trends, issues    │
  └────────────────────────────┬──────────────────────────────┘
                               ↓
  ┌──────── Step 5: Save to MongoDB ─────────────────────────┐
  │  Create new SocialPulse document                          │
  └──────────────────────────────────────────────────────────┘
```

**Data sources used**:
- **Tavily API**: Advanced web search for "latest civic issues municipal complaints Prayagraj"
- **Apify Twitter**: Scrapes tweets matching "(Prayagraj OR Allahabad) AND (municipal OR issue...)"
- **Apify Reddit**: Scrapes r/allahabad, r/prayagraj, r/india, r/uttarpradesh
- **Apify Google News**: "Prayagraj news today", "Allahabad municipal issues"

All 4 sources run in **parallel** via `Promise.all()`.

**Interview Q: Why use 4 different sources instead of just Twitter?**
> Diverse data sources provide a more accurate picture of public sentiment. Twitter may skew toward younger, tech-savvy users. Reddit provides deeper discussions. News articles provide verified facts. Tavily catches content from forums and blogs. Together they reduce sampling bias.

---

### 3.3 Announcement Controller — [announcement.controller.js](file:///e:/sankalp/UrbanFlow/server/src/controllers/urbanconnect/announcement.controller.js)

#### `getAnnouncements` — Read-Only Feed

**Route**: `GET /api/announcements?city=Prayagraj&limit=20`

Simple paginated query with `.populate('authority')`. Returns formatted announcements with `authorityName` and `authorityDepartment`.

#### `searchAnnouncementsForRAG` — Hybrid Vector + Text Search

**Route**: `POST /api/announcements/search`

This is the RAG (Retrieval-Augmented Generation) search endpoint used **internally by the Python AI agent** to fact-check posts.

**How it works**:

```
1. Receive { embedding, queryText, city }
2. Build MongoDB Atlas Aggregation Pipeline:
   a. $search with compound query:
      - Vector search: find semantically similar announcements (via embedding)
      - BM25 text search: find keyword matches in title/body/department
      - Filter: restrict to matching city
   b. $limit: 5 results
   c. $lookup: join Administration collection for authority details
   d. $unwind: flatten the join result
3. Return scored results
```

**Interview Q: What is "Hybrid Search" and why is it superior?**
> Pure vector search finds semantically similar content but may miss exact keywords. Pure BM25 keyword search finds exact matches but misses paraphrased content. **Hybrid search** combines both — MongoDB Atlas computes a **Reciprocal Rank Fusion (RRF)** score that merges vector and keyword rankings. This is the gold standard for RAG retrieval.

---

## 4. Services

### 4.1 CivicConnect Syndication — [civicSyndication.js](file:///e:/sankalp/UrbanFlow/server/src/services/civicSyndication.js)

**Purpose**: Bridges CivicConnect reports into UrbanConnect as posts. Called via `POST /api/urbanconnect/syndicate-civic`.

**Flow**:

```
CivicConnect creates report → HTTP POST to /syndicate-civic
                                       ↓
                               civicSyndication(payload)
                                       ↓
1. Guard: Ignore UNCERTAIN reports
2. Guard: Skip if already syndicated (Question.exists({ reportId }))
3. Find/map the reporter's UrbanConnect User (by email)
   - If no User found → use dummy ObjectId '000000000000000000000000'
4. Create Question with isCivicReport: true, reportId, reportCategory, etc.
5. Invalidate Redis cache
6. Send to Python AI: POST /analyze-civic
7. Update Question with AI results (sentiment, urgency, clusterId, embedding)
```

**Interview Q: What is the dummy ObjectId `000000000000000000000000`?**
> It's a "null object" pattern for MongoDB. When a CivicConnect reporter doesn't have an UrbanConnect account, we still need a valid `ObjectId` for the `author` field (which is required). Using a known dummy ID lets the controller later detect this and spoof the author as "CivicConnect AI Verified" in the feed display logic.

---

### 4.2 Status Sync — [syncReportStatus.js](file:///e:/sankalp/UrbanFlow/server/src/services/syncReportStatus.js)

Simple one-liner service that updates the `reportStatus` of a syndicated Question when CivicConnect changes the report's status (e.g., PENDING → ASSIGNED → RESOLVED):

```javascript
await Question.findOneAndUpdate({ reportId }, { reportStatus: status });
```

This keeps the UrbanConnect feed badges (`PENDING`, `ASSIGNED`, `IN REVIEW`, `RESOLVED`) in sync with CivicConnect.

---

## 5. Cron Jobs — [cronJobs.js](file:///e:/sankalp/UrbanFlow/server/src/utils/cronJobs.js)

The City Pulse pipeline runs automatically every day at **2:00 AM**:

```javascript
cron.schedule("0 2 * * *", async () => {
  await fetchAndAnalyzeCityPulse();
});
```

---

## 6. Supporting Routes

### 6.1 Announcement Routes — [announcement.routes.js](file:///e:/sankalp/UrbanFlow/server/src/routes/announcement.routes.js)

```
GET  /api/announcements          → getAnnouncements
POST /api/announcements/search   → searchAnnouncementsForRAG  (internal, for AI agent)
```

### 6.2 City Pulse Routes — [cityPulse.routes.js](file:///e:/sankalp/UrbanFlow/server/src/routes/cityPulse.routes.js)

```
GET /api/city-pulse/latest → Returns most recent SocialPulse document for "Prayagraj"
```

---

## 7. Request Lifecycle — Complete Example

Let's trace what happens when a user creates a post:

```
📱 User taps "Post" in create.jsx
        ↓
   1. Upload image → POST /api/urbanconnect/upload → Cloudinary → returns URL
        ↓
   2. Create post → POST /api/urbanconnect/ask
        ↓
   3. Server: Find/create User in MongoDB
        ↓
   4. Server: Question.create({ author, title, description, image, taggedAuthorities })
        ↓
   5. Server: Delete Redis cache keys (urbanconnect:questions:*)
        ↓
   6. Server: Respond 201 → Client navigates back to feed
        ↓ (async, fire-and-forget)
   7. Server: POST to Python /analyze-post with { postId, title, description, city }
        ↓
   8. Python AI: Analyze sentiment, urgency, misinformation, compute embedding
        ↓
   9. Python AI: POST to /api/urbanconnect/cluster-check with { postId, embedding, city }
        ↓
  10. Server: Cosine similarity check → cluster or not
        ↓
  11. Server: Update Question.aiAnalysis with results
        ↓
  12. Next feed refresh → User sees post with AI badges (if analysis completed in time)
```

---

> **Next**: [Part 3 — Client-Side Implementation & Data Flow](file:///e:/sankalp/UrbanFlow/urbanconnect_walkthrough_part3.md) covers every React Native screen, component, and the data flow between client and server.
