# Glory Jams — Event Calendar Integration Technical Spec

**Date:** 2026-03-12
**Proposal:** #131
**Status:** Research complete — ready for implementation decision

---

## Current Stack Summary

- **Hosting:** Firebase Hosting (static HTML, no build step)
- **Database:** Firebase Realtime Database (RTDB)
- **Auth:** Firebase Auth (anonymous + email)
- **Frontend:** Vanilla JS, no framework — DM Sans / Rock Salt / Asset fonts
- **APIs:** Google Places API (already integrated for venue search)
- **Data nodes:** `jams`, `checkins`, `profiles`, `hosts`, `submissions`, `subscribers`, `sets`
- **Style:** Dark theme (#0a0a0a bg), orange/gold accents (#ff6b35, #ffd700)

---

## 1. Calendar Widget Options

### Option A: FullCalendar.js (v6) — RECOMMENDED

| Aspect | Detail |
|---|---|
| Size | ~45KB gzipped (core + daygrid) |
| License | MIT |
| CDN | Yes (`cdn.jsdelivr.net/npm/fullcalendar@6`) |
| Mobile | Responsive, touch events, swipe nav |
| Views | Month, week, day, list — list view ideal for mobile |
| Theming | CSS custom properties — easy to match Glory Jams dark theme |
| Data | Accepts JSON event array, supports dynamic fetching |
| No build step | Works via CDN `<script>` tags — fits current architecture |

**Why recommended:** Most mature, best mobile support, list view is perfect for "what's happening this week" — the primary musician use case. CSS variables make theming trivial. Large community, good docs.

**Integration sketch:**
```html
<link href="https://cdn.jsdelivr.net/npm/fullcalendar@6/index.global.min.css" rel="stylesheet">
<script src="https://cdn.jsdelivr.net/npm/fullcalendar@6/index.global.min.js"></script>
```
```js
const cal = new FullCalendar.Calendar(document.getElementById('calendar'), {
  initialView: window.innerWidth < 768 ? 'listWeek' : 'dayGridMonth',
  events: eventsArray, // from Firebase
  headerToolbar: { left: 'prev,next today', center: 'title', right: 'listWeek,dayGridMonth' },
  eventClick: info => showSessionDetail(info.event)
});
cal.render();
```

### Option B: TOAST UI Calendar

| Aspect | Detail |
|---|---|
| Size | ~80KB gzipped |
| License | MIT |
| Mobile | Decent but heavier than FullCalendar |
| Views | Month, week, day |
| Theming | Theme object — more work to match dark theme |

**Verdict:** Heavier, fewer view options (no list view), less mobile-friendly. Not recommended.

### Option C: Custom HTML/CSS Calendar

| Aspect | Detail |
|---|---|
| Size | 0 deps |
| Effort | High — must build month grid, navigation, responsive layout, touch handling |
| Mobile | Full control but lots of work |

**Verdict:** For a simple "upcoming sessions" list, a custom card-based layout (no calendar grid) is actually lighter and better UX than a traditional calendar. Consider a **hybrid approach**: custom upcoming-sessions list for default view + FullCalendar for "full calendar" view.

### Recommendation: Hybrid Approach

1. **Default view (mobile-first):** Custom "Upcoming Sessions" card list — styled to match existing Glory Jams cards. Shows next 7-14 days. Zero dependencies.
2. **Expanded view:** FullCalendar list/month toggle for users who want to browse further out.
3. This avoids loading FullCalendar on first paint (performance win for mobile) while still offering full calendar functionality.

---

## 2. Data Storage

### Option A: Firebase RTDB `events` collection — RECOMMENDED

**Schema:**
```json
{
  "events": {
    "<eventId>": {
      "jamId": "ref-to-jams-node",
      "title": "Sunday Funk Jam",
      "venue": "Eli's Mile High Club",
      "address": "3629 Martin Luther King Jr Way, Oakland",
      "placeId": "ChIJ...",
      "date": "2026-03-15",
      "startTime": "19:00",
      "endTime": "22:00",
      "recurrence": "weekly",
      "genres": ["funk", "soul", "jazz"],
      "hostId": "host-uid",
      "hostName": "Simonoto",
      "description": "Bring your instrument. Backline provided.",
      "capacity": null,
      "rsvpCount": 0,
      "status": "active",
      "created": 1710000000000,
      "updated": 1710000000000
    }
  }
}
```

**Why RTDB over alternatives:**
- Already in the stack — no new services
- Real-time listeners mean RSVP counts update live
- `jams` node already has venue/host data; `events` extends it with specific dates/times
- Security rules already follow the auth pattern — just add an `events` node
- Admin can manage via existing admin.html patterns

**Recurrence handling:** Store a `recurrence` field (`weekly`, `biweekly`, `monthly`, `none`). Generate future occurrences client-side for the next 30 days from template data. No need for Cloud Functions.

**Proposed security rules:**
```json
"events": {
  ".read": true,
  ".write": "auth != null && auth.token.email != null",
  ".indexOn": ["date", "jamId", "status"]
}
```
Note: `.read: true` (public) so musicians can browse without auth — matches the "no account needed" philosophy. Write restricted to authenticated hosts.

### Option B: Google Calendar Embed

| Pro | Con |
|---|---|
| Zero backend work | No custom styling (looks like Google, not Glory Jams) |
| iCal export free | No RSVP integration with Firebase |
| | iframe breaks mobile UX |
| | Can't query events programmatically without Calendar API |

**Verdict:** Poor UX fit. The iframe embed looks foreign in the dark theme and breaks the seamless feel. Not recommended as primary, but could offer "Add to Google Calendar" links per-event.

### Option C: Static JSON file

| Pro | Con |
|---|---|
| Simplest possible | Must redeploy to update |
| No auth needed | No real-time RSVP counts |
| | No admin UI for hosts |

**Verdict:** Too manual for a community site. Events change frequently.

### Recommendation: Firebase RTDB `events` node

Fits the existing architecture perfectly. Real-time updates, existing auth patterns, admin management via admin.html.

---

## 3. RSVP / Attendance Tracking

### Feasibility: HIGH — builds on existing patterns

The codebase already has `checkins` and `subscribers` nodes. RSVP follows the same pattern.

**Proposed schema:**
```json
{
  "rsvps": {
    "<eventId>": {
      "<odientifier>": {
        "name": "Marcus",
        "instrument": "bass",
        "timestamp": 1710000000000,
        "status": "going"
      }
    }
  }
}
```

**identifier** can be:
- Firebase Auth UID (if logged in)
- Anonymous auth UID (for no-account RSVP)
- Phone hash (ties into existing subscriber system)

**Features:**
- **"I'm pulling up" button** — one-tap RSVP, no account needed (anonymous auth)
- **Live count** — "12 musicians going" updates in real-time via RTDB listener
- **Instrument breakdown** — "3 guitarists, 2 drummers, 1 keys" helps musicians decide
- **Optional name claim** — ties into existing `claimed_names` node
- **RSVP → check-in flow** — on event day, RSVP converts to check-in at venue

**Security rules:**
```json
"rsvps": {
  "$eventId": {
    ".read": true,
    ".write": "auth != null",
    "$uid": {
      ".write": "auth.uid === $uid"
    }
  }
}
```

### What NOT to build (scope control):
- No waitlists or capacity limits (adds complexity, jams are open)
- No notifications/reminders (save for later — would need Cloud Functions)
- No social features beyond count (no comments, no follows)

---

## 4. Mobile-First Design Requirements

### Key Constraints
- Musicians check on their phones, often while commuting or between sets
- Most usage: "What's happening tonight/this week?"
- Thumb-zone friendly — primary actions at bottom of viewport
- Dark theme already optimized for low-light venues

### Layout Spec

**Mobile (< 540px) — Primary:**
```
┌─────────────────────────┐
│ GLORY JAMS    [Filter ▾]│  ← sticky header
├─────────────────────────┤
│ ▸ Tonight (2)           │  ← collapsible date groups
│ ┌─────────────────────┐ │
│ │ 🎵 Sunday Funk Jam  │ │
│ │ Eli's Mile High     │ │
│ │ 7pm – 10pm          │ │
│ │ 🎸×3 🥁×1 🎹×2     │ │
│ │ [I'm Pulling Up →]  │ │  ← primary CTA
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │ 🎵 Jazz Open Mic    │ │
│ │ ...                  │ │
│ └─────────────────────┘ │
│                         │
│ ▸ Tomorrow (1)          │
│ ▸ This Week (4)         │
├─────────────────────────┤
│ [📅 Full Calendar]      │  ← secondary: loads FullCalendar
└─────────────────────────┘
```

**Desktop (≥ 768px):**
- FullCalendar month view with event dots
- Click event → slide-out detail panel (reuses mobile card design)
- Sidebar with upcoming list

### Performance Budget
- First paint: < 1.5s on 3G (no FullCalendar on initial load)
- FullCalendar loaded on demand when user taps "Full Calendar"
- Event data: single RTDB query, cached client-side
- Images: none needed (text + icons only)

### Accessibility
- Semantic HTML (`<time>`, `<article>` for events)
- ARIA labels for interactive elements
- Keyboard navigable
- Color contrast meets WCAG AA on dark background

---

## 5. Implementation Plan

### Phase 1: Core Calendar (Est. effort: ~6-8 hours)
1. Add `events` node to RTDB + security rules
2. Build upcoming sessions list view (mobile-first cards)
3. Add date filtering (tonight / this week / this month)
4. Add city/genre filter (reuse existing filter patterns from index.html)
5. Link from landing.html and index.html navigation

### Phase 2: RSVP (Est. effort: ~4-6 hours)
1. Add `rsvps` node to RTDB + security rules
2. "I'm pulling up" button with anonymous auth
3. Live RSVP count on event cards
4. Instrument breakdown display
5. Tie into existing check-in flow

### Phase 3: Admin Management (Est. effort: ~3-4 hours)
1. Add event CRUD to admin.html
2. Recurrence template support (weekly/biweekly/monthly)
3. Auto-generate next 30 days of occurrences from templates
4. Bulk event management (cancel, reschedule)

### Phase 4: Enhanced Calendar (Est. effort: ~2-3 hours)
1. Lazy-load FullCalendar for "full calendar" view
2. "Add to Google Calendar" / "Add to Apple Calendar" (.ics) links
3. Desktop month-view layout

### Phase 5: Polish (Est. effort: ~2 hours)
1. Empty states ("No sessions this week — host one!")
2. Deep links (gloryjams.com/events#2026-03-15)
3. SEO structured data (Event schema.org)
4. Open Graph tags for shared event links

---

## 6. Technical Decisions Summary

| Decision | Choice | Rationale |
|---|---|---|
| Calendar widget | FullCalendar.js (lazy-loaded) | Best mobile support, list view, MIT, CDN-ready |
| Default view | Custom card list | Lighter, faster, matches existing UX |
| Data storage | Firebase RTDB `events` node | Already in stack, real-time, existing auth |
| RSVP | Firebase RTDB `rsvps` node | Anonymous auth, real-time counts, existing patterns |
| Recurrence | Client-side generation | No Cloud Functions needed, simple templates |
| Public read | Yes (events + rsvps) | Matches "no account needed" philosophy |
| Build step | None | Keep current architecture (CDN + vanilla JS) |

---

## 7. Open Questions for Simon

1. **Should events be a separate page (`events.html`) or integrated into index.html?** Recommendation: separate page, linked from nav.
2. **How far out to show events?** Recommendation: 30 days, with "full calendar" for further.
3. **Should hosts be able to create events directly, or admin-only?** Current setup is admin-managed — could add host self-service later.
4. **"Add to Google Calendar" links — worth the effort in Phase 1?** Recommendation: defer to Phase 4.
