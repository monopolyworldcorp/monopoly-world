# Monopoly World — Multiplayer Audit Report
**Date:** 17 March 2026  
**Scope:** Game Room System, Social/Friend System, Multiplayer Gameplay Flow

---

## 🔴 CRITICAL ISSUES (Game-Breaking)

### 1. Auction Not Synced in Multiplayer
**Location:** `G.auc()`, `G.bid()`, `G._rAuc()` (lines ~14600-14723)  
**Problem:** The auction system uses a local JavaScript variable `aucD` that is NEVER synced to Firebase. In multiplayer, when a player lands on an unowned property and starts an auction, ONLY that player's client sees the auction overlay. Other players cannot bid.  
**Impact:** Core Monopoly mechanic broken — auctions are a key part of the game where ALL players compete.  
**Fix:** Store auction state in Firebase (`rooms/{code}/auc`), use Firebase transactions for atomic bidding, sync timer via `endTime` so all clients count down together.

### 2. Disconnected Player Blocks Game Forever
**Location:** `G.end()` idle timer (lines ~15418-15428)  
**Problem:** The idle timer (90s auto-end) only runs on the CURRENT player's client. If that client disconnects (browser crash, internet drop), NO other client has a timeout mechanism. The game is stuck forever — no one can skip the disconnected player's turn.  
**Impact:** Any player disconnect during their turn = permanent game freeze for all players.  
**Fix:** Add cross-client stale-turn detection. All clients monitor `GS._turnStartTs`. If a turn exceeds 120 seconds and the current player isn't responding, the next active player in turn order force-ends the turn.

### 3. Host Disconnect Orphans Room (Lobby)
**Location:** `Rm.create()` onDisconnect (line ~11106)  
**Problem:** Host's `onDisconnect()` only removes their player entry, NOT the room itself. If the host's browser crashes during lobby, the room persists in Firebase with no host. Non-host players are stuck — they can't start the game.  
**Impact:** Orphaned rooms accumulate in Firebase; players get stuck in dead lobbies.  
**Fix:** Host should set `onDisconnect().remove()` on the entire room during lobby phase, and cancel it once the game starts (so mid-game host disconnect doesn't delete the room).

### 4. Invite Auto-Join Doesn't Actually Join
**Location:** `checkInvites()` (lines ~6752-6771)  
**Problem:** When accepting an invite, the code only fills in the room code input and calls `goRoom()` (which navigates to the room screen). It does NOT call `Rm.joinRoom()`. The user still has to manually click Join.  
**Impact:** Invitation system appears broken from user perspective.  
**Fix:** Call `Rm.joinRoom(inv.roomCode)` directly instead of navigating and filling an input.

### 5. Race Condition on Room Join
**Location:** `Rm.joinRoom()` (lines ~11113-11158)  
**Problem:** Room capacity check is read-then-write (not atomic). Two players joining simultaneously could both pass the `length >= maxPlayers` check and both join, exceeding max capacity.  
**Impact:** Rooms can end up with more players than max slots.  
**Fix:** Use Firebase transaction for atomic player-count check and join.

---

## 🟠 SERIOUS ISSUES (Functionality Broken)

### 6. Duplicate UID Can Join Same Room
**Location:** `Rm.joinRoom()` (lines ~11113-11158)  
**Problem:** No check prevents the same Firebase UID from joining a room twice (e.g., multiple browser tabs). Each tab generates a new `myId = 'p_' + Date.now()`, so the duplicate passes all checks.  
**Impact:** One user can occupy multiple slots, breaking the game.  
**Fix:** Check if any existing player in the room has the same `uid` before allowing join.

### 7. sentRequests Not Synced to Cloud
**Location:** `sendFriendRequest()` (lines ~6563-6584)  
**Problem:** After sending a friend request, `sentRequests` array is saved only to localStorage via `_saveProfileObj()`. The function does NOT call `syncProfileToCloud()`. If user clears cache, they can send duplicate requests. Also, `acceptFriend()` reads `sentRequests` from cloud data to clean it up — but it was never synced there.  
**Impact:** Duplicate friend requests; `sentRequests` cleanup fails on accept.  
**Fix:** Call `syncProfileToCloud(prof)` after updating sentRequests.

### 8. Remove Bot Button Missing in Online Mode
**Location:** `Rm._render()` (line ~11331)  
**Problem:** The "Remove Bot" button uses condition `(p.isBot && isHost && offline)` — it only shows in offline mode. In online rooms, the host cannot remove bots.  
**Impact:** Host can add bots to online rooms but cannot remove them.  
**Fix:** Change condition to `(p.isBot && isHost)` to work in both online and offline.

### 9. Entry Fee Security — Client-Side Only
**Location:** `Rm.create()`, `Rm.joinRoom()`, `Rm.leave()`  
**Problem:** Entry fee deduction/refund is entirely client-side (`getECoinBalance()` / `setECoinBalance()`). A malicious client can bypass fees by manipulating localStorage.  
**Impact:** Competitive modes with real entry fees can be exploited.  
**Note:** Proper fix requires Firebase security rules / Cloud Functions (server-side validation). Added basic server-side balance validation.

---

## 🟡 MODERATE ISSUES

### 10. Board.draw() Called on Every Firebase Update
**Location:** `Rm._upd()` (line ~11448)  
**Problem:** When already in game, every Firebase GS update calls `Board.draw()` which redraws the ENTIRE board canvas. Only `Board.tokens()` and `Board.dots()` need updating.  
**Impact:** Performance degradation, especially on mobile — unnecessary full redraws on every remote player's action.  
**Fix:** Remove `Board.draw()` from the "already in game" update path.

### 11. Invite Uses Blocking `confirm()` Dialog
**Location:** `checkInvites()` (line ~6760)  
**Problem:** Uses native `confirm()` which is a blocking modal. If multiple invites arrive, they stack up and block the UI. Also inconsistent with the game's polished UI.  
**Impact:** Poor UX, potential UI freeze with multiple invites.  
**Fix:** Replace with in-game toast notification with join button.

### 12. No Host Migration
**Problem:** If the host disconnects during gameplay, there's no mechanism to promote another player to host. The game can continue (GS is synced), but room-level operations (kick, slot changes) become unavailable.  
**Note:** Complex to implement properly; room cleanup on disconnect handles the lobby case.

### 13. No Reconnection Mechanism
**Problem:** If a player disconnects mid-game, they can't rejoin. Their turns will time out via the idle timer (once we add cross-client timeout), but their assets and properties are stuck.  
**Note:** Would require matching by UID on reconnect and restoring their session.

---

## Summary of Fixes Applied

| # | Fix | Severity | Status |
|---|-----|----------|--------|
| 1 | Multiplayer auction sync via Firebase | 🔴 Critical | ✅ Applied |
| 2 | Cross-client idle timeout for disconnected players | 🔴 Critical | ✅ Applied |
| 3 | Host disconnect room cleanup | 🔴 Critical | ✅ Applied |
| 4 | Invite auto-join | 🔴 Critical | ✅ Applied |
| 5 | Atomic room join (race condition) | 🔴 Critical | ✅ Applied |
| 6 | Duplicate UID prevention | 🟠 Serious | ✅ Applied |
| 7 | sentRequests cloud sync | 🟠 Serious | ✅ Applied |
| 8 | Remove Bot button in online mode | 🟠 Serious | ✅ Applied |
| 9 | Board.draw() performance optimization | 🟡 Moderate | ✅ Applied |
| 10 | Invite UX improvement (toast instead of confirm) | 🟡 Moderate | ✅ Applied |
| 11 | Turn start timestamp for timeout detection | 🔴 Critical | ✅ Applied |
