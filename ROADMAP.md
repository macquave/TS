# Trivia Squads — Product Roadmap

## Philosophy

Win on mechanics first. The core game loop — async rounds, time pressure, team stakes, score reveal — has to feel tight and fair before any engagement layer on top of it means anything. Every phase below builds on a working previous phase.

---

## MLP — Minimum Lovable Product *(in progress)*

**Goal:** Validate the core game loop with a small group of real users.

**What's in scope:**
- Email magic-link auth with display names
- Team creation and invite-link-based team formation (3–8 players)
- Game creation and invite-link-based game acceptance
- 5 rounds per game, randomly assigned to team members
- Multiple-choice questions only, drawn from a pre-seeded question bank
- 30-second per-question timer, server-authoritative
- 12-hour answer windows — games complete within a single day
- Substitute flow when a round owner misses their window
- Score hard-lock during active games; full reveal on completion
- Email notifications: round assigned, 6-hour reminder, 2-hour reminder, sub needed, game complete digest
- Responsive web app (320px–1920px, WCAG 2.1 AA)

**Success criteria:** A friend group can create teams, play a full game start to finish, and feel the tension of the score reveal.

---

## P1 — Question Depth and Variety

**Goal:** Make the question experience feel fresh, themed, and replayable.

**What's in scope:**
- **Themed rounds:** Each round is given a specific theme within its category (e.g., "90s Action Movies" within Film & TV, "World Capitals" within Geography) rather than a broad category alone. Questions within a round feel cohesive and intentional.
- **Fill-in-the-blank questions:** A second question type alongside multiple choice. Answer matched case-insensitively with a small set of accepted variants per question. Timer extended to 45 seconds for fill-in-the-blank questions.
- **Expanded question bank:** Grow to ≥ 1,000 questions per category with tighter theme tagging.
- **Question reporting:** Players can flag a question as incorrect or poorly worded. Flagged questions are reviewed out-of-band before being retired or corrected.

**Why this comes before social features:** Retention depends on questions feeling good. A social loop built on stale or unfair questions collapses quickly.

---

## P2 — Quiz Master and User-Generated Content

**Goal:** Turn engaged players into content creators, building a long-term question supply that doesn't depend solely on internal curation.

**What's in scope:**
- **Quiz Master role:** A separate account type that can submit original questions to the platform. Quiz Masters go through a lightweight onboarding (submission guidelines, example questions).
- **Submission and review pipeline:** Questions submitted by Quiz Masters enter a review queue. Approved questions are tagged with the Quiz Master's display name and enter the live question bank.
- **Attribution in-game:** When a player answers a Quiz Master-authored question, the question card credits the Quiz Master by name. This incentivizes high-quality contributions.
- **Quiz Master leaderboard:** Ranks Quiz Masters by question usage count, approval rate, and player rating.

**Why user-generated content is a long-term bet:** Platforms that make users feel like contributors — not just consumers — build compounding engagement. A great Quiz Master question being answered by thousands of players is a powerful retention hook.

---

## P3 — Social Discovery and Matchmaking

**Goal:** Expand beyond friend groups to give players opponents when they don't have a full team ready, and build the "everyone plays trivia" network effect.

**What's in scope:**
- **Open lobby:** Teams can post a game to a public lobby with optional filters (category preference, team size range). Other teams can browse and accept without needing an invite link.
- **Quick match:** A single player or small group can be matched into a pickup game against a similarly-sized team. The system balances team sizes as closely as possible.
- **Player profiles and stats:** Win/loss/tie records, favorite categories, games played — a light public profile that players can share.
- **Team reputation:** Teams accumulate a record visible in the lobby, giving opponents a sense of who they're playing.

**Why this comes after P2:** A matchmaking lobby is only compelling if there are enough active games and high-quality questions to make random opponents interesting. P1 and P2 build that foundation.

---

## Future Considerations *(no timeline)*

- **Push notifications** (mobile web / PWA) to replace or supplement email reminders
- **Seasonal events and themed tournaments** (e.g., March Madness bracket, Holiday Trivia Cup)
- **Team chat** — a lightweight in-game thread per game for trash talk and reactions after the reveal
- **Native mobile apps** once web product-market fit is confirmed
- **Paid Quiz Master tiers** — revenue sharing or premium placement for top contributors
