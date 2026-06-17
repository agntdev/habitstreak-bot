# HabitStreak Bot — Refined brief

Summary
- A private-chat Telegram bot that helps a single user track daily habits and streaks. Users create habits with a per-day target (N×/day). The bot presents a single editable main message with inline buttons for all interactions after setup; tapping "✓ Done" records an occurrence. A day counts as completed only when the user reaches that habit's daily target; streaks are consecutive completed days in the user's timezone.

Audience
- Individual users who want a lightweight, privacy-first habit tracker inside Telegram. No group support.

Core entities
- User: telegram_user_id, display_name, timezone, main_message_id (optional), created_at
- Habit: id, user_id, name, target_per_day (integer ≥1), created_at, active (bool)
- DailyRecord: id, habit_id, date (YYYY-MM-DD in user timezone), count (integer), completed (bool)
- Streaks derived from DailyRecord.completed by checking consecutive dates

Integrations & notification targets
- No external integrations. All data stored locally on the bot host (no external APIs).
- Notifications/push reminders: intentionally out of scope for v1.

Interaction flows
- General principle: after initial setup most interaction happens by tapping inline buttons; the bot edits its single main message in-place rather than sending new messages.

Commands (initial/setup use only)
- /start
  - Welcome message; if user is new ask for timezone (user types a timezone string); create main menu message and store its message_id.
  - If user already exists, send (edit) the stored main menu message showing habits and quick actions.
  - Timezone input: accept an IANA timezone (e.g. "Europe/London") or simple UTC offset ("UTC+2"). If parsing fails, ask again.

- /add
  - Multi-step flow (typing allowed during this flow):
    1) Ask for habit name (user types text).
    2) Ask for daily target: present inline buttons for 1, 2, 3, 4, 5 and a "Custom" button. If "Custom" chosen, user types a number (max 20).
    3) Confirm and create habit.
  - After creation the bot edits the main message to include the new habit.

- /list
  - Edits the main message to list all habits with their today progress (count/target) and current streak.

- /check
  - Primary interaction is via inline buttons on the main message: each habit row has a "✓ Done" button.
  - When tapped:
    - Increment today's count for that habit (create DailyRecord if missing).
    - If count < target: update progress in the main message (e.g. "2/3 today").
    - If count reaches target: mark DailyRecord.completed=true, increment current streak, update longest streak if beaten, and visually mark the habit as completed for today and disable or hide "✓ Done" for that habit for the rest of the day.
  - Every button action edits the same main message (no extra messages visible in chat).

- /stats
  - Edits main message (or presents a compact modal view via edited message) showing: today's overall progress (how many habits fully completed of total), each habit's longest streak and current streak, and overall longest streak across all habits.

Main message layout (editable)
- Short friendly header line.
- For each habit a single row: "Habit name — X/Y today — 🔥 current_streak — 🏆 longest_streak" plus inline buttons: [✓ Done] (disabled/removed when completed), [Details] (optional: shows per-day counts), [Delete/Edit] (Edit opens a short flow — rename or change target; typing allowed if needed).
- Footer buttons: [Add habit], [Stats], [Help]. All operate via inline callbacks and cause the bot to edit the same message.

Persistence
- Local SQLite database stored on the bot host. Schema corresponds to the core entities above.
- The bot stores main_message_id for each user so it can edit the same message on subsequent interactions and after reinstall.
- The bot process must run continuously (long-polling) so the DB remains authoritative; uninstalling the bot from a client does not remove DB records.

Day boundary & timezone
- The bot evaluates days using midnight-to-midnight in the user's configured timezone (set during /start). This is used for DailyRecord.date and for streak calculations.

Streak rules
- A day's completion is only counted when the habit's target_per_day is fully met that day. Current streak increases when consecutive dates are completed; missing any calendar day in the user's timezone breaks the streak.

UI & tone
- Short, friendly, minimal, emoji-light. All primary interactions via inline buttons. After initial setup the user should not need to send slash commands or free text (except the typed inputs allowed during /add and the timezone entry during /start if needed).

Long-polling
- The bot uses getUpdates (long-polling) only. No webhooks.

Payments
- None.

Non-goals (v1)
- No push/scheduled reminders or notifications.
- No webhook-based architecture.
- No external API calls or third-party tracking.
- No group chat support. No multi-user shared habits.

## Assumptions & defaults
- Database: SQLite file on the bot host — simple, zero-dependency and satisfies "local data" requirement.
  Rationale: SQLite is persistent across restarts and easy to deploy for single-user/small-scale use.

- Timezone input: accept IANA tz names or UTC offsets; if user input cannot be parsed, fall back to server timezone and ask again once.
  Rationale: IANA tz is precise for day boundaries; simple fallback avoids blocking setup.

- Day definition: midnight-to-midnight in the user's timezone (date strings stored as YYYY-MM-DD in that timezone).
  Rationale: aligns streaks with user's local calendar.

- Frequency UI defaults: show quick buttons for targets 1–5; allow a "Custom" numeric entry up to 20/day.
  Rationale: covers most common habits while preventing absurd counts.

- Streak completion rule: require full daily target (N×/day) before the day counts as completed.
  Rationale: matches owner preference and supports clear streak semantics.

- Main message handling: bot creates/stores one main menu message per user (message_id) and edits that message for every UI change; if the stored message is missing (message deleted), the bot will create a new main message and store its id.
  Rationale: guarantees the "edit same message" requirement and recovers from deletions.

- Limits: max habit name length 100 chars; max active habits per user 100.
  Rationale: reasonable UI limits to keep the single message readable and to simplify storage.

- Concurrency: atomic increments for checks using DB transactions to avoid race conditions from rapid button presses.
  Rationale: ensures counts never exceed expected values or corrupt daily records.

- Delete/Edit flows: basic rename and change target flows allowed via /add-style flows; deleting requires a confirmation button.
  Rationale: needed for production completeness without adding heavy UI complexity.

Implementation notes for the builder (actionable)
- Use python-telegram-bot or equivalent that supports long-polling and message editing via callback queries.
- Persist Telegram user id -> user record; store per-user main message id; always answer callback queries promptly (answerCallbackQuery) and then edit the main message.
- Compute streaks lazily on read from DailyRecord table; update longest_streak whenever a completed day extends current streak.

If you want any of the defaults changed (DB engine, stricter timezone picker, allow scheduled reminders, or different UI for frequency selection), tell me the one change now; otherwise I'll proceed with this brief as the build spec.