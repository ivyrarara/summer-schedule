# Decision Log

Why things were built the way they were — not just what changed. New entries
go at the top. Each commit that makes a non-obvious call should get an entry
here, in addition to explaining itself in the commit message.

Context: `index.html` is a single-file bundle. The actual app source (a
custom `x-dc`/React template) lives inside a `<script type="__bundler/template">`
tag as a JSON-encoded string — it's not directly readable/editable as plain
HTML/JS. Edits go through a decode → string-replace → re-encode round trip.

---

## YRDSB school-board holidays added, orange, same mechanism as PA Day

**Decision**: generalized `PA_DAY_DATES` (a list) into `ORANGE_LABEL_DATES`
(a date → label map) and added the York Region District School Board's
2026-2027 holidays: Labour Day (2026-09-07), Thanksgiving Day
(2026-10-12), Winter Break (2026-12-21 through 2027-01-01, weekdays only),
Family Day (2027-02-15), and Mid-Winter Break (2027-03-15 through
2027-03-19). Same override, same position, same orange as PA Day — the
label text just comes from the map instead of being hardcoded to "PA Day".

**Sourcing**: Labour Day, Thanksgiving, Winter Break, and Family Day dates
came from a screenshot of yrdsb.ca the user shared, but that screenshot was
cropped before showing Mid-Winter Break's dates. Rather than guess, looked
it up (web search, corroborated by two independent results) — Monday
2027-03-15 through Friday 2027-03-19. Flagged to the user that this one
date range is sourced differently (search, not their screenshot) so they
can double-check it against their own reference.

**Mid-Winter Break currently has no visible effect**: the season only runs
through 2027-02-25 right now, before March. The dates are in the map
regardless — if the season is extended into March later, this holiday
shows up with no further changes needed, same as how `PA_DAY_DATES`/
`ORANGE_LABEL_DATES` needs no `patchMissingDefaults()`-style migration
(it's a code constant read at render time, not saved state).

## PA데이 reverted to a hardcoded date list, not a toggle

**Decision**: undid the previous entry's PA데이 status-toggle/badge feature
entirely (removed it from `weekdayStatusLabels`, reverted the per-badge
`pillColor`/`pillBg` mechanism back to the plain hardcoded accent color).
Replaced it with a fixed `PA_DAY_DATES` list of five specific dates; on
those dates, the weekday's camp-name-field display (the bold title at the
top of the day card / day detail card — the same spot a real camp's name
would show) is overridden to read "PA Day" in orange (`#F2600A`), rather
than "캠프 이름" or whatever placeholder would otherwise show there.

**Why override the label field instead of adding a new element**: these
are individual specific dates, not a whole week, so the change can't go
through `campFor()`/`WEEKS_DATA_DEFAULT` (that's a per-week assignment,
shared by every weekday in the week). Building `labelField`/
`detailCampNameField` already happens per individual day, so overriding
`.value` (and `.textColor` / `.textStyle.color` for the weekly vs. detail
card, respectively) right after they're built is the smallest change that
reaches exactly the five named dates and no others. Left `locField`/
`detailCampLocField` untouched — the user only asked about the "활동명"
(activity name) position, and 5 of the dates fall inside what's still
technically camp season, so leaving location alone is arguably correct too
(their real camp still has a real location that week).

**Why this needed no patch-forwarding**: unlike the season/`weeksData`
changes earlier in this log, `PA_DAY_DATES` isn't part of saved state at
all — it's a plain code constant consulted at render time — so it applies
immediately to a live app with existing saved data, no
`patchMissingDefaults()`-style migration needed.

## Week-of-month labels recomputed sequentially; PA데이 badge colored orange

**Decision one — week labels**: replaced the per-week, independent
"week-of-month" calculation (`ceil((date + firstWeekday)/7)`, applied only
to a week's *start* date) with a sequential `computeWeekLabels()` that
tracks a running (month, week number) across the whole season. A week that
spans two months now goes to whichever month it has more than 4 days in —
the later month if so, otherwise it continues the earlier month's count.
Critically, this is computed as a running sequence, not a one-off override
for a single flagged week: reclassifying one transition week into the next
month correctly shifts every later week's number within that month too,
instead of leaving a gap or a duplicate.

**Why this surfaced now**: the user's actual live `seasonStart` (visible in
a screenshot: "2026.06.28 – ...") is one day earlier than this repo's
`SEASON_START_DEFAULT` ("2026-06-29") — their saved snapshot has carried
that value since before this repo existed, and (per the "Default view"
entry) a saved `seasonStart` always wins over the code default, so this was
never going to show up by checking the code's own dates. With their real
2026-06-28 start, the week spanning 2026-08-30–09-05 has 5 days in
September and only 2 in August — under the old algorithm it was labeled by
its start date ("8월 5주"); under the new rule, correctly "9월 1주". Verified
by reproducing their exact `seasonStart` value and confirming that specific
relabel, not just eyeballing the general rule.

**Decision two — PA데이 badge color**: when a day's "PA데이" status badge is
active, its pill uses orange (`#F2600A`, the same orange the "+ 활동"/"+ 메모"
buttons use) instead of the app's theme-accent color every other status
badge uses. Added `pillColor`/`pillBg` per badge in `computeDayBadges()`
(defaulting to the existing accent color for every other label) rather than
hardcoding the color in the markup, so only PA데이 changed and nothing else
did.

## Weekday status options swap for the school year (워터플레이/수영/필드트립 → PA데이)

**Decision**: from 2026-09-07 onward (the same date camp season already
ends — "9월 2주," see the season-extension entry below), a weekday's status
options change from `["워터플레이", "수영", "피자데이", "필드트립"]` to
`["피자데이", "PA데이"]`. 피자데이 stays either way since it still applies
during the school year; the three summer-camp-specific labels are replaced
by PA데이 (a school PA/professional-activity day). Implemented as one
`weekdayStatusLabels(dateKey)` function shared by both the weekly view and
the day detail card, keyed off the date string, rather than duplicating the
cutoff logic — same reasoning as reusing `FALL_STATUS_CUTOFF` matching the
existing camp-end date rather than inventing a new one.

## Season extended again, to 2027-02-25

**Decision**: same mechanism as the 2026-12-31 extension — grew
`WEEKS_DATA_DEFAULT` from 27 to 35 weeks, all new weeks `camp: null`. The previous last entry (2026-12-28, 4 days) became a full
7-day week again, for the same reason as before: the real week-chunking is
a pure function of total days from the season start, oblivious to this
array's own `days` values, so a longer season always re-expands whatever
was previously the trailing partial week. `patchMissingDefaults()` already
handles delivering this to users with an already-saved (shorter) snapshot
— verified again here by simulating a saved 27-week snapshot with
`seasonTotalDays` already pointing at the new end date, and confirming
reload grows `weeksData` to 35 with no camp on the new weeks.

## GROCERIES tab replaced with MEAL PLAN only (store lists removed)

**Decision**: renamed the "GROCERIES" tab button to "MEAL PLAN," removed the
한인식품점/로컬마트/확정 식단 sub-nav entirely, and made the confirmed-menu
content (아침/점심/도시락/저녁 categories) render unconditionally as the only
thing in that tab — no more sub-tab to click through to reach it.

**What was left alone on purpose**: the grocery-list state and methods
(`groceryData`, `groceryChecked`, `GROCERY_DEFAULT`, `toggleGroceryChecked`,
`addGroceryItemImpl`, `renameGroceryItem`) still exist but are now fully
unreferenced by anything rendered. Removed the render-time view-model glue
that only existed to wire up the deleted UI (`isKoreanStore`/`isLocalStore`/
`isMenuView`, `selectKorean`/`selectLocal`/`selectMenu`, the three
`storeXStyle` values, the grocery add-item input handlers, `clearChecked`,
and the `groceryItems` list computation) since that was directly and
unambiguously dead once the markup no longer used it. Didn't go further and
strip the underlying state fields/methods/constant — that reaches deeper
into the state shape (what a fresh install initializes, what a saved
snapshot carries) for no behavioral difference, since inert unused state is
harmless. If the store-list feature is confirmed gone for good, that's a
reasonable follow-up cleanup, not a must-do-now one.

## Season extended to 2026-12-31, camp ends, fall recurring activities added

**Decision**: extended `WEEKS_DATA_DEFAULT` from 10 weeks (ending
2026-09-04) to 27 weeks (ending 2026-12-31) — the last real camp
("서머 스포츠 올림픽," week index 9) is unchanged, and every week from
index 10 onward gets `camp: null` (school has started, no more camps).
Added four new recurring activities for 2026-09-21 through 2026-12-20
(inclusive of both boundary weeks): Skate Sun 11:10–12:10 and Wed
4:00–5:00 at Maple, Swimmer 7 Sat 10:00–11:00 and Tue 4:30–5:30 at
Carreville Community Center — seeded via the same
weekday-template + explicit-date-list pattern the existing summer
Sat/Sun/Mon activities already used.

**Why there's no week literally labeled "9월 1주"**: the app labels each
week by its *start date's* week-of-month, and week boundaries are fixed
7-day blocks counted from the season start — they don't reset at month
boundaries. The last camp week (starts 2026-08-31) already runs past
September 1st before hitting the next boundary (2026-09-07, "9월 2주").
So "camp ends starting 9월 1주" became "camp ends starting 9월 2주," which
is the closest the underlying date math allows; flagged this back to the
user rather than silently picking one interpretation.

**The harder problem: this data lives in two places.** The code's
`WEEKS_DATA_DEFAULT`/seed arrays are only the *fresh-install* default —
the user's actual live app already has months of hand-entered data sitting
in their browser's `localStorage`, and `restoreLocalSnapshot` completely
replaces `weeksData`/`dayActivities` with whatever's saved there (a plain
object spread, not a per-key merge — see the "Default view" entry below,
which already established this same fact for the view-state fields).
Just editing the code would have had **zero effect** on their real app:
their shorter, older arrays would keep winning every load, forever.

Added `patchMissingDefaults()`, called after both the local-restore and
the remote-sync-restore paths settle. It's deliberately narrow: append
weeks to `weeksData` only past the user's *current* array length (never
touches an index they already have — including their own edits to the
last camp), and add a `dayActivities` entry for a date only if that date
has *no* entry at all yet. A deleted activity is `[]` in this app (see
`deleteDayActivity`), which is truthy — so patching only fires for dates
that were never seeded, never for ones the user intentionally emptied.
Verified by simulating an old, pre-extension saved snapshot (10-week
array, a hand-edited camp name, a deleted activity, a camp review) and
confirming a reload adds the new weeks/activities while leaving every one
of those four things untouched, and that repeated reloads don't
double-append.

## Camp review box: no resize handle (PR #9)

**Decision**: turned off `resize: vertical` (and the corner grip icon that
comes with it) once the box was already auto-growing to fit its content.

**Why**: manual resize and auto-grow both changing the same `height` fight
each other — the grip invites a manual drag that the next keystroke would
immediately overwrite anyway. With auto-grow in place, the grip was a
leftover affordance with no real use, so it's gone rather than reconciled.

## Camp review box: auto-grow to fit content (PR #8)

**Decision**: box height now recalculates on every keystroke and on every
`onStateSettled()` (which already fires after each state update, e.g.
switching weeks), floor at the ~84px minimum, no cap.

**Why / how**: this needed `element.style.height` to be set imperatively
from JS after measuring `scrollHeight`. That only works if the *stylesheet*
never declares `height` with `!important` for the element — a plain inline
style (however it's set, including via JS) always loses to an `!important`
stylesheet rule, no exceptions. The app's global `.input` rule forces
`height: 1lh !important`. Once the review textarea's own class stopped
declaring a fixed height (to let it grow), that global rule became the only
declared `height` and silently overrode every JS resize. Fix: dropped the
`input` class from the two review textareas entirely, so nothing else in the
stylesheet has a say over `height`. The dedicated `.camp-review-input` class
already re-declares every other visual property it needs, so nothing was
lost by decoupling it from `.input`.

## Camp review box: no border / no corner radius (PR #7), textarea instead of input (PR #6)

**Decision**: switched the review field from `<input>` to `<textarea>`
(so long reviews wrap instead of scrolling off-screen), then on follow-up
feedback stripped the border and `border-radius` so it reads as part of the
card rather than a separate boxed element.

**Why this took two PRs to land at all**: `style="height: 84px !important"`
written inline (via the template's `style="..."` string) does **not** work
in this app. The React runtime sets inline styles through the DOM `style`
*property* API (`el.style.height = value`), and that API silently drops any
value containing the literal text `!important` — it's only meaningful when
parsed from real CSS text (a stylesheet rule, or the raw HTML `style`
attribute parsed by the browser's HTML parser, not JS). Since the app's
global `.input { height: 1lh !important }` rule outranks a plain (non
-important) inline override regardless of specificity, the only way to win
was a real stylesheet rule (`.camp-review-input { height: ... !important }`
in the `<style>` block) with higher specificity. This class of bug (inline
`!important` silently doing nothing) came up twice in this session — see the
entry below too.

## Camp review input: separate feature, gated on "week has ended" (PR #5)

**Decision**: a plain always-visible textarea at the bottom of the "THIS
WEEK'S CAMP" card (weekly banner *and* month-view camp card, reading/writing
the same `campReviews[weekIndex]` state so they stay in sync), shown only
once a real camp is assigned *and* that week's last day is before today.

**Why gated this way**: showing it unconditionally would surface an empty
prompt for camps that haven't happened yet, which reads as premature. The
condition intentionally mirrors "the week is actually over," not "today is
the last day" or similar — a week is either fully in the past or it isn't.

**Process note**: this one was mocked up locally (screenshots sent, three
open questions asked — placement, whether month view should share the data,
plain text vs. something richer) and held back from `main` until explicitly
approved, per the user's request to review the design before it deploys
(this repo is served live via GitHub Pages, so merging to `main` *is*
deploying). Everything after this entry in the log went through the normal
build → PR → merge flow without a pre-deploy mockup step, since it was
either small/mechanical or directly requested.

## Playdate: separate feature from Activity, own banner (no pill) (PR #3, follow-ups)

**Decision**: reverted an earlier "add 누구랑 (who) to every Activity"
change, and instead gave "플레이데이트" (playdate) its own 시간/장소/누구랑
fields in a dedicated card, shown only when that status is toggled on for a
day. Later removed the redundant pill badge that also appeared for that
status, so the fields card is the only visual indicator (with its own ×
to remove it).

**Why the reversal**: Activity covers scheduled, structured things (swim
class, basketball) — a "who" field doesn't make sense there. The original
implementation was chosen from a menu of options the user picked, but once
built and seen in context it was clearly wrong for the underlying concept.
Kept as a lesson: when a feature choice is between "reuse an existing
mechanism" vs. "model this as its own thing," prefer showing a concrete
mockup before committing to the reuse path, since the tradeoff is often
only obvious once rendered.

## "일정 없음" (no schedule) made to actually work (PR #2)

**Decision**: the "no schedule" placeholder text and its layout were driven
by a hardcoded `NO_CAMP_INFO_DAYS = {}` constant that is *always* empty —
so the flag was permanently `false` everywhere and the text could never
render, in the original code as received. Replaced it with a dynamic check
(no activities *and* no label/location text set) in both the weekly view
and the day detail card, and gave the weekly view the same
non-overlapping flex layout the detail card had already been fixed to use
(PR #1) — because once the text could actually appear, the weekly view's
old `position: absolute` button placement had the same overlap risk that
had already been fixed in the detail card.

**Why this was worth digging into rather than patching the symptom**: the
original bug report ("생일" → corrected to "생긴," i.e. newly-appeared dates
after extending the season range) only reproduces when a day has genuinely
no schedule — which never happened before this fix, since the flag was
dead. Fixing only the layout (PR #1) without this would have left the
underlying feature non-functional; the overlap symptom and the "text never
shows" bug were the same root cause wearing two hats.

## Default view = weekly tab, today's date, on every launch (PR #1)

**Decision**: `restoreLocalSnapshot()` and the remote-sync merge in
`initSync()` now explicitly strip `tab`, `viewMode`, `weekIdx`, `monthIdx`,
and `selectedKey` from whatever gets merged into state — everything else
(camp data, notes, activities, etc.) still restores normally.

**Why**: the state's own defaults already computed "today's week, schedule
tab" correctly — `getDefaultWeekIdx()` etc. were right. The bug was that
`restoreLocalSnapshot` unconditionally spread saved `localStorage` state
(including whatever tab/week/month the user had last been looking at) over
those defaults, silently overwriting them on every load. The fix targets
exactly the fields that represent "where you are," leaving "what's in the
schedule" alone.

---

## Working notes on this codebase's quirks (for future edits)

- **`</` inside the embedded JSON string must stay escaped as `</`.**
  The original bundle escapes every `</` this way so that literal
  `</script>` sequences inside the JSON payload never prematurely close the
  outer `<script type="__bundler/template">` tag. `JSON.stringify` doesn't
  escape `/` by default — any edit that re-serializes the template must
  re-apply `.replace('</', '<\\u002F')` afterward, or the page silently
  breaks (the browser's HTML parser closes the script tag early).
- **Inline `style="... !important"` does not work in this app.** See the
  camp-review-box entries above. If a style needs to beat a global
  `!important` rule (several exist, e.g. `.input`), it needs a real
  stylesheet class with sufficient specificity, not an inline override.
- **"Deploying" means merging to `main`.** This repo is served live via
  GitHub Pages from `index.html` at the root — there's no separate deploy
  step. Anything merged is immediately what the user's real, in-use app
  shows. Non-trivial or ambiguous UI changes should be mocked up and
  confirmed before merging, not just before "shipping" in some other sense.
