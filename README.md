# DiscordChoreChartBot

An open-source Discord bot for running household chore charts.

> **Status:** In planning. No code yet. See [PLAN.md](PLAN.md) for the full design.

## Planned features

- **Households:** several private households can share one Discord server
- **Chores:** name, description, frequency, last done date, next due date, skip count, story points, assignee, and last done by
- **Assignment styles** per chore (fixed, rotation, claim, random, or balanced), each with its own list of members it can go to
- **One-click ✅** to mark a chore done from the chore board, and **🙋 Take it** to claim one before doing it
- **Flexible points:** chores can grow in points when skipped, and can be adjusted one time (e.g. after a party)
- **Reminders** (each person chooses their own): Discord channel, Discord DM, email, SMS, or push
- **Point goals** per person, and an **optional leaderboard**
- **Summaries** on a schedule the owner sets, or on demand with `/summary now`

## Tech stack

TypeScript · discord.js · PostgreSQL · Drizzle ORM · pg-boss

## Contributing

The project is just getting started. Issues and ideas are welcome.

## License

[MIT](LICENSE)
