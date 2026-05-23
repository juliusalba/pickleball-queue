# Pickleball Court Queue System — Phase 2 Data Model

## Overview

- **Venue**: Ace Pickleball Club, Philippines
- **Storage**: `localStorage` via `STORAGE_KEY = 'pb_queue_v5'`
- **Prices**: PHP (₱)
- **App**: Single-file vanilla JS (`index.html`)

---

## Enums

### `session_type`
`open_play` | `reserved` | `league` | `tournament` | `clinic` | `private_lesson`

### `session_status`
`scheduled` | `check_in_open` | `in_progress` | `completed` | `cancelled`

### `member_type`
`guest` | `day_pass` | `monthly` | `annual` | `package`

### `package_type`
`10_pack` | `5_pack` | `day_pass` | `monthly`

### `package_status`
`active` | `expired` | `used_up`

### `payment_method`
`cash` | `gcash` | `maya` | `card`

### `payment_status`
`pending` | `paid` | `refunded`

### `booking_status`
`confirmed` | `checked_in` | `no_show` | `cancelled`

### `queue_status`
`waiting` | `notified` | `claimed` | `expired` | `cancelled` | `served`

### `court_status`
`available` | `occupied` | `maintenance`

### `court_surface`
`indoor_hard` | `outdoor_hard` | `outdoor_clay` | `indoor_court`

### `strike_type`
`late_cancel` | `no_show`

### `staff_role`
`owner` | `manager` | `front_desk` | `coach`

### `notification_type`
`waitlist_joined` | `slot_opened` | `claim_expired` | `booking_confirmed` | `reminder` | `no_show_recorded` | `strike_added` | `package_low` | `package_expired`

### `dupr_bucket`
`top` (4.0+) | `mid` (3.0–3.9) | `low` (below 3.0)

---

## Entities

### Court

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `name` | `string` | e.g. "Court 1" |
| `surface` | `court_surface` | Court surface type |
| `amenities` | `string[]` | e.g. `["lighting", "fans", "shade", "scoreboard"]` |
| `enabledSessionTypes` | `session_type[]` | Session types allowed on this court |
| `status` | `court_status` | Current availability |
| `current_session_id` | `string` (FK → Session) \| `null` | Active session |

---

### Player

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `name` | `string` | Full name |
| `phone` | `string` | Contact number |
| `email` | `string` \| `null` | Optional email |
| `dupr` | `number` | Skill rating (0.5 increments, 2.0–6.5) |
| `memberType` | `member_type` | Membership classification |
| `packageCredits` | `number` | Remaining credits (for package holders) |
| `packageExpiry` | `string` (YYYY-MM-DD) \| `null` | Current package expiration |
| `strikes` | `number` (0–3) | Current active strike count |
| `strikeHistory` | `Strike[]` | All past strikes (see Strike entity) |
| `emergencyContact` | `object` | `{ name: string, phone: string }` |
| `waiverSigned` | `boolean` | Liability waiver on file |
| `notes` | `string` | Staff notes |
| `sessionsPlayed` | `number` | Total sessions attended |
| `wins` | `number` | Match wins |
| `losses` | `number` | Match losses |
| `lastPlayed` | `string` (YYYY-MM-DD) \| `null` | Last session date |
| `memberSince` | `string` (YYYY-MM-DD) | Registration date |
| `created_at` | `ISO timestamp` | Record creation |
| `updated_at` | `ISO timestamp` | Last profile update |

---

### Session

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `court_id` | `string` (FK → Court) | Assigned court |
| `date` | `string` (YYYY-MM-DD) | Session date |
| `start_time` | `string` (HH:MM) | Start time |
| `end_time` | `string` (HH:MM) | End time |
| `type` | `session_type` | Session type |
| `status` | `session_status` | Current session status |
| `maxPlayers` | `number` | Max players (default 4 for doubles) |
| `pricePerPerson` | `number` (PHP) | Per-player price (for open play, clinic, league, tournament) |
| `pricePerCourt` | `number` (PHP) | Court rental price (for reserved, private lesson) |
| `creditCost` | `number` | Package credits consumed (1 for open play, 2 for clinic) |
| `checkInOpenAt` | `string` (HH:MM) | Time check-in opens (start_time - 30 min) |
| `notes` | `string` | Session notes |
| `created_at` | `ISO timestamp` | Record creation |
| `updated_at` | `ISO timestamp` | Last update |

#### Session Type Pricing Reference

| Type | Per Player (₱) | Per Court 1hr (₱) | Credits |
|------|----------------|-------------------|---------|
| `open_play` | 150 | — | 1 |
| `reserved` | — | 500 | — |
| `league` | 200 | — | 1 |
| `tournament` | 250 | — | — |
| `clinic` | 350 | — | 2 |
| `private_lesson` | — | 800 | — |

#### Session Status Transitions

```
scheduled → check_in_open (30 min before start)
check_in_open → in_progress (at start time)
in_progress → completed (manual or auto at end_time)
any → cancelled (staff action)
```

---

### Booking

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `session_id` | `string` (FK → Session) | Target session |
| `player_id` | `string` (FK → Player) | Booked player |
| `player_name` | `string` | Denormalized player name |
| `group_size` | `number` (1–4) | Number of players (default 1) |
| `status` | `booking_status` | Booking state |
| `payment_id` | `string` (FK → Transaction) \| `null` | Linked payment |
| `package_id` | `string` (FK → Package) \| `null` | Package used for this booking |
| `booked_at` | `ISO timestamp` | When booking was made |
| `checked_in_at` | `ISO timestamp` \| `null` | Check-in time |
| `cancelled_at` | `ISO timestamp` \| `null` | Cancellation time |
| `cancel_reason` | `string` \| `null` | Why cancelled |
| `no_show` | `boolean` | Marked as no-show |
| `created_at` | `ISO timestamp` | Record creation |
| `updated_at` | `ISO timestamp` | Last update |

---

### QueueEntry

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `session_id` | `string` (FK → Session) | Session they're waiting for |
| `player_id` | `string` (FK → Player) | Player in queue |
| `player_name` | `string` | Denormalized player name |
| `group_size` | `number` (1–4) | Group size |
| `position` | `number` | Auto-assigned FIFO position |
| `status` | `queue_status` | Queue entry state |
| `joined_at` | `ISO timestamp` | When joined queue |
| `notified_at` | `ISO timestamp` \| `null` | When slot notification sent |
| `claimed_at` | `ISO timestamp` \| `null` | When player claimed slot |
| `expired_at` | `ISO timestamp` \| `null` | When claim window expired |
| `called_at` | `ISO timestamp` \| `null` | When staff called this entry |
| `created_at` | `ISO timestamp` | Record creation |
| `updated_at` | `ISO timestamp` | Last update |

#### Queue Status Transitions

```
waiting → notified (slot opens, 15-min countdown starts)
notified → claimed (player claims within window)
notified → expired (claim window passes)
claimed → served (player assigned to court)
waiting → cancelled (player leaves waitlist)
notified → cancelled (player leaves during countdown)
```

---

### Package

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `player_id` | `string` (FK → Player) | Package owner |
| `player_name` | `string` | Denormalized player name |
| `type` | `package_type` | Package type |
| `credits` | `number` | Remaining credits |
| `maxCredits` | `number` | Starting credits (10, 5, or unlimited) |
| `price` | `number` (PHP) | Amount paid |
| `purchasedAt` | `ISO timestamp` | Purchase date |
| `expiresAt` | `string` (YYYY-MM-DD) | Expiration date |
| `status` | `package_status` | Current status |
| `created_at` | `ISO timestamp` | Record creation |
| `updated_at` | `ISO timestamp` | Last update |

#### Package Type Reference

| Type | Credits | Price (₱) | Expiry |
|------|---------|-----------|--------|
| `10_pack` | 10 | 2,500 | 90 days |
| `5_pack` | 5 | 1,400 | 60 days |
| `day_pass` | ∞ | 300 | Same day |
| `monthly` | ∞ | 1,500/mo | 30 days |

#### Credits Behavior
- Open Play: 1 credit/session
- Clinic: 2 credits
- League: 1 credit
- Warning at 2 credits remaining
- Unused credits carry over within expiry window

---

### Strike

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `player_id` | `string` (FK → Player) | Penalized player |
| `player_name` | `string` | Denormalized player name |
| `type` | `strike_type` | `late_cancel` or `no_show` |
| `date` | `string` (YYYY-MM-DD) | When the offense occurred |
| `session_id` | `string` (FK → Session) | Related session |
| `fee` | `number` (PHP) | Fee charged (30 late cancel, 50 no-show) |
| `fee_paid` | `boolean` | Whether fee has been collected |
| `resolved` | `boolean` | Whether strike has expired/reset |
| `resolved_at` | `ISO timestamp` \| `null` | When resolved (auto after reset period) |
| `created_at` | `ISO timestamp` | Record creation |

#### Strike Rules
- Late cancel (< 2 hrs before session): 1 strike + ₱30 fee
- No-show: 2 strikes + ₱50 fee
- 3 strikes total → blocked from online booking (front desk only)
- Strikes reset after 30 days (configurable)
- Fines auto-deducted or added to next booking

---

### Transaction

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `booking_id` | `string` (FK → Booking) \| `null` | Linked booking |
| `package_id` | `string` (FK → Package) \| `null` | Linked package (if package purchase) |
| `player_id` | `string` (FK → Player) | Paying player |
| `player_name` | `string` | Denormalized player name |
| `session_id` | `string` (FK → Session) \| `null` | Related session |
| `session_name` | `string` | Denormalized session label |
| `court_name` | `string` | Denormalized court name |
| `date` | `string` (YYYY-MM-DD) | Transaction date |
| `time` | `string` (HH:MM) | Transaction time |
| `playerCount` | `number` | Players covered |
| `ratePerPerson` | `number` (PHP) | Rate applied |
| `subtotal` | `number` (PHP) | Before fees |
| `processingFee` | `number` (PHP) | Payment processing fee |
| `total` | `number` (PHP) | Final amount |
| `paymentMethod` | `payment_method` | How paid |
| `referenceNumber` | `string` \| `null` | GCash/Maya reference |
| `status` | `payment_status` | Payment state |
| `staffProcessedBy` | `string` | Staff member who processed |
| `createdAt` | `ISO timestamp` | Record creation |
| `paidAt` | `ISO timestamp` \| `null` | When payment confirmed |

---

### Notification

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `type` | `notification_type` | Notification category |
| `title` | `string` | Short title |
| `message` | `string` | Full message text |
| `forPlayer` | `string` (FK → Player) \| `null` | Target player (null = staff/owner) |
| `forStaff` | `boolean` | Whether notification is for staff/owner |
| `read` | `boolean` | Whether marked as read |
| `link` | `string` \| `null` | Deep link to relevant view (e.g. `session:abc123`) |
| `createdAt` | `ISO timestamp` | When created |
| `expiresAt` | `ISO timestamp` \| `null` | Auto-expire time |

#### Notification Types Reference

| Event | `type` value | Recipient | Example Message |
|-------|-------------|-----------|-----------------|
| Waitlist joined | `waitlist_joined` | Player | "You're #3 in queue for Court 1, 3PM" |
| Slot opened | `slot_opened` | All waitlisted | "🎾 A slot just opened for Court 1 at 3PM! Claim now (15 min)" |
| Claim expired | `claim_expired` | Claimer | "Your claim for 3PM expired — still on waitlist" |
| Booking confirmed | `booking_confirmed` | Player | "Booking confirmed for Court 1, 3PM" |
| Reminder | `reminder` | Player | "Reminder: You have Court 1 at 3PM today" |
| No-show recorded | `no_show_recorded` | Staff/Owner | "No-show recorded for [player] at [session]" |
| Strike added | `strike_added` | Staff/Owner | "⚠️ [Player] has 2/3 strikes" |
| Package low | `package_low` | Staff/Owner | "Only 2 credits left on [player]'s 10-pack" |
| Package expired | `package_expired` | Staff/Owner | "[Player]'s 10-pack expired" |

---

### Staff

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (UUID) | Primary key |
| `name` | `string` | Full name |
| `role` | `staff_role` | Permission level |
| `pin` | `string` | Login PIN (4–6 digits) |
| `active` | `boolean` | Whether account is active |
| `created_at` | `ISO timestamp` | Record creation |
| `updated_at` | `ISO timestamp` | Last update |

#### Staff Roles & Permissions

| Role | Permissions |
|------|-------------|
| `owner` | Full access: settings, pricing, staff mgmt, all reports, all actions |
| `manager` | Full access except delete owner, edit staff roles |
| `front_desk` | Check-in, walk-in, basic bookings, view schedule, view players |
| `coach` | View own sessions, manage clinic/lesson rosters, view player stats |

---

### Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `clubName` | `string` | `'Ace Pickleball Club'` | Venue display name |
| `lateCancelWindow` | `number` (minutes) | `120` | Hours before session to avoid late cancel |
| `noShowFee` | `number` (PHP) | `50` | Fee for no-show |
| `lateCancelFee` | `number` (PHP) | `30` | Fee for late cancellation |
| `strikeResetDays` | `number` | `30` | Days until strikes reset |
| `maxWaitlist` | `number` | `4` | Max entries per session waitlist |
| `claimWindowMinutes` | `number` | `15` | Minutes to claim an open slot |
| `cascadeRounds` | `number` | `2` | Notification rounds before opening to public |
| `requireWaiver` | `boolean` | `true` | Require signed waiver to book |
| `sessionDuration` | `number` (minutes) | `60` | Default session length |
| `checkInWindow` | `number` (minutes) | `30` | Minutes before session check-in opens |
| `peakHours` | `object[]` | `[]` | `[{ start: '17:00', end: '21:00', multiplier: 1.25 }]` |
| `memberPricing` | `object` | see below | Member vs guest pricing overrides |

#### `memberPricing` Structure
```js
{
  open_play: { guest: 150, member: 120 },
  league: { guest: 200, member: 175 },
  clinic: { guest: 350, member: 300 },
  // ...
}
```

---

## Top-Level State Shape

```js
state = {
  // Core entities
  courts: Court[],
  sessions: Session[],
  bookings: Booking[],
  queue: QueueEntry[],

  // Phase 2 entities
  players: Player[],
  packages: Package[],
  strikes: Strike[],
  notifications: Notification[],
  transactions: Transaction[],
  staff: Staff[],

  // Singleton
  settings: Settings,

  // Venue config
  venue: { name, address, phone, courts },

  // UI state
  selectedSessionId: string | null,
  currentDate: string (YYYY-MM-DD),
  bookingDate: string (YYYY-MM-DD),
  bookingFilter: string,
  recurringRules: object[],
}
```

---

## Relationships

```
Court 1──N Session
Session 1──N Booking
Session 1──N QueueEntry
Session 1──N Transaction (via Booking)
Player 1──N Booking
Player 1──N QueueEntry
Player 1──N Package
Player 1──N Strike
Player 1──N Transaction
Booking 1──1 Transaction
Package 1──N Booking (credits deducted)
Notification N──1 Player (optional)
```

---

## Business Rules Summary

1. **Court capacity**: Max 4 players per session (doubles)
2. **Queue FIFO**: Entries served in position order
3. **Session duration**: 1 hour default (configurable)
4. **Check-in window**: Opens 30 min before session start
5. **Late cancel threshold**: < 2 hrs before session → 1 strike + fee
6. **No-show penalty**: 2 strikes + fee
7. **Strike block**: 3 strikes → online booking disabled, front desk only
8. **Strike reset**: Auto-reset after configurable period (default 30 days)
9. **Package credits**: Deducted on booking; 1 credit for open play, 2 for clinic
10. **Package expiry**: Unused credits carry over within expiry window
11. **Waitlist cascade**: Slot open → notify all waitlisted → 15 min claim window → cascade → public
12. **Auto-pairing**: Balance teams by DUPR rating for competitive open play
13. **Winner rotation**: Winners stay on court, losers rotate out
14. **Payment methods**: Cash, GCash, Maya (manual reference), Card (future)
15. **Waiver required**: Players must sign waiver before booking (configurable)
