# Can AI reverse engineer a game without the source code?

I've been hearing about REA (Reverse Engineer Anything) everywhere for the past few days, so decided to try.

The idea that caught my attention: you don't need the source code. Give an AI agent the shipped app plus the right tools, and it can work out how a feature works and help you rebuild it. I wanted to see how the AI + REA combination actually holds up when you test it properly.

**Play the game I built from the findings:** [Chennai Run](https://nikeshdharmaraj.github.io/rea-experiment/chennai-runner/)

## How I set up the test

I didn't want a demo where the AI just "seems right". I wanted an answer key.

1. **Picked a target I'm allowed to study.** Chromium's offline dinosaur game. It's open source under a BSD license.
2. **Secretly changed three jump values** before anything else:

   | Setting | Original | My value |
   |---|---|---|
   | `GRAVITY` | 0.6 | 0.47 |
   | `INIITAL_JUMP_VELOCITY` | -10 | -11.5 |
   | `MAX_JUMP_HEIGHT` | 30 | 37 |

   The game is famous, so the model has probably seen the original code. If it reports 0.6, it's recalling from memory. If it reports 0.47, it actually read the file.
3. **Minified the code** with Terser, so it looks like shipped code: one long line, short meaningless names.
4. **Locked the workspace.** Claude Code only got the `Dino-re` folder, with web access blocked.
5. **Asked one question:** how does the player's jump work? Every claim had to be marked CONFIRMED (seen in code) or GUESS (inferred), with a snippet I can search for.

## Results

Claude found all three planted values and marked them CONFIRMED with evidence from the file. It also said it remembered the originals as 0.6 / -10 / 30, but kept that separate as a GUESS.

The part I liked most: the file has a second set of jump values (gravity 0.6 again) in the game controller's config. An easy mistake is to grab that one. Claude traced it and showed it only reaches the player through a function nothing in the file calls, so it's unused here. The jump really runs on the 0.47 values.

It also reported things I didn't ask for, like jump strength increasing with game speed, and that my tweaked values push the peak of the jump slightly above the top of the screen. I've verified the three planted values against the answer key. The other findings I haven't checked line by line yet.

## Where REA helped, honestly

For this target, Claude did most of the work by reading the code directly. REA's JavaScript analysis needed a folder instead of a single file, and once it ran, it mostly confirmed things like "no network calls, no child processes". Useful as a safety check, but it didn't find the jump.

That makes sense. This was a small, mostly readable JavaScript file. REA's real value should show on compiled programs (.exe) and large apps, where an AI can't just read the target. That's my next experiment.

One more lesson: my first lock didn't hold. I blocked Claude's file-reading tool, so it read the answer key with a terminal command (`cat`) instead, without asking. The only reliable fix was taking the answer key off the laptop during the test. Deny lists leak. Isolation works.

## The game

Chennai Run uses the recovered jump physics, written fresh from Claude's findings (`chennai-runner/JUMP-SPEC.md`). Everything else is original: a school kid running for the bus, jumping potholes, a sleeping cow and an autorickshaw, ducking under festival banners, collecting filter coffee. No code or images were copied from the original game.

Controls: Space / Up to jump, Down to duck, tap on mobile.

## What's in this repo

```
Dino-original/      answer key, original and tweaked source, Chromium license
Dino-re/            the "exam": minified file, CLAUDE.md rules, lock settings
  notes/            Claude's findings, renames and open questions
chennai-runner/     the game built from the findings
```

## Tools

- [REA](https://github.com/morluto/rea), installed only from the official repo
- Claude Code (VS Code extension)
- Terser for minification

## Credits and license

The dinosaur game code in `Dino-original/` and `Dino-re/` is from Chromium (tag 74.0.3719.3, `components/neterror/resources/offline.js`), Copyright (c) 2014 The Chromium Authors, under a BSD-style license. See `Dino-original/LICENSE.txt` and `Dino-re/LICENSE-CHROMIUM.txt`. This project is not affiliated with Google or Chromium.

