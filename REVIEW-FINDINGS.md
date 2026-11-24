# Code review findings and fixes

This file records the second code review of the Massage Chair Queue, which a review subagent ran on 24 November 2026, and how each finding was fixed. It also lists the findings from the first review for reference.

The second review found 5 important and 5 minor issues. Findings 1–9 are in the shared phone version, [shared/massage-queue.html](shared/massage-queue.html). Finding 10 is in the single-screen version, [index.html](index.html).

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
