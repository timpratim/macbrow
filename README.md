# macbrow

Talk to your Mac. Say a command and it runs as AppleScript; say a web task and it drives
your Chrome. Routing takes about 300 ms because a System One model *chooses* instead of
generating.

[![macbrow demo](https://img.youtube.com/vi/cPBlb1neXiI/maxresdefault.jpg)](https://youtu.be/cPBlb1neXiI)

*Demo video: [youtu.be/cPBlb1neXiI](https://youtu.be/cPBlb1neXiI)*

> **Experimental. Not for production.** This prototype lets a language model run scripts
> and click around a browser on your machine. An early version, asked to "clean up my
> desktop", moved every file on the Desktop into a folder. The safety policy in
> [`macbrow/policy.py`](macbrow/policy.py) exists because of that. Keep it on, read it
> first, and don't point voice control at a machine whose setup you can't afford to lose.

```
Gradium STT ─► Jev picks a tool + its arguments (one request, ~300 ms) ─► AppleScript ─► Gradium TTS
                 ├─ website task ─► jev-ultrafast drives Chrome, one Jev request per step
                 ├─ unknown action ─► LLM writes a new tool, checked, cached for next time
                 └─ small talk ─► LLM, one sentence
```

Stack: [Gradium](https://gradium.ai) streaming speech-to-text and text-to-speech for the voice
in and out (via the [Gradium plugin for LiveKit Agents](https://docs.livekit.io/agents/models/tts/gradium/)),
[LiveKit Agents](https://docs.livekit.io/agents/) for the voice loop, [TypeSafe's Jev](https://docs.typesafe.ai)
for every decision, [jev-ultrafast](https://github.com/browser-use/jev-ultrafast) (Browser Use ×
TypeSafe) for the browser, and an LLM (GPT-5-mini via LiveKit Inference, or a local model in
LM Studio) only for writing text.

## Why it's fast: choose, don't generate

A typical computer-use agent runs a loop of *screenshot → LLM reasons → emits an action*.
Each turn is a generation call: seconds of latency, free-form output that has to be parsed,
and a model that can hallucinate a button that isn't there.

macbrow inverts this. Code owns the workflow and hands Jev small, typed questions:

- **Routing is one Choice.** The options are the tools that apply to the apps open *right
  now*, plus "chat" and "new action". Jev returns a probability for every option, not
  prose. ~300 ms, and the confidence is a number the code can act on: high → run, middling →
  ask, low → don't.
- **Arguments ride along speculatively.** In the same request, every enum argument of
  every candidate tool is asked as its own Choice ("which app?", "louder or quieter?").
  Only the winner's answers are read. Free-text arguments are filled by having Jev *select*
  the right span of the utterance rather than write one. No second round-trip, no parsing.
- **The browser is the same idea per step.** jev-ultrafast reads the page as an indexed
  table of controls, then one Jev request picks an operation (click, type, select, scroll,
  wait, done) and a target index for each operation. Nothing is generated except the text
  to type. Model output never becomes a selector or code; every target is an element the
  code observed. No screenshots in the loop.
- **Judgments, not chat.** The same primitives handle the rest: is this a follow-up to the
  last web task? Does the request have the dates a form needs? Did the generated script
  really do what was asked? Each is a single Noul or Choice, a few hundred milliseconds.

Measured on this machine: "add the best vacuum cleaner to my Amazon cart" completed in 4
steps and 11 s; "the return date should be November 4th" as a follow-up in the same tab, 6
steps and 3.7 s; "play the Love Hypothesis trailer" from end of speech to video playing,
1.6 s.

## Voice: Gradium

Every word macbrow hears and says goes through [Gradium](https://gradium.ai). Gradium's
streaming STT turns the mic into text with end-of-turn detection, and its TTS speaks the
replies; the greeting, confirmations, clarifying questions and results all come from the
same voice. Both run through the official `livekit-plugins-gradium` package, so swapping
voices is one env var (`GRADIUM_VOICE_ID`) and the rest of the pipeline never touches audio.

Get an API key at [gradium.ai](https://gradium.ai) and see the [Gradium docs](https://docs.gradium.ai)
for the voice library, model names and the raw STT/TTS APIs.

## Setup

```bash
uv sync
cp .env.example .env.local   # GRADIUM_API_KEY (gradium.ai), TYPESAFE_API_KEY, LIVEKIT_* (or lk app env -w)
```

- macOS asks for Automation access per app the first time. Grant it in System Settings ▸
  Privacy & Security ▸ Automation.
- For browser tasks, once: open `chrome://inspect/#remote-debugging` and tick **Allow remote
  debugging for this browser instance**. Click Allow on Chrome's sheet at first connection.
- `MACBROW_CHROME_PROFILE_EMAIL` pins Chrome to one Google account; unset, the last-used
  profile is kept.
- `MACBROW_LLM_PROVIDER=lmstudio` uses a local model on port 1234 instead of LiveKit
  Inference.

## Run

```bash
./console.sh start                                # voice, local mic and speaker; ./console.sh log to read back
uv run python agent.py console                    # same, foreground
uv run python -m macbrow.cli --dry "open github dot com"   # route only, no audio
uv run python -m macbrow.cli --policy             # every tool and whether policy allows it
```

## What it can and can't do

**Allowed:** open, switch and quit ordinary apps; open files, folders and URLs; browser tabs
and web tasks; Slack, Messages and Mail; Notes, Reminders, Calendar; media, volume,
notifications, screenshots, dark mode.

**Blocked:** System Settings and preference writes; terminals and developer tools; shell
commands outside a short allow-list; package managers and language tooling; system and
config paths; deleting, moving or duplicating files; emptying the trash; power and session
control; admin privileges; keychain and passwords. In the browser: buying, checkout,
payments, sign-in and account changes.

The policy runs when a tool is loaded, when a new tool is generated, and on the rendered
script right before execution. Anything that changes state (a risky script, an add-to-cart,
a freshly written tool) is read back and needs a spoken yes. It does not protect against a
mis-heard word that happens to be a valid command; `MACBROW_POLICY=off` exists for debugging
and the greeting says so out loud.

## Layout

| Path | Role |
|---|---|
| [`agent.py`](agent.py) | LiveKit entrypoint: Gradium STT/TTS, router before LLM |
| [`macbrow/router.py`](macbrow/router.py) | Jev routing, speculative arguments, follow-up and completeness judgments |
| [`macbrow/agent.py`](macbrow/agent.py) | State machine: act, ask, confirm, learn |
| [`macbrow/registry.py`](macbrow/registry.py) + [`tools/seed.json`](tools/seed.json) | Tools with typed argument slots; learned ones go to `tools/learned.json` |
| [`macbrow/generator.py`](macbrow/generator.py) | LLM writes a new tool: compile, effect, policy and Jev-review gates, 3 repair rounds |
| [`macbrow/browser_task.py`](macbrow/browser_task.py) | jev-ultrafast in the user's Chrome: pinned profile, follow-ups, clarifying questions, spoken results |
| [`macbrow/policy.py`](macbrow/policy.py) | The safety policy |
| [`macbrow/chrome.py`](macbrow/chrome.py), [`resolvers.py`](macbrow/resolvers.py), [`applescript.py`](macbrow/applescript.py), [`cli.py`](macbrow/cli.py) | Profile lookup, computed args, osascript, text REPL |

Add a tool by appending to `tools/seed.json`: a name, description, optional app `scope`,
`args` (enum with `criteria`, or `text`), an AppleScript with `{{placeholders}}`, and what to
`speak` afterwards. See the existing entries.

## Development

```bash
uv run pytest -q                                  # 26 offline tests
uv run ruff check . && uv run ruff format --check .
```

MIT, see [LICENSE](LICENSE).
