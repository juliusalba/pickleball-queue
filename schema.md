# Pickleball Court Queue System — Data Model

## Entities

### Court
- `id` (string, UUID)
- `name` (string, e.g. "Court 1")
- `status` (enum: available | occupied | maintenance)
- `current_session_id` (FK → Session | null)

### Session
- `id` (string, UUID)
- `court_id` (FK → Court)
- `date` (string, YYYY-MM-DD)
- `start_time` (string, HH:MM)
- `end_time` (string, HH:MM)
- `type` (enum: open_play | reserved | league | tournament)
- `notes` (string, optional)
- `created_at` (ISO timestamp)

### Booking
- `id` (string, UUID)
- `session_id` (FK → Session)
- `player_name` (string)
- `group_size` (int, 1-4, default 1)
- `status` (enum: confirmed | checked_in | no_show | cancelled)
- `booked_at` (ISO timestamp)

### QueueEntry
- `id` (string, UUID)
- `session_id` (FK → Session) — which session they're waiting for
- `player_name` (string)
- `group_size` (int, 1-4)
- `position` (int, auto-assigned FIFO)
- `status` (enum: waiting | called | served | cancelled)
- `joined_at` (ISO timestamp)
- `called_at` (ISO timestamp | null)

---

## Session Types & "Open Play with Queuing"

### Session Types
| Type | Description |
|------|-------------|
| `open_play` | Open to anyone; queue managed by staff |
| `reserved` | Specific players pre-booked; no queue |
| `league` | Organized league play |
| `tournament` | Tournament block |

### "Open Play with Queuing" Metric
A session is counted as **"open play with queuing"** when:
- `session.type = 'open_play'` AND
- `session` has at least 1 `QueueEntry` with `status = 'waiting'`

A session is **"fully booked"** when:
- `session.type = 'reserved'` AND all 4 player slots are filled
- OR `session.type = 'open_play'` with no queue and all 4 slots filled

---

## Business Rules

1. **Court capacity per session**: Max 4 players (doubles)
2. **Queue FIFO**: Queue entries served in position order
3. **Session duration**: 1 hour (configurable)
4. **Queue → Court assignment**: Staff clicks "Call Next" to move queue entry to a court session
5. **Check-in**: Players can be checked in; no-shows free up slots
6. **Daily view**: Staff sees all sessions for the day across both courts
