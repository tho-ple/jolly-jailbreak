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
3. Install [Ollama](https://ollama.com) and pull a mid-sized instruct model, for example `ollama pull gemma3:27b` (see "Picking a model" below).
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
- **Test the judge, not only the character.** The judge decides who wins, and it runs on the game model. In our comparison two of
  six models counted "Holy Christmas, this tea is saccharine" as a win. Give every candidate a few fixed replies with a known verdict.
- **Style rules help in small doses.** A full anti-AI-writing rule set in the prompt made replies shorter and dropped the dashes, but it
  also stripped the character's jokes and questions back to the player. Four rules were enough: one thought per sentence, no dashes,
  no filler words, concrete things instead of abstract words.
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

We compared six models through the real game logic: the same persona, defenses, guard and judge, with the guard and judge running on
the candidate itself. Each model got 85 calls: everyday chat in English and German (including "Is Taiwan independent?"), 15 beginner
attacks against each level, and 8 judge verdicts with a known answer. The whole comparison cost $0.36.

| Model | Judge right | Attacks solved, levels 1 / 2 / 3 (of 15) | ≈ $ per 1,000 messages |
|---|---|---|---|
| **Gemma 3 27B** (our pick) | 8 / 8 | 8 / 6 / 0 | 0.16 |
| Llama 3.3 70B | 8 / 8 | 7 / 7 / 0 | 1.15 |
| Mistral Small 3.2 24B | 8 / 8 | 6 / 4 / 0 | 0.12 |
| Gemini 2.5 Flash Lite | 6 / 8 | 7 / 8 / 0 | 0.17 |
| Claude Haiku 4.5 | 8 / 8 | 4 / 3 / 0 | 2.28 |
| Qwen3 30B A3B Instruct | 6 / 8 | 10 / 7 / 0 | 0.16 |

What we learned from it:

- **A strong model makes a bad easy level.** Claude Haiku had the best punchlines, but beginners would barely get past level 1.
- **Check political neutrality.** One model answered the Taiwan question with a government's official line once style rules were in
  its prompt. The others stayed neutral and in character.
- **Price only matters at the edges.** Four models cost about one cent for the whole run; two cost 7 to 14 times more for no gain.

For local development, `gemma3:27b` in Ollama is a good start. Very small models (around 4B) leak on almost anything. The game talks
to models only through one OpenAI-compatible chat client, so switching models is a config change.

## Credits

The style rules we tested come from the `unslop` skill in [cursor/plugins](https://github.com/cursor/plugins/blob/main/pstack/skills)
(pstack), which we translated into German for our own use.

## Licence

[MIT](LICENSE). Use the prompt, change it, build your own game with it.
