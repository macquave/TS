# Requirements Document

## Introduction

Trivia Squads is an async-first, team-based trivia web application for friend groups and coworkers. Teams of 3–8 players compete against other teams across games of 5 rounds each. Each round is owned by one randomly selected player who answers 5 themed-category questions within a 24-hour window; no synchronous play is required. This document specifies the requirements for the Minimum Lovable Product (MLP), which validates the core game loop on a responsive web platform with email-based authentication and notifications. Scope is strictly limited to the core loop: team formation, game creation/acceptance, round assignment, round play, substitute flow, and game completion with scoring reveal.

## Glossary

- **System**: The Trivia Squads web application, including its frontend, backend services, database, and background jobs.
- **Auth_Service**: The authentication subsystem, implemented via Supabase email magic link.
- **Team_Service**: The subsystem that manages team creation, membership, and invitations.
- **Game_Service**: The subsystem that manages game creation, acceptance, state transitions, and game completion.
- **Round_Service**: The subsystem that manages round assignment, round state (Not Started, In Progress, Complete, Forfeited), and round-owner substitution.
- **Question_Service**: The subsystem that selects Questions from the Question Bank for each Round, enforces reuse constraints, and persists the selected Questions against Rounds for gameplay.
- **Scoring_Service**: The subsystem that computes round scores, game scores, and win/loss records.
- **Status_Service**: The subsystem that exposes in-game completion state to players while enforcing score hiding.
- **Notification_Service**: The subsystem that sends transactional email notifications.
- **Player**: An authenticated user of the System.
- **Display_Name**: The human-readable name a Player chooses at first sign-in, shown to other Players in place of the Player's email address. Unique per Player but not globally unique.
- **Team**: A named group of 3 to 8 Players. A Player MAY belong to multiple Teams simultaneously.
- **Organizer**: The Player who created a Team. The Organizer has no elevated permissions in the MLP beyond being the implicit creator of the Team.
- **Game**: A single competition between exactly two Teams, consisting of exactly 5 Rounds.
- **Round**: A single themed question set within a Game, consisting of exactly 5 Questions, owned by one Player at a time.
- **Round_Owner**: The Player currently responsible for answering the 5 Questions of a Round. A Round has exactly one Round_Owner at any time (either the Original_Owner or an Active_Substitute, never both counted toward scoring).
- **Original_Owner**: The Player initially assigned to a Round at Game start.
- **Active_Substitute**: A Player assigned to a Round after the Original_Owner failed to complete the Round within their answer window.
- **Category**: A themed topic drawn from the Category_Catalog (Sports, History, Science, Pop Culture, Geography, Music, Film & TV, Food & Drink, Technology, Literature).
- **Category_Catalog**: The fixed set of 8–10 Categories available for Round themes in the MLP.
- **Question**: A single trivia question with exactly one correct answer and a set of answer choices.
- **Question_Bank**: A pre-populated, curated collection of approved Questions stored by Category, from which the Question_Service selects Questions for Rounds. The Question Bank is populated and maintained out-of-band (outside runtime gameplay) and is not generated on demand during gameplay in the MLP.
- **Answer_Window**: The 24-hour period during which a Round_Owner may open and complete their Round. A Round_Owner's Answer_Window starts at Game start (for an Original_Owner) or at substitute assignment (for an Active_Substitute).
- **Question_Timer**: A 30-second countdown that begins when a Round_Owner first views a Question and ends when the Player submits an answer or 30 seconds elapse.
- **Round_Score**: The number of correct answers in a Round divided by the total number of Questions in that Round (always 5). Range: [0.0, 1.0].
- **Sub_Penalty_Multiplier**: The factor 0.8 applied to a Round_Score when the Round was completed by an Active_Substitute rather than the Original_Owner.
- **Effective_Round_Score**: The Round_Score that contributes to the Team's game score, equal to Round_Score if completed by the Original_Owner, or Round_Score × Sub_Penalty_Multiplier if completed by an Active_Substitute, or 0 if forfeited.
- **Game_Score**: The sum of a Team's 5 Effective_Round_Scores. Range: [0.0, 5.0].
- **Game_State**: The lifecycle state of a Game, one of: Pending_Acceptance, Active, Complete.
- **Round_State**: The lifecycle state of a Round, one of: Not_Started, In_Progress, Complete, Sub_Needed, Forfeited.
- **Game_Invite_Link**: A shareable URL created when a Team creates a Game, used by a second Team to accept and start the Game.
- **Team_Invite_Link**: A shareable URL used to invite Players to join a Team.

## Requirements

### Requirement 1: Authentication

**User Story:** As a new or returning Player, I want to sign in using my email address, so that I can access my teams and games without managing a password.

#### Acceptance Criteria

1. WHEN a visitor submits an email address to sign in, THE Auth_Service SHALL send a magic link to that email address and respond with a confirmation state within 5 seconds.
2. WHEN a visitor clicks a valid, unexpired magic link, THE Auth_Service SHALL create an authenticated session for the corresponding Player.
3. IF a magic link has expired or has already been used, THEN THE Auth_Service SHALL display a message indicating the link is invalid and SHALL offer the Player an option to request a new magic link.
4. WHEN a Player with an active session opens the application, THE Auth_Service SHALL identify the Player without requiring re-authentication.
5. WHEN a Player initiates sign-out, THE Auth_Service SHALL terminate the session and return the Player to the sign-in screen.

### Requirement 2: Team Creation and Formation

**User Story:** As a Player, I want to create a team and invite friends or coworkers, so that we can play trivia games together.

#### Acceptance Criteria

1. WHEN an authenticated Player submits a team name, THE Team_Service SHALL create a new Team with that name and record the Player as the Organizer and first member.
2. WHEN a Team is created, THE Team_Service SHALL generate a unique Team_Invite_Link that can be shared to allow other authenticated Players to join the Team.
3. WHEN an authenticated Player opens a valid Team_Invite_Link for a Team that has fewer than 8 members, THE Team_Service SHALL add the Player to the Team as a non-Organizer member.
4. IF an authenticated Player opens a Team_Invite_Link for a Team that already has 8 members, THEN THE Team_Service SHALL reject the join attempt and display a message indicating the Team is full.
5. IF an authenticated Player opens a Team_Invite_Link for a Team of which they are already a member, THEN THE Team_Service SHALL navigate the Player to the Team's page without adding a duplicate membership.
6. THE Team_Service SHALL display the current member list and member count for every Team to each of its members.
7. THE Team_Service SHALL display each Team member identified by their Display_Name and SHALL NOT expose any Player's email address to other Players.
8. WHEN a Player leaves a Team, THE Team_Service SHALL remove the Player from the Team's member list.
9. THE Team_Service SHALL allow a Player to belong to any number of Teams simultaneously.

### Requirement 3: Game Creation and Acceptance

**User Story:** As an Organizer (or any Player acting on behalf of a Team in the MLP), I want to create a game and share an invite link with another team, so that we can start a competition.

#### Acceptance Criteria

1. WHEN a Player who is a member of a Team with at least 3 members initiates Game creation for that Team, THE Game_Service SHALL create a new Game in Game_State Pending_Acceptance with the initiating Team as one side and generate a unique Game_Invite_Link.
2. IF a Player initiates Game creation for a Team that has fewer than 3 members, THEN THE Game_Service SHALL reject the creation and display a message stating that a Team requires at least 3 members to start a Game.
3. WHEN a Player who is a member of at least one Team with at least 3 members opens a valid Game_Invite_Link for a Game in Game_State Pending_Acceptance, THE Game_Service SHALL prompt the Player to select one of their eligible Teams (Teams with at least 3 members, excluding the creating Team), attach the selected Team to the Game as the opposing side, and transition the Game to Game_State Active.
4. WHERE a Player opening a Game_Invite_Link belongs to exactly one eligible Team, THE Game_Service SHALL attach that Team to the Game without prompting for a selection.
5. IF a Player opens a Game_Invite_Link but is not a member of any Team with at least 3 members, THEN THE Game_Service SHALL display a message stating that acceptance requires membership in a Team of at least 3 Players and SHALL offer the Player a path to create or join a Team before returning to the invite.
6. IF a Player opens a Game_Invite_Link and their only eligible Team is the same as the creating Team, THEN THE Game_Service SHALL reject the acceptance and display a message stating that a Team cannot play against itself.
7. IF a Player opens a Game_Invite_Link for a Game that is already in Game_State Active or Complete, THEN THE Game_Service SHALL display a message indicating the Game is no longer accepting opponents and SHALL navigate the Player to the Game's status view if that Player is a member of either participating Team.
8. IF at the moment of Game acceptance any Player is a member of both the creating Team and the accepting Team, THEN THE Game_Service SHALL reject the acceptance and display a message identifying by Display_Name the Player(s) whose overlapping membership prevents the Game from starting.
9. WHEN a Game transitions to Game_State Active, THE Game_Service SHALL timestamp the Game start and SHALL initiate Round assignment as specified in Requirement 4.

### Requirement 4: Round Category Selection and Ownership Assignment

**User Story:** As a Team member, I want rounds to be assigned fairly and randomly at game start, so that each game feels balanced and no one has to coordinate schedules.

#### Acceptance Criteria

1. WHEN a Game transitions to Game_State Active, THE Round_Service SHALL create exactly 5 Rounds for that Game.
2. WHEN Rounds are created for a Game, THE Round_Service SHALL randomly draw exactly 5 distinct Categories from the Category_Catalog, assigning one Category to each of the 5 Rounds of the Game.
3. WHEN Rounds are created for a Game, THE Round_Service SHALL randomly assign each of the 5 Rounds per Team to a Player who is a member of that Team at Game start, such that the maximum number of Rounds assigned to any single Player on the Team is minimized given the Team's size at Game start.
4. WHERE a Team has exactly 5 members at Game start, THE Round_Service SHALL assign each of the 5 Rounds to a distinct Player on that Team.
5. WHERE a Team has fewer than 5 members at Game start, THE Round_Service SHALL assign one or more Players on that Team to own multiple Rounds, ensuring the distribution is as even as possible (no Player is assigned more than ceil(5 / team_size) Rounds).
6. WHERE a Team has more than 5 members at Game start, THE Round_Service SHALL assign exactly 5 distinct Players on that Team as Round_Owners and SHALL record the remaining Players as sitting out for that Game.
7. WHEN a Round is assigned, THE Round_Service SHALL set the Round_State to Not_Started and SHALL record the Original_Owner and the start of the Original_Owner's Answer_Window at the Game start timestamp.
8. IF a Player who was assigned as an Original_Owner is no longer a member of the Team at any point before that Round enters Round_State Complete, THEN THE Round_Service SHALL trigger the substitute flow specified in Requirement 7 for that Round.

### Requirement 5: Question Bank and Round Question Selection

**User Story:** As a Round_Owner, I want to answer fresh, category-appropriate questions without waiting for external services to respond, so that gameplay feels quick and never blocks on a live integration.

#### Acceptance Criteria

1. THE System SHALL maintain a pre-populated Question Bank containing at least 500 approved Questions per Category in the Category_Catalog at MLP launch.
2. WHEN a Game transitions to Game_State Active, THE Question_Service SHALL select 5 Questions from the Question Bank for each of the 10 Rounds in the Game (5 Rounds per Team), drawing randomly from the Question Bank entries matching each Round's assigned Category and subject to the reuse constraint in Acceptance Criterion 3.
3. WHEN selecting Questions for a Round owned by a member of a given Team, THE Question_Service SHALL exclude any Question that has appeared in any of that Team's 10 most recent Completed Games, counted inclusive of both Teams' Rounds in those Games.
4. IF the set of Questions in a Category that satisfy the reuse constraint in Acceptance Criterion 3 contains fewer than 5 Questions, THEN THE Question_Service SHALL relax the constraint progressively (considering the Team's 9 most recent Games, then 8, and so on) until at least 5 eligible Questions are available.
5. THE Question_Service SHALL store the selected Questions against each Round at Game start such that opening the same Round again returns the identical Question set.
6. THE Question_Service SHALL ensure that every Question in the Question Bank has exactly one correct answer, has between 2 and 6 answer choices inclusive, has question text no longer than 200 characters, and appears no more than once in any single Round's 5-Question set.
7. THE System SHALL support replacement or extension of the Question Bank out-of-band (outside the runtime gameplay path) so that Category coverage can be expanded or refreshed without a code deployment.

### Requirement 5b: Known Limitation — Question Bank Freshness

**User Story:** As a product owner, I want the Question Bank's topical freshness to be an accepted MLP tradeoff, so that we do not spec runtime regeneration or currentness validation into the core loop.

#### Acceptance Criteria

1. THE System SHALL NOT automatically regenerate, refresh, or validate the currentness of Questions in the Question Bank during normal runtime operation.
2. THE System SHALL treat Question Bank maintenance (including Category expansion, topical refresh, and quality remediation) as an out-of-band operations concern that is explicitly out of scope for the MLP runtime.

### Requirement 6: Round Play and Answer Timer

**User Story:** As a Round_Owner, I want to answer my 5 questions with a 30-second timer per question, so that gameplay is quick and feels fair across Players.

#### Acceptance Criteria

1. WHEN a Round_Owner opens a Round during their Answer_Window, THE Round_Service SHALL display a preview screen showing the Round's Category, the number of Questions, the per-Question Question_Timer duration, and an explicit "Start Round" confirmation control, and SHALL NOT display any Question content until the Round_Owner activates the "Start Round" control.
2. IF at the moment a Round_Owner attempts to activate the "Start Round" control the remaining Answer_Window is less than 2 minutes and 30 seconds (the total time required for 5 Questions at 30 seconds each), THEN THE Round_Service SHALL prevent the Round from starting, SHALL display a message stating there is not enough time remaining to complete the Round, and SHALL treat the Round as if its Answer_Window had elapsed, transitioning it to Round_State Sub_Needed as specified in Requirement 7.
3. WHEN the Round_Owner activates the "Start Round" control with at least 2 minutes and 30 seconds remaining in the Answer_Window, THE Round_Service SHALL transition the Round to Round_State In_Progress and SHALL display the first Question.
4. WHEN a Round_Owner views a Question, THE Round_Service SHALL start a Question_Timer for that Question set to 30 seconds.
5. WHEN a Round_Owner submits an answer for a Question before the Question_Timer reaches 0, THE Round_Service SHALL record the submitted answer as the final answer for that Question and SHALL stop the Question_Timer.
6. IF a Question_Timer reaches 0 without a submitted answer, THEN THE Round_Service SHALL record the Question as unanswered and SHALL mark the Question as incorrect.
7. IF a Round_Owner navigates away from a Question after the Question_Timer has started and before submitting an answer, THEN THE Round_Service SHALL continue the Question_Timer and SHALL apply the rule in Acceptance Criterion 6 if the Question_Timer reaches 0.
8. WHEN all 5 Questions in a Round have been resolved (answered or timed out), THE Round_Service SHALL transition the Round to Round_State Complete and SHALL compute the Round_Score as the count of correct answers divided by 5.
9. ONCE a Round has transitioned to Round_State In_Progress, THE Round_Service SHALL allow the Round to complete all 5 Questions regardless of whether the Answer_Window elapses during play.
10. IF a Round_Owner attempts to open a Round whose Answer_Window has elapsed and which is not yet in Round_State In_Progress, THEN THE Round_Service SHALL prevent the Player from viewing the preview or any Question content and SHALL display a message indicating the Round is no longer available.
11. THE Round_Service SHALL prevent a Round_Owner from re-attempting any Question whose Question_Timer has started.

### Requirement 7: Substitute Flow

**User Story:** As a Team member, I want another teammate to be able to complete a round if the original owner misses their window, so that our team does not automatically forfeit the round.

#### Acceptance Criteria

1. IF a Round's Answer_Window elapses while the Round is not in Round_State Complete, THEN THE Round_Service SHALL transition the Round to Round_State Sub_Needed and SHALL select an Active_Substitute as specified in Acceptance Criterion 2.
2. WHEN an Active_Substitute is required for a Round, THE Round_Service SHALL select an eligible substitute at random from the set of Team members who currently have no open Round in this Game, where "open Round" means a Round in Round_State Not_Started, In_Progress, or Sub_Needed for which the Player is the current Round_Owner. Eligible substitutes therefore include Team members who were not assigned any Round at Game start (sitters) and Team members whose previously assigned Rounds have all reached Round_State Complete.
3. THE Round_Service SHALL NOT select as an Active_Substitute any Player who is the current Round_Owner of the Round needing a substitute, including the Original_Owner whose missed window triggered the substitute flow.
4. IF two or more Rounds on the same Team transition to Round_State Sub_Needed at the same moment, THEN THE Round_Service SHALL perform substitute assignment sequentially in ascending order of Round number, re-evaluating the eligible-substitute set against the updated Round_Owner state after each assignment.
5. WHEN an Active_Substitute is assigned, THE Round_Service SHALL start a new 24-hour Answer_Window for the Active_Substitute beginning at the moment of assignment.
6. IF no eligible Team member is available to serve as an Active_Substitute, THEN THE Round_Service SHALL transition the Round to Round_State Forfeited and SHALL record the Effective_Round_Score for that Round as 0.
7. IF an Active_Substitute's Answer_Window elapses while the Round is not in Round_State Complete, THEN THE Round_Service SHALL transition the Round to Round_State Forfeited and SHALL record the Effective_Round_Score for that Round as 0.
8. WHEN an Active_Substitute completes a Round, THE Scoring_Service SHALL compute the Effective_Round_Score for that Round as Round_Score × Sub_Penalty_Multiplier (0.8).
9. THE Round_Service SHALL record at most one Active_Substitute per Round; if an Active_Substitute's window elapses without completion, THE Round_Service SHALL NOT select a second Active_Substitute.

### Requirement 8: In-Game Status View and Score Hiding

**User Story:** As a Player, I want to see completion progress for both teams during the game but not see any scores until everyone is finished, so that the reveal at the end is meaningful and nobody gets an unfair advantage.

#### Acceptance Criteria

1. WHILE a Game is in Game_State Active, THE Status_Service SHALL display, for each Round in the Game, one of the following completion states: "Complete" (Round_State Complete), "In Progress with X hours left" (Round_State Not_Started or In_Progress with a remaining Answer_Window), "Sub Needed" (Round_State Sub_Needed), "Forfeited" (Round_State Forfeited), or "Not Started" (Round_State Not_Started and Answer_Window has not elapsed).
2. WHILE a Game is in Game_State Active, THE Status_Service SHALL NOT expose Round_Score, Effective_Round_Score, Game_Score, individual Question correctness, submitted answer values, or any data derived from scores for that Game in any API response or UI surface.
3. WHILE a Game is in Game_State Active, THE Round_Service SHALL NOT expose to any Player other than the Round_Owner the Questions, submitted answers, or answer correctness for a Round.
4. WHEN a Game transitions to Game_State Complete, THE Status_Service SHALL make Round_Score, Effective_Round_Score, Game_Score, per-Question correctness, and the win/loss outcome visible to members of both participating Teams.
5. IF a Player who is not a member of either participating Team attempts to view a Game's status, THEN THE Status_Service SHALL deny access.

### Requirement 9: Game Completion and Win/Loss Recording

**User Story:** As a Team member, I want the game to complete and scores to be revealed as soon as every round is resolved, so that we can see the outcome quickly.

#### Acceptance Criteria

1. WHEN every Round in a Game has reached Round_State Complete or Round_State Forfeited, THE Game_Service SHALL transition the Game to Game_State Complete.
2. IF all Rounds for both Teams reach a terminal state (Complete or Forfeited) before 48 hours after Game start have elapsed, THEN THE Game_Service SHALL transition the Game to Game_State Complete immediately upon the last terminal transition.
3. WHEN a Game transitions to Game_State Complete, THE Scoring_Service SHALL compute each Team's Game_Score as the sum of that Team's 5 Effective_Round_Scores.
4. WHEN a Game transitions to Game_State Complete and the two Teams have different Game_Scores, THE Scoring_Service SHALL record a win for the Team with the higher Game_Score and a loss for the other Team.
5. WHEN a Game transitions to Game_State Complete and the two Teams have equal Game_Scores, THE Scoring_Service SHALL record a tie for both Teams.
6. THE Scoring_Service SHALL display each Team's cumulative win/loss/tie record aggregated across all of that Team's completed Games.

### Requirement 10: Scoring Invariants

**User Story:** As a product owner, I want scoring to always produce valid, bounded values, so that leaderboards and game outcomes are trustworthy.

#### Acceptance Criteria

1. THE Scoring_Service SHALL compute each Round_Score as (count of correct answers in the Round) / 5.
2. THE Scoring_Service SHALL compute each Effective_Round_Score as Round_Score if the Round was completed by the Original_Owner, Round_Score × 0.8 if the Round was completed by an Active_Substitute, or 0 if the Round is Forfeited.
3. THE Scoring_Service SHALL ensure that every Round_Score is in the closed interval [0.0, 1.0].
4. THE Scoring_Service SHALL ensure that every Effective_Round_Score is in the closed interval [0.0, 1.0].
5. THE Scoring_Service SHALL ensure that every Team's Game_Score equals the sum of that Team's 5 Effective_Round_Scores and is in the closed interval [0.0, 5.0].

### Requirement 11: Round Ownership Invariants

**User Story:** As a product owner, I want round ownership to be unambiguous at all times, so that scoring cannot double-count or lose a round.

#### Acceptance Criteria

1. THE Round_Service SHALL ensure that every Game in Game_State Active or Complete has exactly 5 Rounds.
2. THE Round_Service SHALL ensure that every Round has exactly one Round_Owner at any given time, equal to the Original_Owner unless an Active_Substitute has been assigned, in which case the Round_Owner is the Active_Substitute.
3. THE Scoring_Service SHALL ensure that each Round contributes exactly one Effective_Round_Score to its Team's Game_Score, attributed to either the Original_Owner or the Active_Substitute, never both.
4. THE Round_Service SHALL ensure that within a single Game, no Player on a Team is simultaneously the active Round_Owner of more Rounds than required by the minimum-imbalance assignment defined in Requirement 4, Acceptance Criterion 3.

### Requirement 12: Email Notifications

**User Story:** As a Player, I want to receive email notifications at key moments in a game, so that I do not forget to complete my round or miss the final results.

#### Acceptance Criteria

1. WHEN a Game transitions to Game_State Active, THE Notification_Service SHALL send a "Round Assigned" email to each Round_Owner containing the Round's Category, the Answer_Window deadline, and a direct link to open the Round.
2. WHEN a Round_Owner's Answer_Window has 12 hours remaining and the Round is not in Round_State Complete, THE Notification_Service SHALL send a "12-Hour Reminder" email to that Round_Owner.
3. WHEN a Round_Owner's Answer_Window has 2 hours remaining and the Round is not in Round_State Complete, THE Notification_Service SHALL send a "2-Hour Urgent Reminder" email to that Round_Owner.
4. WHEN an Active_Substitute is assigned to a Round, THE Notification_Service SHALL send a "Substitute Needed" email to the Active_Substitute containing the Round's Category, the new Answer_Window deadline, and a direct link to open the Round.
5. WHEN a Game transitions to Game_State Complete, THE Notification_Service SHALL send a "Game Complete Digest" email to every member of both participating Teams containing each Team's Game_Score, per-Round Effective_Round_Scores, and the win/loss/tie outcome.
6. THE Notification_Service SHALL NOT send per-Round completion emails during a Game.

### Requirement 13: Responsive Web Platform

**User Story:** As a Player, I want the app to work well on my phone as well as my laptop, so that I can play or check status from wherever I am.

#### Acceptance Criteria

1. THE System SHALL render all Player-facing views as a responsive web application that adapts layout to viewport widths from 320 pixels to 1920 pixels.
2. WHEN a Player opens any Player-facing view on a mobile device with a viewport width of 320 to 767 pixels, THE System SHALL render the view without horizontal scrolling and with tap targets of at least 44 by 44 pixels for primary interactive elements.
3. THE System SHALL meet WCAG 2.1 Level AA contrast requirements for all text and primary interactive elements.

### Requirement 14: Known Limitation — Team Size Asymmetry

**User Story:** As a product owner, I want team size asymmetry to be an accepted tradeoff rather than compensated for, so that MLP scope stays focused on validating the core loop.

#### Acceptance Criteria

1. THE Scoring_Service SHALL NOT apply any handicap, multiplier, or adjustment based on the relative sizes of the two Teams in a Game.
2. THE System SHALL display a note on the Game creation and acceptance screens stating that team size differences are not compensated for in the MLP.

### Requirement 15: Player Profile and Display Name

**User Story:** As a Player, I want a display name that shows up to my teammates and opponents, so that my email address is not exposed and so that people can recognize me in the app.

#### Acceptance Criteria

1. WHEN a Player completes their first successful authentication, THE Auth_Service SHALL require the Player to choose a Display_Name before accessing any Team or Game surface.
2. THE Auth_Service SHALL reject an empty Display_Name or a Display_Name longer than 30 characters and SHALL prompt the Player to enter a valid value.
3. WHEN an authenticated Player navigates to their profile, THE System SHALL allow the Player to edit their Display_Name, subject to the length and non-empty constraints in Acceptance Criterion 2.
4. WHEN a Player updates their Display_Name, THE System SHALL reflect the updated Display_Name in all Team member lists, Game status views, and Game Complete Digest emails going forward.
5. THE System SHALL NOT display a Player's email address to any other Player in any Team, Game, or Notification surface.
6. WHERE a Player's Display_Name is referenced in a Notification email addressed to that Player, THE Notification_Service MAY address the Player by their Display_Name in the email body.
7. IF a Player attempts to join a Team or to update their Display_Name to a value that exactly matches the Display_Name of another current member of any Team to which they would belong, THEN THE System SHALL reject the action and SHALL prompt the Player to choose a distinct Display_Name for that Team context.
