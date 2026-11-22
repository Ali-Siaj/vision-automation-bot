# Vision and OCR

*How the bot sees the screen. No image templates or source code are published.*

## 1. One coordinate system

The game can run on any monitor, so the bot never works in raw screen pixels.

- Every capture is resized into a fixed **1920×1080 "game pixel" space**.
- All regions to read and all click targets are defined in that space.
- A conversion step turns a game pixel into the right real screen pixel, including the monitor's own position, so multi-monitor setups work.

This means a region that is correct on one PC is correct on all of them.

## 2. Finding things: template matching, used carefully

The workhorse is OpenCV's normalised template matching. A raw "best match" is not trusted on its own:

- **A threshold per kind of target, tuned on real screenshots.** For example, the Connection-lost dialog scores about 0.9 where it is and under 0.35 on every other screen I tried, so it is accepted at 0.65.
- **Ambiguity guard.** To choose a row in the Build menu, the bot compares candidate pictures. A row is clicked only if its picture matches strongly *and no other level's picture matches the same row nearly as well*. Two rows that look alike are never guessed between; the bot refuses and logs why.
- **Several hits, not just the best.** When a screen has several identical controls (for example two upgrade buttons side by side), matches that overlap a better one are suppressed so each button is returned once.
- **Context for the "which resource?" question.** An upgrade button is classified as Gold or Elixir by matching the coin or drop icon beside its price, and only accepted if the winner clearly beats the other.
- **Missing pictures fail loudly.** If a required template is absent, the bot reports it at start instead of failing mysteriously mid-run, and a "Check Images" button audits them all.

### Scale-tolerant detection for pop-ups

Some dialogs are drawn by the operating system, so their size depends on the device. For those, the bot searches **0.5× to 1.6×** the reference picture in a coarse pass, then refines around the best scale in a second, finer pass. The click target is computed from the *matched* scale, so it lands on the button whatever size the dialog is.

I tested this against a real screenshot and against the same dialog pasted at 0.6×, 0.9×, 1.3× and 1.6×, and against screens that must *not* match, which score far below the threshold.

## 3. Reading numbers: two OCR pipelines

Tesseract is not good at game fonts out of the box, so each number has its own pre-processing.

### Damage percentage (during a battle)

- Isolate the digits with a **colour mask** (HSV range) instead of a plain threshold, so busy backgrounds do not matter.
- **Drop any blob touching the crop's edge.** The crop ends just before the "%" sign, and without this the sliver of the "%" was read as an extra digit.
- Thicken, **upscale 6×**, binarise, and read as a single raw line with a digit whitelist.
- This was **tuned against screenshots with known values** (0, 3, 29, 39 and 57 percent), because the font made Tesseract confuse 9, 3 and 5.
- Anything over 100 is rejected outright, and an impossible reading can never trigger the end-of-attack sequence.
- The reader sleeps when no battle is on, so Tesseract is not started needlessly.

### Gold and Elixir amounts (in the shop)

- Grayscale, threshold, **upscale 4×**, add a border, read as a single line with a digits-only whitelist.
- **Sanity layer on top:** a blank result is retried once, and a value above the storage capacity is capped to "full" rather than believed.
- **Loot learning.** From consecutive readings the bot estimates the income per attack, but only accepts values inside believable limits, so a misread cannot poison the estimate. It uses the estimate to schedule an *early* check when a resource should have reached the next upgrade price.

## 4. What I learned from real logs

Running the bot for hours produced evidence no unit test would have:

- Occasional misreads (for example an extra trailing digit) are a recurring source of wrong decisions, which is why the checks live in several layers rather than one.
- Log lines such as `Ignoring impossible damage reading: 293` show the guard doing its job in a real session.
