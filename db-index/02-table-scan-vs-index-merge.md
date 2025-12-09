# Why can a table scan be better than index merge?

Are you sure about the random I/O? Please specifically explain. Also give me a walkthrough example of the decision framework using a meeting scheduler app.

---

# Part 1: Why Table Scan Can Beat Index Merge

## The I/O Story

### Sequential I/O (Table Scan)

```
┌─────────────────────────────────────────────────────────────────┐
│  DISK LAYOUT - Table Pages (stored sequentially)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Page 1    Page 2    Page 3    Page 4    Page 5    Page 6      │
│  ┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐          │
│  │████│──▶│████│──▶│████│──▶│████│──▶│████│──▶│████│          │
│  └────┘   └────┘   └────┘   └────┘   └────┘   └────┘          │
│                                                                 │
│  Disk head moves: ────────────────────────────▶                 │
│  (one continuous sweep)                                         │
│                                                                 │
│  Speed: ~100-200 MB/s (HDD) or ~500+ MB/s (SSD)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Table scan reads pages in order:**
- OS can prefetch next pages (read-ahead)
- Disk head moves smoothly in one direction
- Very efficient bulk reading

---

### Random I/O (Index Merge + Row Fetch)

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Read Index A (firstname = 'James')                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Index A returns row IDs: [23, 156, 892, 1547, 3921, ...]       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: Read Index B (age = 30)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Index B returns row IDs: [45, 156, 723, 1547, 2845, ...]       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: Intersect row IDs                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Matching IDs: [156, 1547]  (CPU work to merge sorted lists)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: Fetch actual rows (THE KILLER STEP)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Row 156 is on Page 12                                          │
│  Row 1547 is on Page 97                                         │
│                                                                 │
│  Page 1    Page 12   Page 45   Page 97   Page 120              │
│  ┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐                   │
│  │    │   │▓▓▓▓│   │    │   │▓▓▓▓│   │    │                   │
│  └────┘   └────┘   └────┘   └────┘   └────┘                   │
│              ▲                 ▲                                │
│              │                 │                                │
│              └────── JUMP ─────┘                                │
│                                                                 │
│  Disk head moves: ↗ ↘ ↗ ↘ ↗ (jumping around!)                  │
│                                                                 │
│  Speed: ~100-200 IOPS (HDD) = only 100-200 rows/second!        │
│         ~10,000+ IOPS (SSD) = better but still slower          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Math That Kills Index Merge

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: 1,000,000 row table, query returns 50,000 rows       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TABLE SCAN:                                                    │
│  ───────────                                                    │
│  • Table size: ~100 MB (1M rows × 100 bytes)                    │
│  • Sequential read speed: 200 MB/s                              │
│  • Time: 100 MB ÷ 200 MB/s = 0.5 seconds                        │
│                                                                 │
│  INDEX MERGE + ROW FETCH:                                       │
│  ────────────────────────                                       │
│  • Index A lookup: ~10ms                                        │
│  • Index B lookup: ~10ms                                        │
│  • Merge operation: ~5ms                                        │
│  • Fetch 50,000 rows randomly:                                  │
│    - HDD: 50,000 ÷ 150 IOPS = 333 seconds (5+ minutes!)        │
│    - SSD: 50,000 ÷ 10,000 IOPS = 5 seconds                     │
│                                                                 │
│  WINNER: Table scan (even on SSD for this case)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Tipping Point

Index is better when selecting **small percentage** of rows:

| Rows Selected | % of Table | Best Approach |
|---------------|------------|---------------|
| 10 | 0.001% | Index |
| 1,000 | 0.1% | Index |
| 10,000 | 1% | Depends |
| 50,000 | 5% | Usually table scan |
| 100,000+ | 10%+ | Definitely table scan |

**Rule of thumb:** If query returns >5-10% of table, table scan often wins.

---

# Part 2: Meeting Scheduler App - Full Walkthrough

## Step 1: Define the Schema

```sql
-- Core tables for a meeting scheduler
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE meetings (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    organizer_id INT NOT NULL REFERENCES users(id),
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    room_id INT,
    status VARCHAR(20) DEFAULT 'scheduled', -- scheduled, cancelled, completed
    is_recurring BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE attendees (
    id SERIAL PRIMARY KEY,
    meeting_id INT NOT NULL REFERENCES meetings(id),
    user_id INT NOT NULL REFERENCES users(id),
    response VARCHAR(20) DEFAULT 'pending', -- pending, accepted, declined, tentative
    responded_at TIMESTAMP,
    UNIQUE(meeting_id, user_id)
);

CREATE TABLE rooms (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    capacity INT,
    floor VARCHAR(20),
    has_video BOOLEAN DEFAULT FALSE
);
```

---

## Step 2: Identify Query Patterns

Let's list the **most common queries** in a meeting scheduler:

```
┌────┬─────────────────────────────────────────────────────────────┬───────────┐
│ #  │ Query Pattern                                               │ Frequency │
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q1 │ Get user's upcoming meetings                                │ VERY HIGH │
│    │ WHERE attendee.user_id = ? AND meeting.start_time > NOW()   │ (every    │
│    │ ORDER BY start_time                                         │  page load│
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q2 │ Check room availability for time range                      │ HIGH      │
│    │ WHERE room_id = ? AND start_time < ? AND end_time > ?       │ (booking) │
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q3 │ Find user's meetings on specific date                       │ HIGH      │
│    │ WHERE attendee.user_id = ? AND DATE(start_time) = ?         │ (calendar)│
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q4 │ Get all attendees for a meeting                             │ HIGH      │
│    │ WHERE meeting_id = ?                                        │ (details) │
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q5 │ Find pending invitations for user                           │ MEDIUM    │
│    │ WHERE user_id = ? AND response = 'pending'                  │ (notif.)  │
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q6 │ Get meetings organized by user                              │ MEDIUM    │
│    │ WHERE organizer_id = ? ORDER BY start_time DESC             │ (my mtgs) │
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q7 │ Search meetings by title                                    │ LOW       │
│    │ WHERE title ILIKE '%keyword%'                               │ (search)  │
├────┼─────────────────────────────────────────────────────────────┼───────────┤
│ Q8 │ Find users by email (login/invite)                          │ HIGH      │
│    │ WHERE email = ?                                             │ (auth)    │
└────┴─────────────────────────────────────────────────────────────┴───────────┘
```

---

## Step 3: Analyze Each Query

### Q1: User's Upcoming Meetings (MOST CRITICAL)

```sql
-- The actual query
SELECT m.*, a.response
FROM meetings m
JOIN attendees a ON m.id = a.meeting_id
WHERE a.user_id = 123 
  AND m.start_time > NOW()
  AND m.status = 'scheduled'
ORDER BY m.start_time
LIMIT 20;
```

**Analysis:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Query Q1 Breakdown                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tables involved: attendees → meetings (JOIN)                   │
│                                                                 │
│  Filter columns:                                                │
│    • attendees.user_id = ?     (high selectivity - one user)    │
│    • meetings.start_time > ?   (range - future dates)           │
│    • meetings.status = ?       (low selectivity - most are      │
│                                 'scheduled')                    │
│                                                                 │
│  Order by: meetings.start_time                                  │
│                                                                 │
│  RECOMMENDED INDEXES:                                           │
│  ─────────────────────                                          │
│  1. attendees(user_id, meeting_id)  -- find user's meetings     │
│  2. meetings(start_time) WHERE status = 'scheduled'             │
│     OR meetings(status, start_time) -- composite                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Best index for attendees:**
```sql
-- This index serves: finding all meetings for a user
CREATE INDEX idx_attendees_user_meeting 
ON attendees(user_id, meeting_id);
```

**Best index for meetings:**
```sql
-- Composite: status first (equality), then start_time (range + sort)
CREATE INDEX idx_meetings_status_start 
ON meetings(status, start_time);
```

---

### Q2: Room Availability Check

```sql
-- Check for conflicts
SELECT * FROM meetings
WHERE room_id = 5
  AND start_time < '2024-01-15 15:00:00'  -- new meeting end
  AND end_time > '2024-01-15 14:00:00'    -- new meeting start
  AND status = 'scheduled';
```

**Analysis:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Query Q2 Breakdown                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  This is an OVERLAP query (tricky!)                             │
│                                                                 │
│  Filters:                                                       │
│    • room_id = ?          (equality - high selectivity)         │
│    • start_time < ?       (range)                               │
│    • end_time > ?         (range)                               │
│    • status = 'scheduled' (equality - low selectivity)          │
│                                                                 │
│  PROBLEM: Can't use composite index efficiently for TWO ranges  │
│                                                                 │
│  RECOMMENDED INDEX:                                             │
│  ──────────────────                                             │
│  meetings(room_id, start_time) -- room first, then start_time   │
│                                                                 │
│  Why? Filter by room (equality), then scan start_time range     │
│  The end_time check happens after index lookup (filter)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```sql
CREATE INDEX idx_meetings_room_time 
ON meetings(room_id, start_time);
```

---

### Q4 & Q5: Attendees Queries

```sql
-- Q4: Get attendees for a meeting
SELECT u.*, a.response 
FROM attendees a
JOIN users u ON a.user_id = u.id
WHERE a.meeting_id = 456;

-- Q5: Pending invitations
SELECT m.*, a.* 
FROM attendees a
JOIN meetings m ON a.meeting_id = m.id
WHERE a.user_id = 123 
  AND a.response = 'pending';
```

**Analysis:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Attendees Table - Multiple Access Patterns                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Q4: WHERE meeting_id = ?                                       │
│  Q5: WHERE user_id = ? AND response = ?                         │
│                                                                 │
│  NOTICE: These queries access attendees DIFFERENTLY             │
│                                                                 │
│  Option A: Two single-column indexes                            │
│    idx_attendees_meeting(meeting_id)                            │
│    idx_attendees_user(user_id)                                  │
│    Problem: Q5 would need index merge or filter on response     │
│                                                                 │
│  Option B: Two composite indexes (BETTER)                       │
│    idx_attendees_meeting(meeting_id)        -- for Q4           │
│    idx_attendees_user_response(user_id, response)  -- for Q5    │
│                                                                 │
│  Option C: One clever composite (if storage is concern)         │
│    We already need: idx_attendees_user_meeting(user_id,         │
│                                                meeting_id)      │
│    Add: idx_attendees_meeting(meeting_id)                       │
│    For Q5: Use user_id index, filter response (acceptable if    │
│            pending invites are rare)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 4: Check for Redundancy & Consolidate

```
┌─────────────────────────────────────────────────────────────────┐
│  INDEX CONSOLIDATION REVIEW                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ATTENDEES TABLE:                                               │
│  ────────────────                                               │
│  Needed for Q1: (user_id, meeting_id)                           │
│  Needed for Q4: (meeting_id)                                    │
│  Needed for Q5: (user_id, response)                             │
│                                                                 │
│  FINAL INDEXES:                                                 │
│  • idx_attendees_user_meeting(user_id, meeting_id) ✓            │
│    └── Also covers queries on just user_id                      │
│  • idx_attendees_meeting(meeting_id) ✓                          │
│  • idx_attendees_user_response(user_id, response) ?             │
│    └── SKIP if pending invitations are rare (filter is OK)      │
│                                                                 │
│  MEETINGS TABLE:                                                │
│  ───────────────                                                │
│  Needed for Q1: (status, start_time)                            │
│  Needed for Q2: (room_id, start_time)                           │
│  Needed for Q6: (organizer_id, start_time)                      │
│                                                                 │
│  FINAL INDEXES:                                                 │
│  • idx_meetings_status_start(status, start_time) ✓              │
│  • idx_meetings_room_time(room_id, start_time) ✓                │
│  • idx_meetings_organizer(organizer_id, start_time) ✓           │
│                                                                 │
│  USERS TABLE:                                                   │
│  ────────────                                                   │
│  • email already has UNIQUE constraint (automatic index) ✓      │
│  • No additional indexes needed                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 5: Final Index Recommendations

```sql
-- ============================================================
-- MEETING SCHEDULER APP - RECOMMENDED INDEXES
-- ============================================================

-- USERS: email already indexed via UNIQUE constraint

-- MEETINGS: 3 composite indexes for different access patterns
CREATE INDEX idx_meetings_status_start 
ON meetings(status, start_time);

CREATE INDEX idx_meetings_room_time 
ON meetings(room_id, start_time) 
WHERE status = 'scheduled';  -- Partial index (PostgreSQL)

CREATE INDEX idx_meetings_organizer 
ON meetings(organizer_id, start_time DESC);

-- ATTENDEES: 2 indexes for the two main access patterns
CREATE INDEX idx_attendees_user_meeting 
ON attendees(user_id, meeting_id);

CREATE INDEX idx_attendees_meeting 
ON attendees(meeting_id);

-- ROOMS: Probably no index needed (small table, rarely queried alone)
```

---

## Summary: Why NOT Index Merge Here?

```
┌─────────────────────────────────────────────────────────────────┐
│  WHY COMPOSITE INDEXES WIN FOR MEETING SCHEDULER                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SCENARIO: Q1 - Get user's upcoming meetings                    │
│                                                                 │
│  BAD APPROACH (Index Merge):                                    │
│  ───────────────────────────                                    │
│  • Index on attendees(user_id)                                  │
│  • Index on meetings(start_time)                                │
│  • Index on meetings(status)                                    │
│                                                                 │
│  Execution:                                                     │
│  1. Find all meeting_ids for user_id=123 → 500 meetings         │
│  2. Find all meeting_ids where start_time > now → 100,000       │
│  3. Find all meeting_ids where status='scheduled' → 90,000      │
│  4. Intersect: 500 ∩ 100,000 ∩ 90,000 → 200 meetings            │
│  5. Fetch 200 rows randomly from meetings table                 │
│  6. Sort by start_time                                          │
│                                                                 │
│  Problems: Lots of index reads, intersection work, random I/O   │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  GOOD APPROACH (Composite Indexes):                             │
│  ──────────────────────────────────                             │
│  • Index on attendees(user_id, meeting_id)                      │
│  • Index on meetings(status, start_time)                        │
│                                                                 │
│  Execution:                                                     │
│  1. Find meeting_ids for user_id=123 → 500 meetings (1 lookup)  │
│  2. For each, probe meetings index on (status, start_time)      │
│     → Already filtered AND sorted!                              │
│  3. No additional sorting needed                                │
│                                                                 │
│  Result: Minimal I/O, no merge, pre-sorted output               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

| Query Type | Index Strategy |
|------------|----------------|
| `WHERE a = ?` | Single column or leftmost of composite |
| `WHERE a = ? AND b = ?` | Composite `(a, b)` |
| `WHERE a = ? AND b > ?` | Composite `(a, b)` — equality first! |
| `WHERE a = ? ORDER BY b` | Composite `(a, b)` |
| `WHERE a = ? AND b = ? ORDER BY c` | Composite `(a, b, c)` |
| `WHERE a > ? AND b > ?` | Pick more selective column, filter other |

**Golden Rule:** One well-designed composite index beats multiple single-column indexes merged together.

