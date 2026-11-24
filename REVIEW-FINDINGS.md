# Code review findings and fixes

This file records the code reviews of the Massage Chair Queue and how each finding was fixed. Reviews repeat, with each round checking the previous round's fixes, until a round finds no real bugs.

| Round | Date | Found | Status |
|---|---|---|---|
| 1 | 24 Nov 2026 | 8 (6 important, 2 minor) | Fixed in `e1eced5` |
| 2 | 24 Nov 2026 | 10 (5 important, 5 minor) | Fixed in `050ada7` |
| 3 | 24 Nov 2026 | 7 (1 important, 6 minor) | Fixed |
| 4 | 24 Nov 2026 | 4 (1 important, 3 minor) + 3 polish | Fixed |
| 5 | 24 Nov 2026 | 6 (2 important, 4 minor) + 4 polish | Fixed (see below) |

The shared phone version is [shared/massage-queue.html](shared/massage-queue.html), and the single-screen version is [index.html](index.html).

# Fifth review

The fifth review confirmed that most of the fourth round's fixes work:
- Reconnecting never stacks listeners and stops after `revoked`.
- The dead-connection flag clears once the connection is back.
- The read-only switch disables every button.
- The request id doesn't affect older pages.
- The time limits and pointer listeners are safe.
- `index.html` has no regressions.

It found that R4-1 didn't cover every case, and that a sound-button bug from round 2 had been missed in every round since.

| # | Severity | Finding | Fixed in |
|---|---|---|---|
| R5-1 | Important | A take whose outcome was unknown could still leave an orphan ticket and lead to a duplicate | shared |
| R5-2 | Important | "Tap to enable sound" turned sound off | shared |
| R5-3 | Minor | A lock-shortening request that timed out could land late and cut short the next save's lock | shared |
| R5-4 | Minor | A reset that timed out but landed could, on retry, wipe a ticket taken in the meantime | shared |
| R5-5 | Minor | Access errors on the live connection caused endless reconnect attempts | shared |
| R5-6 | Minor | On touch screens, the first tap didn't unlock sound | shared |
| P-4 | Polish | Waiting-list times kept the old clock estimate after the startup clock check | shared |
| P-5 | Polish | A brief rate limit on the live connection stopped an unattended kiosk for good | shared |
| P-6 | Polish | One save attempt could run far past the 25 s limit | shared |
| P-7 | Polish | The take warning said "Reload" while the page was already reconnecting | shared |

### R5-1. Orphan ticket when a take's outcome is unknown (important)

**Cause.** R4-1 stopped a retry within one tap from adding a second ticket. But if the save landed with its reply lost, and the retries then also failed, the tap ended with an error and its request id was thrown away. The ticket sat on the board without the "You" tag, the phone never gave the "your turn" alert for it, and tapping again created another ticket.

**Fix.**
- **Request ids are remembered on the device:** the ids this device has sent are saved in browser storage and survive a reload. On every live update, the page claims any ticket carrying one of them as this device's own, so the "You" tag and the alerts work.
- **The next tap reuses an unknown request:** a take whose outcome is unknown stays pending. The next tap reuses its id, so if the earlier save had landed, the tap returns that ticket instead of adding one.
- **A failed take checks the queue:** when a take reports failure, the page first checks whether its ticket is already in the queue. If it is, it shows "You're #002" instead of an error.

### R5-2. "Tap to enable sound" turned sound off (important)

**Cause.** After a reload, with sound saved as on, the footer button reads "Tap to enable sound". But tapping it ran the on/off toggle, which turned sound off and saved that. The person thought they had enabled alerts, and their turn passed silently. This came from round 2 and no later review caught it.

**Fix.** When sound is on but the page isn't yet allowed to play it, the button only unlocks audio. It toggles sound on and off only in its other states.

### R5-3. A late lock-shortening request could cut short the next lock

**Cause.** The fourth round put a 5-second limit on the request that shortens the lock. If that request didn't answer in time, the page moved on, but the request could still land later and shorten the lock the next save had just taken. That would bring back R3-1.

**Fix.** The page counts the shortening requests it stopped waiting for that may still land. While any are outstanding, a new lock is treated as lasting only 1 second, so a slow save backs off instead of risking an overwrite.

### R5-4. A retried reset could wipe a ticket taken since

**Cause.** If a reset landed but timed out, another phone could take #001 in the new queue before the retry ran. The retry then reset again and erased that ticket.

**Fix.** Each reset carries an id that is saved with the new queue. A retry that finds its own id already saved does nothing.

### R5-5. Access errors on the live connection caused endless reconnects

**Cause.** The docs list `not_granted`, `capability_disabled`, `capability_removed` and `transform_error` as errors the live connection can end with. The page treated them as a temporary outage and kept reconnecting, every 30 seconds, forever.

**Fix.**
- Access errors now make the page view-only with "The queue is no longer available here."
- `invalid_argument` and `transform_error` stop reconnecting and ask for a reload.

### R5-6. First tap on a touch screen didn't unlock sound

**Cause.** The page unlocked sound when a finger first touched the screen. Phones only allow audio once the touch ends (a click). So on phones that first touch did nothing, and the listener was then gone.

**Fix.**
- Sound now unlocks on click.
- The listener stays until audio is actually running, so a scroll doesn't use it up.
- It runs after the tapped button's own handler, so it can't swallow that tap.

### Polish

- **P-4, waiting-list times:** the startup clock check now refreshes everything, waiting-list times included. To keep that safe, any refresh that arrives while a finger or mouse button is down now waits until the tap ends before rebuilding buttons. This also protects taps from updates arriving from other devices, not just the clock check.
- **P-5, rate limit on the live connection:** the page now tries again after a full minute instead of stopping for good.
- **P-6, save attempts running past the limit:** each store call's time limit is now capped by what's left of the 25-second limit.
- **P-7, take warning:** it now says the page is reconnecting and to keep it open. It only says "Reload the page" when the page has stopped trying.

## Fifth review testing

Tested against a temporary stand-in for the claude.ai store, which can now apply a sequence of faults, including a save that lands without sending a live update.

| Test | Result |
|---|---|
| Take lands, its reply is lost, both retries fail, and a live update arrives | Showed "You're #002", with no error and no extra ticket |
| Same, but no live update arrives (outcome unknown) | Request kept as pending. The next tap reused it, showed "You're #003", and created no #004 |
| Orphan ticket from an earlier take | Claimed by the live update and shown with the "You" tag |
| Reset lands but reports an error | Reset once only, and the queue stayed as reset |
| Sound saved as on, then a tap on "Tap to enable sound" | Sound stayed on and saved as on, and the label became "Sound on" |
| Another device's update arrives while a button is pressed | The button survived, and the list caught up after the tap |
| Live connection ends with `capability_disabled` | Showed "The queue is no longer available here.", buttons disabled, no reconnect attempts |
| Script errors | None. The only console messages were the browser blocking vibration for scripted, non-human clicks, as expected. |

**Not tested:** the real claude.ai store and real phones, including touch audio rules and touch event timing. R5-3 was checked by reading the code only, because the stand-in can't deliver a request after the page has stopped waiting for it.

# Fourth review

The fourth review checked the third round's fixes. It confirmed that these work: R3-1 (the lock is shortened before the next save on every path), R3-3, R3-4, R3-5 and automatic reconnecting. It found that R3-7's retries introduced a new bug, and that R3-6 was only partly fixed.

| # | Severity | Finding | Fixed in |
|---|---|---|---|
| R4-1 | Important | One tap could create two tickets when a save landed but reported an error | shared |
| R4-2 | Minor | A successful take or reset still hid a dead live connection | shared |
| R4-3 | Minor | Reconnecting gave up after about 30 s, and only for one error code | shared |
| R4-4 | Minor | Errors that retrying can't fix still said "Check your connection" | shared |
| P-1 | Polish | The first tap on "Turn on alert sound" could still be lost | shared |
| P-2 | Polish | The startup clock check rebuilt the action buttons | shared |
| P-3 | Polish | A Skip at exactly 3:00 could be refused | shared |
| Extra | Not tested by the review | A store call that never answers would freeze all saving | shared |

### R4-1. One tap, two tickets (important)

**Cause.** The third round added retries for temporary errors. But a save can reach the store even though its reply is lost, and the store's docs warn that "a retried write can apply twice". The retry then read a queue that already had the new ticket and added another. The phone only knew about the second ticket, so when the first was called, it didn't give the "your turn" alert, and the orphan ticket held up the line as a no-show.

**Fix.** Each tap on "Get my number" gets a request id, which is stored on its ticket. Before adding a ticket, the save checks for that id. If it's already there, it returns the existing ticket and saves nothing. The other actions (start, end, skip, leave, reset) were already safe to repeat, because they check the ticket's state first.

### R4-2. A successful take or reset hid a dead live connection

**Cause.** After a take or reset, the page always showed its success message, which replaced "Lost the connection" even though the board was no longer updating.

**Fix.** While the live connection is down, a take now says: "You're #007, but this page isn't receiving updates, so it can't alert you. Reload the page." A reset keeps the connection error.

### R4-3. Reconnecting gave up too soon

**Cause.** The page stopped reconnecting after 5 tries, about 30 seconds, so an unattended kiosk stayed frozen after a longer outage. It also only reconnected for `unavailable`, although the docs say to treat unknown codes the same way.

**Fix.**
- The page keeps reconnecting while it's open, waiting longer each time, up to 30 seconds between tries.
- It reconnects for any error except `revoked`, `invalid_argument` and `resource_exhausted`.

### R4-4. Errors that retrying can't fix said "Check your connection"

**Fix.**
- `revoked`, along with the access codes `not_granted`, `capability_disabled` and `capability_removed`, now shows "The queue is no longer available here." and stops offering actions.
- `resource_exhausted` now shows "Too many requests right now. Wait a moment and try again."
- When the live connection is revoked, the page also stops offering actions.

### Polish

- **P-1, first tap on "Turn on alert sound":** the sound refresh after the first touch now waits until that tap's click has been handled. With no click, such as a scroll, it runs half a second after the finger lifts.
- **P-2, startup clock check:** it now refreshes only the timers, not the action buttons.
- **P-3, Skip at exactly 3:00:** the Skip re-check allows 1 second of slack, because the button may have turned on using a slightly different clock estimate.

### Extra: time limit on store calls

The fourth review couldn't check whether a store call can hang. If one did, every later save from that page would wait for it forever, and the page would be stuck on "Getting your number…". Each store call now has a 15-second limit (5 seconds for shortening the lock). A call that times out is treated like a dropped request and retried. Because of R4-1, a retried take still can't create a second ticket.

## Fourth review testing

Tested against a temporary stand-in for the claude.ai store that can make a save land and then report an error, make a call never answer, or return specific error codes.

| Test | Result |
|---|---|
| Save lands but reports an error | One ticket (#002), 2 saves in total, and the phone knows #002 is its own |
| Lock request returns `resource_exhausted` | Showed "Too many requests right now…" |
| Live connection ends with an unknown code | Showed "Reconnecting…", then "Live" |
| Take while the live connection is down | Showed the "isn't receiving updates" warning in red |
| A read that never answers | Timed out after 15 s, the retry saved, and the button worked again (about 15.7 s in total) |
| Lock request returns `revoked` | Showed "The queue is no longer available here.", and buttons were disabled |
| Script errors | None |

**Not tested:** the real claude.ai store, and the order of touch events on real phones.

# Third review

The third review checked the second round's fixes. It confirmed these work: the one-at-a-time save chain can't stall, the server-clock math is right, server time is used consistently, the take button always comes back, two-tap buttons stay ready, ending a session retries, and the switch to view-only works. It found 1 important and 6 minor issues.

| # | Severity | Finding | Fixed in |
|---|---|---|---|
| R3-1 | Important | Shortening the lock after a save could cut short the next save's lock | shared |
| R3-2 | Minor | The first tap after each page load did nothing | shared |
| R3-3 | Minor | The server clock was only learned after a save, and Skip could still fire early | shared |
| R3-4 | Minor | A tab could still stay silent after another tab reset the queue | index.html |
| R3-5 | Minor | On a slow connection, the page held the lock for 25 s without saving | shared |
| R3-6 | Minor | A successful save could hide a dead live connection | shared |
| R3-7 | Minor | One dropped request failed the whole action and left the lock held | shared |

### R3-1. Lock shortening could cut short the next save's lock (important)

**Cause.** After each save, the page sent a request to shorten its lock to 1 second without waiting for it, then started the next queued save straight away. If the store handled the next save's 5-second lock request first and the shortening request second, the new lock ended after about 1 second while the page believed it had 5. On a slow connection, another device could then take the lock and save over this one, losing a ticket or reusing a number.

**Fix.** The page now waits for the shortening to finish before the next save starts. This applies after a successful save, after a save that turned out to need no change, after a too-slow attempt and after an error.

### R3-2. First tap after a page load did nothing

**Cause.** The first touch on a page unlocks sound, and that redrew the whole page, including the action buttons, before the finger lifted. The tap then landed on a button that no longer existed. For example, "I'm here" needed two taps after a reload.

**Fix.**
- Unlocking sound now refreshes only the sound button and the "Your number" card. The action buttons and waiting list are left alone.
- The first-touch refresh waits until the tap has finished.

### R3-3. Server clock only learned after a save, and Skip could fire early

**Cause.** A freshly loaded device used its own clock until it first saved something. So a kiosk with a fast clock could allow Skip seconds after a call, and a lone kiosk with a slow clock let sessions overrun again. The Skip action also didn't re-check the 3-minute wait.

**Fix.**
- On load, the page asks for a short lock on a separate `queue/clock` record. That gives it the server time straight away without ever holding up the queue.
- Skip now re-checks the 3-minute wait when it saves, using a freshly measured offset. If the wait isn't over, it refuses and says why.

### R3-4. A tab could stay silent after a reset in another tab

**Cause.** Each tab remembered the last number it announced. The tab that pressed Reset cleared its memory, but other tabs didn't, so their next #001 was skipped as "already announced".

**Fix.** That memory is removed. Since the second round, only the tab that calls a number announces it, once, so the memory no longer served a purpose.

### R3-5. Slow connection held the lock for 25 s without saving

**Cause.** When a save was too slow to land safely, the page immediately asked for the lock again, renewing it over and over for the full 25 seconds without saving and blocking every other device. The cutoff was also measured from when the lock was requested rather than when it was granted, which was stricter than needed.

**Fix.**
- The cutoff is now measured from the estimated time the lock was granted.
- After a too-slow attempt, the page releases the lock and waits 0.5–1.5 s.
- It gives up after 2 too-slow attempts and says the connection is too slow.

### R3-6. A successful save could hide a dead live connection

**Cause.** Clearing an earlier error after a successful save also cleared "Lost the connection to the queue", replacing it with "Live" while the board was no longer updating.

**Fix.**
- The page tracks whether the live connection is down, and successful saves don't clear that message.
- When the connection drops, the page reconnects automatically (up to 5 times, waiting longer each time), as the store's docs recommend. It shows "Live" again once updates arrive.

### R3-7. One dropped request failed the whole action

**Cause.** Any error during a save ended the action with "Could not reach the queue", even a brief outage the store's docs say to retry. The lock also wasn't released after an error.

**Fix.**
- Temporary errors are retried up to twice after a short random pause. Errors that retrying can't fix, such as access or quota errors, still stop the action at once.
- The lock is released after any error once it was held.

## Third review testing

Tested in a local browser against a temporary stand-in for the claude.ai store. The stand-in could add network delay, run the server clock 3 minutes ahead, drop one request or stop the live connection.

| Test | Result |
|---|---|
| Server clock 3 min ahead | The page's clock showed server time from load, before any save |
| One dropped request while taking a number | Retried and succeeded, with no error shown |
| Skip forced right after a call | Refused with "A no-show can be skipped 3 minutes after the number is called" |
| Lock after a save | Released to about 1 s instead of held for 5 s |
| First tap on "I'm here" after a page load | The button survived the first touch, and the timer started on that tap |
| Live connection stops | Showed "Reconnecting…", then "Live", and the board updated again |
| 1.2 s round trips | Take succeeded in about 3 s |
| 2.4 s round trips | Take succeeded in one attempt |
| 3.2 s round trips | Two attempts, the lock released each time, then "connection is too slow", and the button worked again |
| Single-screen version: a reset in another tab, then a call | #001 announced once |
| Script errors | None |

**Not tested:** the real claude.ai store. The stand-in's lock follows the store's documentation, but these were not confirmed on the real store: how it orders two lock requests from the same page, whether a renewal can shorten a lock, and how precise its lock expiry times are.

# Second review

## Summary

| # | Severity | Finding | Fixed in |
|---|---|---|---|
| 1 | Important | Sessions run 2 minutes over when the phone that started them sleeps or reloads | shared |
| 2 | Important | Two writes from one page can overlap, causing double tickets or a lost change | shared |
| 3 | Important | The "it will chime" promise isn't kept after a page reload | shared |
| 4 | Important | Taking a number fails on slow connections or when several people tap at once | shared |
| 5 | Important | View-only people are told it's a connection problem, and their page keeps retrying | shared |
| 6 | Minor | One tap by anyone ends someone else's session | shared |
| 7 | Minor | Wrong phone clocks show garbled timers and allow an early Skip | shared |
| 8 | Minor | The second tap on Leave, Remove or Skip is lost in a busy queue | shared |
| 9 | Minor | A red error stays in the status line after later actions succeed | shared |
| 10 | Minor | Two open tabs announce twice, and one can stay silent after a reset | index.html |

## Important findings

### 1. Sessions run 2 minutes over

**What went wrong.** When a session hit 0:00, the board could keep showing "0:00 left" for 2 more minutes before calling the next person.

**Cause.** The first review's fix for phone clocks let only the page load that tapped "I'm here" end the session at 0:00. Every other device waited a 2-minute grace period. That page load is identified by a random id created each time the page loads. So if the person's phone locked, slept or reloaded the page, nobody ended the session on time. That's the normal situation for someone sitting in the chair.

**Scenario.** Alex taps "I'm here" on his phone, locks it and sits down. His phone's browser pauses the page in the background, so it never ends the session at 0:00. The kiosk and other phones wait 2 more minutes.

**Fix.** Every device now uses the server's clock instead of its own. When a device asks for the save lock, the server replies with the time the lock will expire. From that, the page works out the difference between its own clock and the server's. Session start times, call times and end times are saved on the server's clock. Any device can now end a session 3 seconds after 0:00, and the per-page-load ownership rule is removed.

### 2. Overlapping writes from one page

**What went wrong.** One tap could create two tickets. Or a session ending at the same moment someone took a number could erase that number, and the next person would be given the same number.

**Cause.** Two problems combined:
- Every save from a page used the same lock-holder id. The server treats a second lock request from the same holder as a renewal and grants it, so the lock didn't keep one page's saves apart.
- Every live update from another device re-enabled "Get my number" while the first request was still saving.

**Scenario A.** The queue is busy, so the first save is retrying. Another phone's change arrives and re-enables the button. The user taps again and gets two tickets, or both saves read the same state and two people end up with the same number.

**Scenario B.** On the kiosk, the session ends at 0:00 at the same moment someone taps "Get my number" on that kiosk. Both saves read the same queue and the last one wins. Ticket #5 disappears, and the next person is also given #5.

**Fix.**
- All saves from one page now run one at a time, in order.
- "Get my number" stays disabled until its save finishes, even while updates from other devices arrive.

### 3. No chime after a reload

**What went wrong.** The "Your number" card said "it will chime when it's your turn", but after the page reloaded, no chime played.

**Cause.** The sound setting wasn't saved, so it was off again after a reload. Browsers also only allow sound after a tap on each page load. Phones often reload background tabs, so the card's promise was usually wrong.

**Fix.**
- The sound choice is now saved on the device.
- The card only says "it will chime" when sound is on and the browser is ready to play it. Otherwise it shows a **Turn on alert sound** button.
- The footer button reads "Tap to enable sound" when sound is on but still needs a tap.

### 4. Taking a number fails on slow connections or at busy moments

**What went wrong.** On a weak signal, no save could ever succeed. After about 25 seconds the page said "The queue is busy". At a busy moment, the fourth of four people tapping together also got "busy".

**Cause.**
- A save gave up if more than 2 seconds passed between asking for the lock and saving. With about 1 second of network delay, every attempt took longer than that.
- Locks can't be released, so each save blocked the queue for the full 3 seconds. The 12 retries only covered about 6–8 seconds.

**Fix.**
- The lock now lasts 5 seconds. The page measures the network delay and only gives up if the save might arrive less than half a second before the lock expires. A connection with 1 second of delay now succeeds.
- After saving, the page shortens its lock to 1 second so other devices don't have to wait the full 5 seconds. The lock system doesn't document this shortening, so if the server ignores it, the lock simply lasts the full 5 seconds.
- When another device holds the lock, the page waits until that lock expires instead of guessing.
- Retries now continue for up to 25 seconds instead of stopping after 12 attempts.
- If saves failed because the connection was too slow, the page now says so ("Your connection is too slow…") instead of "The queue is busy".

### 5. View-only people told it's a connection problem

**What went wrong.** For people who can only view the queue, such as people outside your organization, every button stayed enabled and every tap said "Could not reach the queue". After a session ended, their page retried once a second.

**Cause.** The page couldn't always tell in advance whether someone may make changes, so it let them try. The fallback to view-only only ran when the final save failed, but these people's first step, asking for the lock, failed before that.

**Fix.** Both the lock request and the save now share one error handler. When access is the problem, the page switches to view-only at the first refused request. Ending a session at 0:00 also waits 5 seconds after a failed attempt instead of retrying every second.

## Minor findings

### 6. One tap ends someone else's session

**What went wrong.** "End session early" ended a session after a single tap, on any device that can make changes.

**Fix.** It now needs a second tap within 4 seconds, like Skip, Leave and Remove.

### 7. Garbled timers and early Skip from wrong phone clocks

**What went wrong.** A phone whose clock was a few seconds slow showed "Waiting for arrival · -1:-5". A phone whose clock was fast showed "30:05 left" and a progress bar outside its range. It could also allow Skip right after someone was called.

**Fix.**
- All timers, the Skip wait and ending sessions now use the server's clock (see finding 1).
- Times can no longer go below 0:00.
- The progress bar stays between empty and full.

### 8. Second tap lost in a busy queue

**What went wrong.** A user tapped "Leave", another phone took a number, and the list rebuilt. The second tap then only asked for confirmation again.

**Fix.** The page now remembers which button is waiting for its second tap, separately from the buttons themselves. When the list rebuilds, that button stays ready.

### 9. Red error stays in the status line

**What went wrong.** After one failed tap, "The queue is busy…" stayed in red even after later actions succeeded.

**Fix.** Any successful save now replaces an error with the normal "Live" message.

### 10. Single-screen version: double alerts and a silent tab after reset

**What went wrong.** With two tabs open, every call chimed and was read aloud twice. After a reset, one tab could stay silent when #001 was called again.

**Cause.** Each tab announced every call it saw and kept its own record of the last number it announced.

**Fix.** Only the tab whose action called the number announces it. When a session ends at 0:00, only the tab that ends it announces the next number, because the other tab finds the session already ended.

## Known limitation

Reset and Remove are only shown to Editors, but this isn't enforced by the queue's storage. The whole queue is stored as a single record, so its access rules can't allow one change and refuse another within it. Someone with Contributor access and browser developer tools could still reset the queue. Enforcing this would mean splitting the queue into separate records with their own access rules.

## First review (earlier, already fixed)

The first review found these issues, which were fixed in commit `e1eced5`:

1. **Duplicate or lost numbers:** a lock could lapse during a slow save, so a write could overwrite another device's.
2. **Stuck at 0:00:** a failed attempt to end a session was never retried.
3. **Greyed-out buttons:** buttons stayed disabled after a failed action.
4. **Early finish from clock differences:** a phone with a fast clock could end sessions early. Finding 1 above replaced this fix.
5. **Anyone could reset:** any Contributor could reset the queue or remove other people's tickets.
6. **Two tabs:** in the single-screen version, two tabs could overwrite each other's changes.
7. **Misleading error:** every `invalid_argument` error was treated as missing access.
8. **Sound turned back on:** taking a number turned sound back on after the user switched it off.

## Testing

The shared page was tested in a local browser against a temporary stand-in for the claude.ai store. The stand-in had adjustable network delay and the same lock behavior: a lock request from the holder that already has it succeeds as a renewal.

| Test | Result |
|---|---|
| Double tap on "Get my number" | One ticket created |
| Another device's update arrives while a take is saving | Button stayed disabled, and both tickets were kept |
| Slow connection (0.8 s round trips) | Take succeeded in about 2 s, with no "busy" error |
| Session ends at 0:00 at the same moment as a take | Session ended, the new ticket was kept and called, and no number was reused |
| Call time stamped 5 s in the future by another device | Timer showed 0:00, not a negative time |
| Single-screen version, ticket added by "another tab" | Kept, and the next number continued correctly |
| Script errors in either page | None |

**Not tested:** the real claude.ai store on several phones at once, view-only access, and how the sound prompt behaves after a real phone reloads the page in the background. Check these with two devices before relying on them.
