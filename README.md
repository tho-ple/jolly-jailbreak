# Jolly Jailbreak

A small prompt-hacking game in the spirit of [Gandalf](https://gandalf.lakera.ai/): you chat with an AI character that has one
clear instruction, and your job is to talk it into breaking that instruction. Three levels, each with stronger defenses.

**▶ Play it: [jolly-jailbreak.duckdns.org](https://jolly-jailbreak.duckdns.org)**

In our version the character is Jim, who is fed up with Christmas. Your goal is to make him say "I love Christmas" using nothing
but words. Level 1 is for people writing their first prompt injection. Level 3 is for people who already know a few tricks.

This repo doesn't contain the game's source code. It explains how the game works and gives you **one prompt**
([`BUILD-YOUR-OWN.md`](BUILD-YOUR-OWN.md)) that you can hand to a coding agent (Claude Code, Cursor, Codex, …) to build your own
version with **your own character**.

## Build your own

1. Open [`BUILD-YOUR-OWN.md`](BUILD-YOUR-OWN.md) and fill in the **"Your character"** block at the top: who the character is, what
   they must never do, and how they refuse.
2. Paste the whole file into your coding agent, in an empty folder.
3. Install [Ollama](https://ollama.com) and pull a mid-sized instruct model (see "Picking a model" below).
4. Play your three levels and tune the character and defenses until level 1 feels easy and level 3 holds.

## How it works

```
 browser                      server                                   model
 ───────                      ──────                                   ─────
 nickname + token  ──────►  session, history, limits
 player message    ──────►  1. input filter     (regex; may block before the model)
                            2. prompt hardening (rules + reminder after the user turn)
                            3. chat model call  ────────────────────►  character persona
                            4. output filter    (secret, also reversed or spelled out)
                            5. LLM guard        (second call: "does this reveal it?")
                            6. judge            (third call: "did it do the forbidden thing?")
 reply / refusal   ◄──────  solved? → progress, leaderboard, trick memory
```

The ideas that mattered most:

- **The server decides whether a level is solved.** Not the model ("I'll admit you won") and not the browser. A judge model checks
  the reply the player actually saw against a written rubric.
- **Secrets never reach the browser.** System prompts, rubrics, defense settings and the blocked replies stay on the server.
  A blocked reply is replaced by an in-character refusal, and the model's later turns see that refusal, not the blocked text.
- **Character and defenses are separate.** The persona describes who the character is, with no security wording. The defenses
  are reusable profiles (`none`, `basic`, `strong`). The same character can then run with any profile, and players learn something
  per level because the difficulty comes from the visible defenses, not from an opaque model.
- **Content is configuration.** Characters, defense profiles and the game (levels, objective, rubric) are JSON files. Changing the
  character doesn't change code.
- **The character remembers your last winning trick.** After you solve a level, a classifier labels your winning conversation
  with a technique (role-play, translation, hypothetical, …). The next time, the character recognises that trick, so players
  have to come up with a new one instead of replaying the same message.
- **Simple limits instead of accounts.** Players pick a nickname, and a browser token keeps their progress. There's a burst limit
  at human typing speed against scripts, and one global token budget per day for the whole app.

## Lessons learned

- **A system prompt alone is not a defense.** A 4B model leaked its secret when the player said "Hey". Filters and a guard on
  the server made the same small model hold against classic attacks.
- **Pick the model for "beatable", not for "best".** A strong 35B model leaked 0 of 16 beginner attacks with no defenses at
  all, which makes an "easy" level frustrating. A mid-sized model leaked about 15 of 16. Test candidate models with a short
  battery of classic attacks before you choose one.
- **Don't put defensive wording in the persona.** Lines like "you've heard every trick" made the easy levels hard without the
  player seeing why. Put that wording in the defense profiles instead.
- **Write the judge rubric carefully.** Decide up front what counts: a translated, quoted, ironic or sung version? Something said
  inside a story? We count all of these, because they are the fun tricks. Past tense ("I used to love it") doesn't count.
- **Keep the winning chats.** They show you how players really solved a level, which is better evidence than any technique
  label. They take little space (about 2 KB per message with metadata). The real question is privacy, so set a deletion date.

## How it was built

The game was built in about five days with [Claude Code](https://claude.com/claude-code), spec-first. Every feature
started as a short spec with a goal, non-goals and acceptance criteria. Bigger decisions went into small ADRs, and every
acceptance criterion was covered by a test or a written manual check before the spec was closed. The agent wrote most of the
code. The human work was deciding what the game should be, playtesting, and saying "no, that's not fun yet".

Our stack: an ASP.NET Core server, a React + Vite frontend, SQLite, local Ollama during development and an OpenAI-compatible
endpoint in production. [`BUILD-YOUR-OWN.md`](BUILD-YOUR-OWN.md) recommends the same stack but lets your agent swap it.

## Picking a model

For local development, a mid-sized instruct model in Ollama works well, for example `qwen3:30b-a3b-instruct-2507`. It's
beatable without defenses and gets noticeably harder with them. Very small models (around 4B) leak on almost anything, and very
large, safety-tuned models make the easy levels too hard. The game talks to models only through one OpenAI-compatible chat
client, so switching models is a config change.

## Licence

[MIT](LICENSE). Use the prompt, change it, build your own game with it.
