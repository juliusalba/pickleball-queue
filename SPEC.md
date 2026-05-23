# Ace Pickleball Club — Phase 2 Feature Specification

## Business Context
- Location: Philippines (prices in PHP)
- Multi-court (2-4 courts, configurable)
- Owner + Staff + Players as user types
- No subscription — pay-per-play + packages + memberships
- Currently: single-file vanilla JS app (localStorage persistence)

---

## 1. Waitlist & Cancellation Cascade

### Waitlist Rules
- Players join waitlist from the booking screen or session detail
- Max 4 players per waitlist entry (group size)
- Position visible ("#2 in queue")
- When a slot opens: **all waitlisted players get a claim notification**
- Claim window: **15 minutes** — first to claim wins
- If no claims in 15 min → cascade to next waitlisted OR open for public booking
- Claim via: in-app button (if player has account) OR front desk call

### Cancellation Cascade Flow
```
Player A cancels booking
→ System checks waitlist for that session
→ If waitlist not empty:
    → Notify ALL waitlisted players simultaneously
    → 15-minute countdown starts
    → First to claim gets the slot
    → If no claims in 15 min → cascade to next round
    → After 2 rounds (30 min) → slot opens for public booking
→ If waitlist empty:
    → Slot immediately opens for public booking
```

### Waitlist Entry States
- `waiting` — in queue, hasn't been called
- `notified` — slot opened, 15-min countdown active
- `claimed` — player claimed the opening
- `expired` — claim window passed
- `cancelled` — player manually left waitlist
- `served` — player got on court

### No-Show / Late Cancel Policy (Configurable)
- **Late cancel** (< 2 hrs before): 1 strike, small fee
- **No-show**: 2 strikes, fee charged
- **3 strikes total** → blocked from online booking (must go to front desk)
- Strikes reset monthly (configurable)
- Fees auto-deducted or added to next booking

---

## 2. Player Directory & Profiles

### Player Profile Fields
```
- id, name, phone, email
- dupr (skill rating, 0.5 increments 2.0–6.5)
- memberType: 'guest' | 'day_pass' | 'monthly' | 'annual' | 'package'
- packageCredits: number (for 10-pack holders)
- packageExpiry: date
- strikes: number (0-3)
- strikeHistory: [{ date, reason, type }]
- emergencyContact: { name, phone }
- waiverSigned: boolean
- notes: string (staff notes)
- sessionsPlayed, wins, losses (stats)
- lastPlayed, memberSince (dates)
```

### Player Directory View
- Searchable list: name, phone, DUPR, member type
- Quick filters: All / Members / Guests / On Strike
- Click → full player profile modal
- Add/edit player (front desk or owner)

### Member Types & Pricing (PHP)

| Type | Price | Details |
|------|-------|---------|
| Guest / Drop-in | ₱150/session | Pay as you go |
| Day Pass | ₱300/day | Unlimited open play for 1 day |
| 10-Pack | ₱2,500 | 10 sessions, 90-day expiry |
| Monthly | ₱1,500/mo | Unlimited open play |
| Annual | ₱12,000/yr | Priority booking, discounts |

### Session Types & Pricing (PHP)

| Type | Per Player | Per Court (1hr) |
|------|-----------|----------------|
| Open Play | ₱150 | — |
| Reserved (Private) | — | ₱500 |
| League | ₱200 | — |
| Tournament | ₱250 | — |
| Clinic (Group) | ₱350 | — |
| Private Lesson | — | ₱800 |

---

## 3. Package / Pass System

### Package Purchase Flow
1. Player buys package (front desk or owner does this)
2. System creates package record linked to player
3. When player books → credits deducted automatically
4. Warning at 2 credits remaining
5. Package expires → must renew

### Package Types (Owner Configurable)
- **10-Pack** — 10 credits, ₱2,500, 90-day expiry
- **5-Pack** — 5 credits, ₱1,400, 60-day expiry
- **Day Pass** — unlimited same-day open play, ₱300
- **Monthly** — unlimited open play, auto-renews, ₱1,500

### Credits Behavior
- Open Play: 1 credit/session
- Clinic: 2 credits
- League: 1 credit
- Unused credits carry over within expiry window

---

## 4. Auto-Pairing Algorithm (Open Play)

### Goal
Balance teams by DUPR skill rating to keep games competitive.

### Algorithm — Balance Teams
```
Input: N players checked in for open play session

1. FILTER by minimum players needed (4 for doubles)
2. SORT players by DUPR rating descending
3. GROUP into buckets:
   - Top: DUPR 4.0+
   - Mid: DUPR 3.0-3.9
   - Low: DUPR below 3.0
4. BUILD 4-player teams:
   a. Best + worst as partners (partner delta ≤ 0.5)
   b. Two best pairings that minimize total team DUPR delta
   c. Team A avg DUPR ≈ Team B avg DUPR ± 0.3
5. TRACK recent matchups — avoid pairing same opponents within 2 rounds
6. OUTPUT: Team A vs Team B with court assignment
```

### Auto-Rotation Rules
- Winner stays, loser rotates out to waiting room
- If waiting room has players → next team promoted
- If no waiters → winners stay on, new opponents from checked-in pool
- Track games played: avoid repeating same 4-player combo for 3 rounds minimum

### Pairing UI
- Open Play session shows: "Auto-pair teams" button
- Generates teams → shows on matchup board
- Staff can manually override any pairing

---

## 5. Payments (Philippines-Ready)

### Payment Methods
- **Cash** — logged at front desk
- **GCash** — reference number entered manually
- **Maya (PayMaya)** — reference number entered manually
- **Card** — (future: Stripe/Square integration)

### Payment Flow
1. Player books → sees price breakdown (session × players + fees)
2. Selects payment method
3. If GCash/Maya: enters reference number
4. Staff/owner approves → booking confirmed
5. Receipt generated (in-app + SMS/email)

### Receipt / Transaction Record
```
{
  id, bookingId, playerName, sessionId,
  sessionName, courtName, date, time,
  playerCount,
  ratePerPerson, subtotal, processingFee, total,
  paymentMethod, referenceNumber,
  status: 'pending' | 'paid' | 'refunded',
  staffProcessedBy,
  createdAt, paidAt
}
```

---

## 6. Staff Roles & Permissions

### Roles
| Role | Permissions |
|------|------------|
| **Owner** | Full access: settings, pricing, staff mgmt, all reports, all actions |
| **Manager** | Full access except delete owner, edit staff roles |
| **Front Desk** | Check-in, walk-in, basic bookings, view schedule, view players |
| **Coach** | View own sessions, manage clinic/lesson rosters, view player stats |

### Staff Dashboard
- Owner sees all staff + permissions
- Add/edit/remove staff
- Log staff activity (who processed what booking)

---

## 7. Notifications (Client-Side Toast + In-App)

### Notification Types
| Event | Message | Who |
|-------|---------|-----|
| Waitlist joined | "You're #3 in queue for Court 1, 3PM" | Player |
| Slot opened | "🎾 A slot just opened for Court 1 at 3PM! Claim now (15 min)" | All waitlisted |
| Claim expired | "Your claim for 3PM expired — still on waitlist" | Claimer |
| Booking confirmed | "Booking confirmed for Court 1, 3PM" | Player |
| Reminder (2hr) | "Reminder: You have Court 1 at 3PM today" | Player |
| No-show recorded | "No-show recorded for [player] at [session]" | Staff/Owner |
| Strike added | "⚠️ [Player] has 2/3 strikes" | Staff/Owner |
| Package low | "Only 2 credits left on [player]'s 10-pack" | Staff/Owner |
| Package expired | "[Player]'s 10-pack expired" | Staff/Owner |

### Implementation
- In-app toast notifications (already exists: `showToast()`)
- Notification center: bell icon in header → dropdown list
- Notifications stored in `state.notifications[]`
- Unread badge count on bell icon
- "Mark all read" button

---

## 8. Smart Session Features

### Session Statuses
- `scheduled` — future session, no players yet
- `check_in_open` — 30 min before start, players can check in
- `in_progress` — session is live
- `completed` — past session
- `cancelled` — cancelled by staff

### Check-In Rules
- Check-in opens 30 minutes before session start
- Pre-booked players tap "Check In" (or staff does it)
- If player no-shows: staff marks as no-show → strike applied
- Session auto-transitions to `in_progress` at start time

### Session Recap (Post-Session)
- Auto-shows match results summary
- Prompts for: "Enter scores?" (if open play with match tracking)
- Updates player stats (wins/losses/plays)

---

## 9. Dashboard & Analytics

### Owner Dashboard Widgets
1. **Today's Overview** — courts booked, revenue, check-ins vs no-shows
2. **Waitlist Performance** — how many waitlist → conversions today
3. **Top Players** — most active this week
4. **Utilization Rate** — % of court hours used today
5. **Revenue Today / This Week / This Month**
6. **Strike Report** — players currently on strike
7. **Package Expiry Alert** — packages expiring in next 7 days

### Revenue Breakdown
- By session type (open, clinic, league, etc.)
- By court
- By payment method (cash vs GCash vs Maya)
- Peak hours heatmap

---

## 10. Settings Panel (Owner Only)

### Court Management
- Add/remove courts
- Court name, surface type (Indoor Hard, Outdoor Clay, etc.)
- Court amenities (lighting, fans, shade, scoreboard)
- Per-court enabled session types

### Pricing Control
- Edit all session prices per type
- Set peak hours (higher pricing)
- Set member vs guest pricing

### Cancellation Policy Config
- Late cancel window: 2 hrs (editable)
- No-show fee: ₱50 (editable)
- Late cancel fee: ₱30 (editable)
- Strike reset period: 30 days (editable)

### Waitlist Config
- Max waitlist per session: 4 (editable)
- Claim window: 15 min (editable)
- Cascade rounds: 2 (editable)

---

## File Structure

All in `index.html` (single-file app):
- CSS: design tokens, components, responsive (existing + new)
- HTML: layout, all views, modals (existing + new)
- JS: state management, all functions (existing + new)

### New JS Modules (conceptual sections)
1. **Waitlist Engine** — join, notify, claim, expire, cascade logic
2. **Pairing Algorithm** — `balanceTeams()`, `autoRotate()`, `findNextOpponents()`
3. **Package Manager** — purchase, deduct, expire, warn
4. **Payment Processor** — GCash/Maya/Cash flows
5. **Notification Center** — toast + bell icon + notification history
6. **Player Directory** — CRUD, search, filters, strike management
7. **Check-In System** — timed check-in, no-show detection
8. **Strike System** — apply, reset, block booking

---

## Data Model Changes

### New State Fields
```js
state = {
  // EXISTING
  courts, sessions, bookings, queue,
  transactions, staff, venue,
  playerDirectory, selectedSessionId,
  currentDate, bookingDate, bookingFilter,
  recurringRules,

  // NEW
  packages: [{
    id, playerName, type: '10_pack'|'5_pack'|'day_pass'|'monthly',
    credits, maxCredits, price, purchasedAt, expiresAt,
    status: 'active'|'expired'|'used_up'
  }],
  strikes: [{
    id, playerName, type: 'late_cancel'|'no_show',
    date, sessionId, fee, resolved
  }],
  notifications: [{
    id, type, message, forPlayer, read,
    createdAt, expiresAt
  }],
  settings: {
    lateCancelWindow: 120, // minutes
    noShowFee: 50, // PHP
    lateCancelFee: 30, // PHP
    strikeResetDays: 30,
    maxWaitlist: 4,
    claimWindowMinutes: 15,
    cascadeRounds: 2,
    requireWaiver: true,
    clubName: 'Ace Pickleball Club'
  },
  // expanded playerDirectory
  players: [{ /* full player profile */ }]
}
```

---

## Build Priority

```
Step 1:  Update schema.md + data model
Step 2:  Player directory view + profile modal + CRUD
Step 3:  Strike system + no-show/late-cancel logic
Step 4:  Package/pass system (purchase, deduct, expiry)
Step 5:  Waitlist engine (join, notify, claim, cascade)
Step 6:  Payment flow (GCash/Maya reference, receipts)
Step 7:  Notification center (bell icon, toast, history)
Step 8:  Auto-pairing algorithm for open play
Step 9:  Check-in system (30-min window, auto-transitions)
Step 10: Owner dashboard widgets + analytics tweaks
Step 11: Settings panel (owner-only: pricing, policies, courts)
```
