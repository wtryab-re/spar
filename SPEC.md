# SPAR: Product Spec

SPAR is a mobile app that pairs you with one rival for a focus match. You both start a timer at the same moment and try to stay in the app. The last player still in the match wins.

**Status:** draft v1, 2026-10-07
**Platforms:** iOS and Android

---

## 1. Tech stack

| Layer | Choice |
|---|---|
| App | React Native (Expo), TypeScript |
| Backend | Supabase (Postgres, Auth, Realtime) |
| Screen stays awake | `expo-keep-awake` |
| Detecting leaving and returning to the app | React Native `AppState` |
| Away warnings | `expo-notifications` (local notifications, scheduled on the device) |

The **server is the source of truth** for match timers, pauses, and results. The phone reports events; the server decides outcomes. This stops players from cheating by changing their phone's clock.

---

## 2. Opening the app and matchmaking

### 2.1 Goal prompt
- The first time the user enters the app each day, a prompt asks: **"What are you trying to accomplish today?"**
- Shown **once per day** ("day" resets at midnight in the user's time zone).
- **Required:** the user must enter a goal before they can look for a rival.
- The goal is shown to the rival during the match.
- Each day's goal is saved to the user's goal history.
- A **"Mark achieved"** button lets the user mark today's goal as achieved at any time that day. If it isn't tapped by midnight (user's time zone), the goal is recorded as not achieved.

### 2.2 Matchmaking
- The user taps to find a rival and joins the matchmaking queue.
- Each match is strictly **1v1**.

Matching is tried in this order:

1. **Another searching user.** Pick at random from users currently in the queue.
2. **An online user who isn't searching.** Send an invite ("Someone wants to spar. Accept?") to **several** random users who have SPAR open but aren't in the queue. The **first to accept** gets the match. Every other invite is withdrawn, and those users see a short "Match already taken" message. Invites stay open until someone accepts or the 10-second search window ends. If the window ends with no one accepting, all invites are withdrawn and the searching user is matched with a bot.
3. **A bot.** If no real user is matched within **about 10 seconds** of the search starting, match with a bot (see section 6).

---

## 3. The match

### 3.1 Start
- Both players' **match timers start at 0, at the same moment**.
- The match timer counts **focus time**: time spent inside SPAR during the match.
- The screen is kept awake for the whole match. Normal auto-lock returns when the match ends.

### 3.2 Match screen
- Your match timer
- Your rival's profile
- Your rival's goal for today
- Your rival's live progress (their match timer, and whether it is running or paused)
- Pauses-left counter, e.g. "Pauses left: 2"
- **Admit Defeat / Forfeit** button

### 3.3 Leaving the app (pausing)
Leaving SPAR **pauses** that player's match timer. Leaving includes switching apps, going to the home screen, locking the phone, or closing the app. Returning resumes the timer.

| Time away (single absence) | What happens |
|---|---|
| 2 min | Warning notification: "You're about to forfeit" |
| 3 min | Second warning notification |
| 4 min | Automatic **forfeit**, and the rival wins |

- Each player may leave **at most 3 times** per match.
- Leaving a **4th time** is an immediate forfeit.
- On the 3rd pause, warn the player that the next pause forfeits.
- Warnings are scheduled locally on the phone the moment the player leaves and cancelled when they return, so they arrive even if the app is suspended or offline.

### 3.4 Network drop
- The player's match timer **keeps running**.
- There is a **10-second grace period** to reconnect. During it, the rival sees "Rival reconnecting… 7s".
- If the player reconnects within 10 seconds, the match continues normally.
- If they do not reconnect in time, the match ends:
  - The rival gets a **win**.
  - The disconnected player gets **no loss** recorded.
- Network drops **do not** count toward the 3-pause limit.

### 3.5 Telling a network drop apart from leaving the app
To the server, both look like the phone going silent. The rule is:
- The app sends a **"backgrounded"** event the moment the player leaves. If the phone then goes silent, the server applies the **leaving-the-app** rules from 3.3. This also covers swipe-to-close, because a phone always goes to the background before an app is closed.
- If the phone goes silent **while the app was in the foreground**, the server applies the **network-drop** rules from 3.4.

### 3.6 How a match ends

| Event | Player who caused it | Rival |
|---|---|---|
| Taps Admit Defeat / Forfeit | Loss | Win |
| Away for 4 minutes in a single absence | Loss | Win |
| Leaves the app a 4th time | Loss | Win |
| Network drop longer than 10 s | No result recorded | Win |
| Both players drop at the same time | Higher match focus time wins. If the times are equal, it is a **draw** for both. | |

There is no fixed match length. The match runs until one of the events above happens.

---

## 4. Profiles

| Field | Notes |
|---|---|
| Name | |
| Country | Auto-detected, and the user can change it |
| Photo | Optional. Otherwise the user picks a built-in avatar |
| Wins | |
| Losses | |
| Draws | Counted separately. Not added to wins |
| Win streak | Consecutive wins. A draw keeps the streak alive **and adds 1** to it |
| Play streak | Consecutive days with at least one match played |
| Focus time: today | Resets at midnight in the user's time zone |
| Focus time: this month | Resets on the 1st, in the user's time zone |
| Focus time: all-time | |
| Goals achieved | Count of daily goals the user has achieved |
| Goal history | Past daily goals, each marked achieved or not |

- Network-drop results are **not shown** on profiles.
- A network drop (no result recorded) **does not break** the disconnected player's win streak, and the match **counts toward their play streak**.
- Matches against bots count toward all stats like any other match.

---

## 5. Leaderboard and Loserboard

| Board | Scopes | Sort options |
|---|---|---|
| **Leaderboard** | My country, my continent, world | Most wins (all-time), or most focus time (all-time) |
| **Loserboard** | My country, my continent, world | Most losses (all-time) |

- For now, each board lists **all users**. A top-100 cap may be added later.
- The current user's own rank is always shown and highlighted.
- **Bots never appear** on either board.

---

## 6. Bot opponent

- Used only when no real user is available.
- Users are **never told** they are playing a bot.
- Each bot has a believable generated profile (name, country, avatar, stats). Stats are kept in a **modest mid-range** so their absence from the leaderboards isn't suspicious.
- Each bot also shows a believable generated **daily goal**, since rivals can see each other's goals.
- The bot **gives up at about 60 minutes**, so the user wins if they are still in the match then. A small random variation (for example, 55–65 min) is suggested so the timing doesn't become an obvious pattern.
- If the user forfeits before then, the bot wins.

---

## 7. Later (out of scope for v1)

- In-match chat with the rival, including reporting, blocking, and moderation
- Smarter matchmaking, for example preferring users who just opened the app to find a rival
- Top-100 cap on the leaderboards

---

## 8. Open questions

1. **Accounts:** sign-in methods (e.g. email, Apple, Google), to be decided later.