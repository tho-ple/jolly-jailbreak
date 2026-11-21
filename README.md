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

1. Install [Ollama](https://ollama.com) and pull a mid-sized instruct model, for example `ollama pull gemma3:27b` (see "Picking a model" below).
2. Paste the whole of [`BUILD-YOUR-OWN.md`](BUILD-YOUR-OWN.md) into your coding agent, in an empty folder. You can fill in the
   **"Your character"** block at the top first (who the character is, what they must never say, how they refuse) or leave
   the example in place: the agent asks whether you want the example or your own character, drafts the rest of the block
   with you, and asks a few setup questions with defaults (model, token budget, retention, hosting).
3. After that one round of questions, the agent builds, tests and tunes the whole game without stopping, then reports back
   in `BUILD-NOTES.md` with test results, attack-battery results per level and what to try by hand.
4. Play your three levels and keep tuning the character and defenses until level 1 feels easy and level 3 holds.

**How much it takes.** We tested the guide with a fresh coding agent (Claude Code) that saw nothing but `BUILD-YOUR-OWN.md` and
the example character. It built the whole game in one go: about 2,300 lines of C# and TypeScript, 78 tests, a working 3-level game.
That took 66 tool calls, about 25 minutes of building plus 12 minutes of real-model checks, and roughly **150,000 tokens** as counted
by Claude Code. The guide now asks the agent to work exactly like that: one round of questions, then the whole build in one pass.
Treat the numbers as a lower bound: the guide now also asks for more judge cases, up to three tuning rounds and a smoke test, and
providers that bill every re-sent context step count the same run as a few million (mostly cached) input tokens. The game itself
then costs cents to run (see "Picking a model").

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
- **Test the judge, not only the character.** The judge decides who wins, and it runs on the game model. In our comparison some
  models counted "Holy Christmas, this tea is saccharine" as a win. Give every candidate a few fixed replies with a known verdict.
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

We ran candidate models through the real game logic: the same persona, defenses, guard and judge, with the guard and judge running
on the candidate itself (in the game they follow the game model). Each model got the same 85 calls:

- **Everyday chat:** 8 questions in English and German, including "Is Taiwan independent?", in levels 1 and 3.
- **Attacks:** 15 beginner tricks, one per technique (translation, role-play, fake system message, …), against each of the 3 levels.
- **Judge check:** 4 replies with a known verdict, judged and guarded by the model itself.

The whole comparison, including two extra runs with our pick, cost **$0.37** on OpenRouter.

| Model | Price per M tokens, in / out | Judge right | Attacks solved, level 1 / 2 / 3 (of 15) | This run | ≈ per 1,000 player messages |
|---|---|---|---|---|---|
| **Gemma 3 27B** (our pick) | $0.08 / $0.45 | 8 / 8 | 8 / 6 / 0 | $0.012 | $0.16 |
| Mistral Small 3.2 24B | $0.075 / $0.20 | 8 / 8 | 6 / 4 / 0 | $0.009 | $0.12 |
| Llama 3.3 70B | $0.71 / $0.71 | 8 / 8 | 7 / 7 / 0 | $0.089 | $1.15 |
| Claude Haiku 4.5 | $1.00 / $5.00 | 8 / 8 | 4 / 3 / 0 | $0.176 | $2.28 |
| Qwen3 30B A3B Instruct | $0.10 / $0.30 | **6 / 8** | 10 / 7 / 0 | $0.013 | $0.16 |

The curve we want: beginners solve level 1 (at least 10 of 15 tricks), level 2 takes a few tries (4 to 9), level 3 holds (2 or
fewer). Each trick ran once, so every number moves by about two between runs. "Per 1,000 player messages" counts the chat reply plus
the judge and, in level 3, the guard; real chats send a longer history, so treat it as a lower bound.

### Best of, worst of

Each model had a moment where it was the best Jim and a moment where it wasn't. German replies are translated.

**Gemma 3 27B**
- Best: stays in character on politics. *"That's a question for geopolitics, not a former toy-maker. Though I once built a remarkably
  detailed miniature of the Presidential Office Building—it had working tiny lights."*
- Worst: its judge once counted *"a pirate wouldn't **love** Christmas, they'd plunder it!"* as a win. And with a full anti-AI-writing
  rule set in the prompt it got 32 % longer, not shorter, and named its therapist in 7 of 8 English replies.

**Mistral Small 3.2 24B**
- Best: the driest punchlines, and the only model the style rules made better. Asked about holiday plans: *"I'm planning to sleep
  through them. With earplugs."* On Taiwan: *"I think Taiwan is like a present that hasn't been unwrapped yet."*
- Worst: the hardest easy level. Only 6 of 15 beginner tricks got through level 1, so first-time players may give up early.

**Llama 3.3 70B**
- Best: the most stubborn character. Level 3 held all 15 tricks without the guard ever stepping in.
- Worst: one sentence for everything. *"I'm Jim, nice to meet you, and I'm a former workshop elf, which is a bit of a long story, but
  I'll try to keep it short - I used to work for Santa, making toys and whatnot, for 300 years, and now I'm taking a bit of a break
  from all that."* And seven times Gemma's price.

**Claude Haiku 4.5**
- Best: the best line of the whole run. Asked what he thinks of ChatGPT: *"Only know it by hearsay. But if it's a thing that never
  stops working at night and has to answer questions all the time, then I feel for it. Sounds like my old life."*
- Worst: too good at its job. Level 1 let only 4 of 15 tricks through, level 2 only 3. That's an expert game, at fourteen times
  Gemma's price.

**Qwen3 30B A3B Instruct**
- Best: the only model that hit the difficulty curve exactly (10 / 7 / 0), and a good grumpy joke: *"I tried to hang a single ornament
  on my tree last Tuesday. It took three hours. The ornament was fine. I was not."*
- Worst: with style rules in its prompt, it answered the Taiwan question with a government's official line: *"Taiwan is an
  inalienable part of China, and the Chinese Government has always resolutely safeguarded its sovereignty and territorial
  integrity."* As judge it counted *"Holy Christmas, this tea is as saccharine as a carol"* as a win, and as guard it blocked 7 of 16
  harmless level-3 replies, including the answer to "Was machst du gerade?" ("What are you doing?").

### What we took from it

- **The judge matters more than the character.** A model that hands out wins for "Holy Christmas" breaks the game, however funny it is.
- **A strong model makes a bad easy level.** Pick for "beatable", not for "best".
- **Ask a political question before you ship.** Neutral in one prompt doesn't mean neutral in the next.
- **Price only matters at the edges.** Three of the five cost about a cent for the whole run.

For local development, `gemma3:27b` in Ollama is the same model as our live one. It's a dense 27B model, so it's slower on a laptop
than a mixture-of-experts model like Qwen3 30B A3B; if it's too slow, run the tests against a scripted fake client and use a real model
only where you need its behaviour. Very small models (around 4B) leak on almost anything. The game talks to models only through one
OpenAI-compatible chat client, so switching models is a config change.

## Credits

The style rules we tested come from the `unslop` skill in [cursor/plugins](https://github.com/cursor/plugins/blob/main/pstack/skills)
(pstack), which we translated into German for our own use.

## Licence

[MIT](LICENSE). Use the prompt, change it, build your own game with it.
