# Chore Chart Discord Bot: Project Plan

A Discord bot that runs chore charts for households. It starts with one household, then becomes a public bot that many households can use, including several private households on the same Discord server.

---

## 1. Tech stack

| Area | Choice | Why |
|---|---|---|
| Language | TypeScript (Node.js LTS) | Type safety; the best-supported Discord library |
| Discord library | discord.js v14 | Slash commands, buttons, modals, select menus |
| Database | PostgreSQL (managed: Neon or Supabase) | Cloud-hosted, with backups; data is relational |
| ORM | Drizzle ORM + drizzle-kit migrations | Type-safe queries; schema kept in code |
| Job scheduler | pg-boss (job queue stored in Postgres) | Reminders, summaries and overdue handling with no extra service |
| Dates and timezones | Luxon | Each household has its own timezone |
| Validation | Zod | Checks command input and environment settings |
| Email | Resend or Amazon SES | Reminder and summary emails |
| SMS | Twilio | Text reminders |
| Push | ntfy first, then web push (PWA) later | See section 7 |
| Logging | pino | Structured logs |
| Tests | Vitest | Mostly for scheduling and assignment logic |
| Hosting | Railway or Fly.io (always-on) | The bot must stay connected to send scheduled reminders |

---

## 2. Core concepts

- **Discord server (guild):** can hold **many households**.
- **Household:** the unit that owns chores. It has an owner, members and settings. All data is separated by `household_id`.
- **Member:** a Discord user in a household, with the role `owner`, `admin` or `member`. A user can belong to more than one household.
- **Chore:** a recurring (or one-time) task that belongs to a household.
- **Chore event:** a history row for each time a chore is done, skipped or reassigned. Points, the leaderboard and summaries are built from these rows.

### Privacy between households
- Every database query is filtered by `household_id`, and membership is checked **on the server**, not just in Discord.
- Personal views (`/mychores`, settings, stats) use **ephemeral** replies, which only the user can see.
- Each household can optionally get a **private role and channel** created by the bot, for shared posts (the board, leaderboard and summaries).
- **DM-only mode** posts nothing in the server at all.
- Tell users plainly that server Administrators can see every channel; only DMs and ephemeral replies are hidden from them.

---

## 3. Chore data model

### Required fields (from the requirements)

| Field | Column | Notes |
|---|---|---|
| Name | `name` | |
| Description | `description` | optional |
| Frequency | `frequency_type`, `interval`, `weekdays`, `day_of_month`, `times_per_day`, `due_times` | see below |
| Last done date | `last_done_at` | timestamp |
| Next expected date | `next_due_at` | calculated |
| Skip count | `skip_count` | resets to 0 when the chore is done |
| Story points | `story_points` | base size, integer (e.g. 1, 2, 3, 5, 8); can grow when skipped (see "Story points" below) |
| Assigned to | `assigned_to` | Discord user ID; empty means unassigned or open to claim |
| Last done by | `last_done_by` | Discord user ID |

### Frequency
- `frequency_type`: `once` | `daily` | `every_n_days` | `weekly` | `monthly`
- `interval`: e.g. every **2** weeks
- `weekdays`: e.g. Mon/Wed/Fri (used with weekly)
- `day_of_month`: e.g. the 1st (used with monthly)
- `schedule_mode`:
  - `fixed`: due on the calendar schedule whenever it was last done (e.g. "trash goes out every Tuesday")
  - `after_done`: due a set time after the last completion (e.g. "clean the fridge 30 days after it was last cleaned")

### Chores done multiple times per day
Some chores happen more than once a day: "feed the dog" twice, "wipe the counters" after every meal. Each due time in a day is a **slot**.

New chore fields:
- `times_per_day`: how many times per day (default **1**). It works with any frequency, e.g. "twice a day, Mon/Wed/Fri".
- `due_times`: optional list of times of day, one per slot, e.g. `08:00, 18:00`.
  - **With times:** each slot is due at its own time.
  - **Without times:** the slots are just a count ("3 times today"), all due by the end of the day.
- `min_gap_minutes`: optional minimum time between completions. This stops a double-click, or two people at once, from filling both "feed the dog" slots for one feeding.

How it behaves:
- **✅ Done** fills the **oldest open slot**. Each completion earns the chore's story points, so a twice-a-day 1-point chore is worth 2 points a day.
- `next_due_at` is the next open slot, which may be later today. `last_done_at` and `last_done_by` refer to the most recent completion.
- **The board** shows progress: `Feed the dog · 1 of 2 today · next 6:00 PM ✅`.
- **Reminders** are sent for each slot.
- **↩ Undo:** a completion can be undone for a few minutes after it's made, in case of mistakes.
- **Missed slots** follow the overdue behavior (below):
  - `skip`: a slot that's still open when the next slot starts (or at the end of the day, for the last slot) is auto-skipped.
  - `roll_forward`: missed slots stay open as overdue, and the next ✅ fills the oldest one first (recorded as late).
- **Rotation** can advance after each slot (morning person and evening person) or once per day. The owner picks with `rotate_by` (`occurrence` | `day`).

### State changes
| Action | Effect |
|---|---|
| **Done (✅)** | `last_done_at = now`, `last_done_by = user`, `skip_count = 0`, recalculate `next_due_at`, advance the assignee (depends on assignment style), record a `done` event with the **current** points (including skip growth and any one-time adjustment), then clear `bonus_points` |
| **Skip** (assignee, owner or admin only) | `skip_count += 1`, move `next_due_at` to the next occurrence, advance the assignee if rotating, record a `skip` event |
| **Overdue** | Depends on the household's overdue behavior (below) |
| **Reassign** | Change `assigned_to`, record a `reassign` event |
| **Take / Release** | Set or clear `taken_by` for the current occurrence, record a `take` or `release` event (see section 4) |

### Overdue behavior (household owner setting)
This answers the question "if a chore gets done a day late, is that a **delay** or a **skip**?" The owner chooses one of two behaviors:

| Setting | What happens when a chore isn't done by its due date | Done a day late counts as |
|---|---|---|
| **`skip`**: skip to the next scheduled date | The chore is automatically skipped: `skip_count += 1`, `next_due_at` moves to the next scheduled occurrence, the assignee advances if rotating, and a `skip` event is recorded | A completion of the *next* occurrence. The missed one stays counted as a skip |
| **`roll_forward`**: move forward one day | The chore stays on the board as overdue. Each day it isn't done, `next_due_at` moves forward by one day. `skip_count` is **not** increased | A **delay**: a `done` event that is 1 day late (its `due_at` is the original due date). No skip is recorded |

Details:
- `overdue_since` is set to the original due date when a chore first becomes overdue, so the board can show "Overdue 3 days" even as `next_due_at` keeps moving. It is cleared when the chore is done.
- After a late completion, the next due date is calculated as usual: from the calendar for `fixed` chores, or from the completion date for `after_done` chores.
- With `roll_forward`, a chore only gets skipped when someone presses ⏭ Skip.
- Summaries and the leaderboard can show late completions separately from skips. Lateness = completion time minus the event's `due_at`.
- Default for new households: `roll_forward`, which is less punishing. The owner can change it.
- **Per-chore override:** the owner can set a chore's own `overdue_behavior`, different from the household's. When a multiple-times-per-day chore is created, the suggested default is `skip`, since making up a missed dog feeding later usually doesn't make sense.

### Story points: changing them, and growth when skipped
Every chore has its **own** point settings. Nothing here is household-wide except the permission setting.

- **Base points** (`story_points`): the chore's normal size. It can be changed at any time with `/chore points` or `/chore edit`.
- **Skip growth** (optional, per chore): a chore that keeps getting skipped gets bigger, because the job really is bigger by then (the bathroom gets dirtier).
  - `skip_point_increase`: points added for each skip, e.g. **+1**. `0` means off.
  - `max_points`: optional cap, e.g. never more than **13**.
- **One-time adjustment** (`bonus_points`, with an optional `bonus_reason`): change the points for **this time only**, when a chore is bigger (or smaller) than usual. Example: after a party, the bathroom is worth +4 more, with the reason "after the party".
  - It can raise or lower the points, but the total never goes below 0.
  - The skip growth cap does not limit it, since someone chose this value on purpose.
  - It clears automatically when the chore is done.
- **Current points** = base points + skip growth (up to the cap) + one-time adjustment. Skip growth is calculated from `skip_count`, so it isn't stored separately.
- **When the chore is done**, the person who did it earns the **current** points. Then the skip count and the one-time adjustment reset, so the chore goes back to its base size.
- **Example:** "Clean bathroom" has base 3, +2 per skip, cap 9. Skipped twice → worth **7**. After a party the owner adds +4 → worth **11**. Sam does it, earns 11, and the chore is back to 3.
- **The board shows the breakdown:** `Clean bathroom · 11 pts (3 + 4 from skips + 4 after the party)`.
- In `roll_forward` households, skips only happen when someone presses ⏭ Skip, so growth only happens then too.

**Who can change points** (household owner setting, `points_edit_policy`). This covers base points, skip increase, the cap and one-time adjustments. The owner can always change them:

| Setting | Who can change a chore's points |
|---|---|
| **`owner`** | Only the household owner |
| **`admins`** (default) | The owner and admins |
| **`assignee`** | The owner, admins, and the chore's current assignee |
| **`members`** | Any member |

Every change is recorded as a `points_change` event (who, old value, new value), shown in `/chore view`. That way nobody can quietly inflate a chore to climb the leaderboard.

---

## 4. Assignment styles

- **Household default style** (`households.default_assignment_style`): the style a new chore starts with.
- **Each chore has its own style** (`chores.assignment_style`), set from the household default when the chore is created. The owner can change it per chore at any time. Changing the household default later does not change existing chores.

| Style | How it works |
|---|---|
| **Fixed** | Always the same one person |
| **Rotation** | Moves through the chore's members, in order, after each done or skip |
| **Claim** | No regular assignee; a chore member clicks **🙋 Take it** to claim it |
| **Random** | A random chore member is picked each new cycle |
| **Balanced** | Goes to the chore member with the fewest story points this period |

### Who each chore can go to (chore members)
Each chore has its own list of **members**: the people it can be assigned to. This is what lets different chores rotate among different people.

- **`everyone`** (default): the whole household. People who join the household later are included automatically.
- **`selected`**: only the people the owner picks. For Rotation, the owner also sets their **order**.

**Example: bathrooms**

| Chore | Style | Chore members |
|---|---|---|
| Private bathroom (Sam's) | Fixed | Sam |
| Shared bathroom (kids') | Rotation | Alex → Jordan → Riley |
| Guest bathroom | Rotation | Everyone, or use **Claim** so anyone can take it |

Rules:
- Rotation, Random, Balanced and Claim only ever pick from the chore's members. Fixed uses the single chosen person.
- Only chore members can **🙋 Take** a chore (owners and admins can too). For everyone else, private views don't show a Take button.
- Anyone in the household can still mark a chore ✅ done, for example to help out. Points go to whoever did it (`last_done_by`).
- **When someone leaves the household**, they're removed from every chore's members. If a chore has no members left, it becomes unassigned and the owner is notified.
- **When someone new joins**, they're added only to `everyone` chores. The owner adds them to `selected` chores with `/chore members`.

### Taking a chore yourself ("I'll do it")
Any member can put their name on a chore **before** doing it, whatever the assignment style. This tells everyone "I've got this one" and keeps two people from doing the same chore.

- **🙋 Take it** button (on unassigned chores on the board, in private views, or via "Ask to take it"), or the `/chore take` command, assigns the chore to you for the **current occurrence only**. For multiple-times-per-day chores, that means the next open slot.
- **Release** (`/chore release`) gives it back if your plans change.
- When the occurrence is done or skipped, the "taken" mark clears, and the chore's normal assignment style picks the next person. Rotation order and fixed assignees are not changed. To change the regular assignee permanently, the owner uses `/chore assign`.
- If a taken chore goes overdue, it stays with the person who took it, and the overdue behavior applies as usual.
- Effective assignee = `taken_by` if set, otherwise `assigned_to`. Reminders go to the effective assignee.
- The **Claim** assignment style is simply a chore with no regular assignee, where the only way to get assigned is to take it.

**Self-assignment policy** (household owner setting, `self_assign_policy`):

| Setting | Unassigned chores | Chores already assigned to someone else |
|---|---|---|
| **`ask`** (default) | Take it instantly | The current assignee gets an **Accept / Decline** button. If they don't answer before the chore is due, the request expires |
| **`open`** | Take it instantly | Take it instantly; the previous assignee is notified |
| **`unassigned_only`** | Take it instantly | Not allowed |
| **`off`** | Not allowed | Not allowed |

Owners and admins can always take or reassign any chore.

---

## 5. Database tables

```
households            id, guild_id, name, owner_id, timezone, privacy_mode (channel|dm_only),
                      channel_id, role_id, default_assignment_style,
                      overdue_behavior (skip|roll_forward),
                      self_assign_policy (ask|open|unassigned_only|off),
                      points_edit_policy (owner|admins|assignee|members), created_at
household_members     household_id, user_id, role (owner|admin|member), joined_at
household_invites     id, household_id, invited_user_id, invited_by, status, expires_at
chores                id, household_id, name, description, frequency_type, interval, weekdays,
                      day_of_month, schedule_mode, times_per_day, due_times, min_gap_minutes,
                      last_done_at, next_due_at, overdue_since, skip_count,
                      story_points, skip_point_increase, max_points, bonus_points, bonus_reason,
                      assigned_to, taken_by, taken_at, last_done_by,
                      assignment_style (fixed|rotation|claim|random|balanced; starts as household default),
                      overdue_behavior (nullable override), rotate_by (occurrence|day),
                      member_pool (everyone|selected), active, created_at
chore_members         chore_id, user_id, position   -- used when member_pool = selected;
                                                     -- position = rotation order
take_requests         id, chore_id, requested_by, current_assignee, due_at,
                      status (pending|accepted|declined|expired), created_at
chore_events          id, chore_id, household_id, type (done|skip|reassign|take|release|points_change),
                      user_id, points, due_at (the occurrence or slot this event is for),
                      details (e.g. old and new point values),
                      auto (true if made by the overdue job), undone_at, created_at
contact_methods       id, user_id, type (email|sms|push), value_encrypted, verified_at
reminder_rules        id, user_id, household_id, channel (discord_channel|discord_dm|sms|email|push),
                      trigger (before_due|at_due|overdue|daily_digest), offset_minutes,
                      time_of_day, enabled
user_settings         user_id, quiet_hours_start, quiet_hours_end, active_household_id
summary_schedules     household_id, cadence (daily|weekly|monthly|off), day_of_week,
                      day_of_month, time_of_day, destination (channel|dm)
board_messages        household_id, channel_id, message_id   -- so the board can be edited in place
```

---

## 6. Discord commands and interactions

### Households
- `/household create <name>`: creates the household; optionally creates a private role and channel
- `/household invite @user`: the invited user accepts with a button
- `/household members` · `/household leave` · `/household switch`
- `/household settings` (owner): timezone, privacy mode, channel, default assignment style, overdue behavior (skip or roll forward), self-assignment policy, who can change points

### Chores
- `/chore add`: opens a form (modal) for name, description and points, then select menus for frequency, assignment style (pre-filled with the household default) and chore members
- `/chore members`: choose who a chore can go to (everyone, or selected people) and set the rotation order
- `/chore edit` · `/chore remove` · `/chore view` · `/chore assign` · `/chore skip`
- `/chore points`: change a chore's points, either **permanently** (base points, skip increase, cap) or **this time only** (a one-time adjustment with an optional reason). Allowed by the household's points policy
- `/chore take` · `/chore release`: put your name on a chore before doing it, or give it back
- `/chores`: the household's chore list (ephemeral)
- `/mychores`: the user's assigned and taken chores (ephemeral)

### The board (marking chores done)
- `/board`: posts the household's chore chart in its channel, grouped as **Overdue / Due today / Upcoming**
- Each chore has a **✅ button**. One click marks it done, and the board updates itself.
- **Exactly two buttons per chore.** The second button depends on the chore, because Discord shows the same buttons to everyone viewing a shared message:

  | Chore state | Buttons on the shared board |
  |---|---|
  | Assigned (or taken) | **✅ Done** · **⏭ Skip** |
  | Unassigned | **✅ Done** · **🙋 Take it** |

- **Only the assignee can skip** (owners and admins too). If anyone else clicks ⏭ Skip, they privately see "Only Sam can skip this." with a **🙋 Ask to take it** button, which follows the self-assignment policy.
- If the person who *took* a chore clicks ⏭ Skip, they privately choose **Release it** (give it back) or **Skip this time**.
- Taken chores show "🙋 Sam has this".
- Points are shown on each chore, with the breakdown: `11 pts (3 + 4 from skips + 4 after the party)`.
- **Private views** (`/mychores`, `/chores`, DM reminders) are built for one person, so they follow the rule exactly: the assignee sees ✅ + ⏭ Skip, and everyone else sees ✅ + 🙋 Take it.
- The ✅ button also appears in DM, channel and email reminders (as a link) wherever possible.
- **Discord limit:** a message can hold at most 25 buttons (5 rows of 5). With 2 buttons per chore that's 10 chores per message (2 chores per row), so bigger households get the board split across several messages.

### Leaderboard and summary
- `/leaderboard [week|month|all]`: posted **in the household channel**; ranks members by story points
- `/summary now`: runs a summary immediately
- `/summary schedule` (owner): sets the cadence, day, time and destination
- Summary contents: chores completed (on time and late, with days late), overdue, skipped (with skip counts), and points per member

### Reminders (each user sets their own)
- `/reminders`: a settings menu to add, remove or toggle rules (channel + trigger + timing)
- `/contact add email|phone|push`: sends a verification code, which the user confirms with `/contact verify`
- `/reminders quiet-hours`

---

## 7. Notifications

There is one **dispatcher** with a separate **provider** for each channel. Reminder jobs call `notify(user, message)`, and the dispatcher sends it through every channel the user has enabled.

| Channel | Provider | Notes |
|---|---|---|
| Discord channel | discord.js | @mentions the user in the household channel |
| Discord DM | discord.js | Fails if the user blocks DMs; fall back to the channel |
| Email | Resend / SES | Needs your own verified domain; include an unsubscribe link |
| SMS | Twilio | Costs money per message. US numbers need **A2P 10DLC registration**. Must support STOP/opt-out |
| Push | **ntfy** first (user installs the ntfy app and subscribes to a private topic) | Later: a web push PWA dashboard. Note that Discord DMs already arrive as phone push notifications |

**Contact info is personal data:** verify it before use, encrypt it at rest, delete it when a user leaves or asks, and never show it to other members.

### Scheduled jobs (pg-boss)
- **Reminder sweep** (every minute): finds rules that are due and sends them, respecting quiet hours and the household timezone
- **Overdue sweep** (every 5 minutes, so slot times are handled promptly, in each household's timezone): with `skip`, auto-skips chores and slots that are past due; with `roll_forward`, sets `overdue_since`, keeps missed slots open, and moves `next_due_at` forward one day for daily-or-less chores
- **Summary jobs:** run on each household's chosen schedule
- **Board refresh:** updates board messages when chores change

---

## 8. Project structure

```
DiscordChoreChartBot/
  src/
    index.ts                 starts the bot and the job worker
    config.ts                environment variables (checked with Zod)
    bot/
      commands/              one file per slash command group
      interactions/          button, select-menu and modal handlers
      views/                 builds the board, leaderboard and summary messages
      deploy-commands.ts     registers slash commands with Discord
    domain/                  pure logic, no Discord code (easy to test)
      schedule.ts            frequency → next_due_at
      assignment.ts          the 5 assignment styles
      chores.ts              done / skip / reassign
      households.ts          membership and permissions
      points.ts              leaderboard and summary totals
    db/
      schema.ts              Drizzle schema
      client.ts
      migrations/
    jobs/                    reminders, overdue sweep, summaries
    notify/                  dispatcher + discord, dm, email, sms and push providers
    lib/                     time, crypto, logging
  tests/
  drizzle.config.ts
  .env.example
  package.json
```

---

## 9. Build phases

| Phase | Goal | Done when |
|---|---|---|
| **1. Foundation** | Project setup, Discord app, Postgres on Neon, deploy to Railway or Fly | A `/ping` command works from the cloud |
| **2. Households** | Create, invite, members, settings, privacy channel | Two households on one server can't see each other's data |
| **3. Chores** | CRUD, frequency, multiple-times-per-day slots, next-due calculation | Unit tests cover every frequency type, both schedule modes, slots, and both overdue behaviors |
| **4. Board + ✅** | Board message, done, skip, take/release, event history | Clicking ✅ or 🙋 updates the chore and the board |
| **5. Assignment styles** | Fixed, rotation, claim, random, balanced; per-chore members and rotation order | Tests for each style and member pool; household default applies to new chores |
| **6. Discord reminders** | Scheduler, reminder rules, channel and DM delivery | Reminders arrive at the right local time |
| **7. Leaderboard + summaries** | `/leaderboard`, `/summary now`, scheduled summaries | Weekly summary posts automatically |
| **8. Email** | Contact verification, email provider | Verified email reminders with unsubscribe |
| **9. SMS + push** | Twilio (after 10DLC registration), ntfy | Texts and push notifications arrive |
| **10. Public launch** | Privacy policy, terms, data deletion, rate limits, Discord bot verification | Ready for other households to add it |

Phases 1–7 make a complete, usable bot for your own household. Phases 8–10 prepare it for other households.

---

## 10. Open questions

1. **Points:** should skipping a chore cost points, or only cost the points you would have earned?
2. **Who can mark done:** anyone in the household, or only the assignee and admins?
3. **Push notifications:** is ntfy acceptable to start, or is a custom phone app or PWA a requirement?
4. **Cost:** SMS has real per-message costs. Should it be a paid or premium feature once the bot is public?
