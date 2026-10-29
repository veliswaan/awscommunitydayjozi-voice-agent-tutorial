# Build a Voice AI Agent with Kiro IDE and Amazon Nova Sonic

This is a hands-on tutorial that we'll follow to build a real-time **voice** AI agent you can actually talk to, using the **Kiro IDE** as our pair-programmer, **Strands Bidi Agents** as the framework, and **Amazon Nova Sonic** as the speech-to-speech model. We'll then push the project to **GitHub** and publish a public landing page for it using **GitHub Pages**.

**Duration**: 45 – 60 minutes  
**Difficulty**: Beginner  
**What you'll build**: *Jozi Buddy* (or any voice agent of your choice) — an AI agent that listens through your microphone and talks back through your speakers, in near real time.

> **How this tutorial is structured.** The tutorial runs in phases that build on each other — **Phase 1** (set up Kiro and your project) → **Phase 2** (your first talking agent) → **Phase 3** (push to GitHub) → **Phase 4** (publish a landing page with GitHub Pages). You can stop after any phase and still have something that works: by the end of Phase 2 you already have a talking agent on your laptop; by the end of Phase 4 you have a public project page.

> **A note on Kiro and Strands.** [Kiro](https://kiro.dev) is an AI-powered IDE that pairs with you to design and write code. Instead of pasting code snippets from this tutorial yourself, you'll ask Kiro to build each piece — and this tutorial will show you both the prompts to use *and* the code Kiro should produce, so you can double-check its work. [Strands Agents](https://strandsagents.com/) is the open-source SDK we'll use to wire the agent up, and its new **Bidi** (bidirectional) API makes two-way voice conversations easy.

## Table of Contents

- [1. Prerequisites and account setup](#1-prerequisites-and-account-setup)
- [2. Phase 1: Set up Kiro and your project](#2-phase-1-set-up-kiro-and-your-project)
- [3. Phase 2: Your first talking agent](#3-phase-2-your-first-talking-agent)
- [4. Phase 3: Push your code to GitHub](#4-phase-3-push-your-code-to-github)
- [5. Phase 4: Publish a landing page with GitHub Pages](#5-phase-4-publish-a-landing-page-with-github-pages)
- [6. What's next](#6-whats-next)

---

## 1. Prerequisites and account setup

### What you'll need

- **Python 3.11 or 3.12** installed on your laptop. (3.13 works on most things but we're using PyAudio under the hood, so stick to 3.11 or 3.12 if you want zero-friction installs.) Download from [python.org](https://www.python.org/downloads/).
- **[Kiro IDE](https://kiro.dev)** installed and signed in.
- **A working microphone and speakers (or headphones).** Headphones are nicer — the agent won't hear itself echoing — but we'll turn on echo cancellation so built-in laptop speakers work too.
- **An AWS account** with Amazon Bedrock available in `us-east-1`. New accounts come with free credits that are more than enough for this tutorial.
- **A GitHub account.** Free is fine. Grab one at [github.com/signup](https://github.com/signup) if you don't have one.
- **git** installed locally. Check with `git --version`; if it's missing, install from [git-scm.com](https://git-scm.com/).

### A quick word on cost

Amazon Nova Sonic is billed per minute of audio in and out. For this tutorial — a few short conversations while you build and test — expect to spend well under **$1 USD**. If you have AWS free credits, you likely won't pay anything.

### Enable Amazon Nova Sonic model access

Before you can use Nova Sonic, you have to turn it on in your Bedrock account:

1. In the AWS Console, search for **Amazon Bedrock** and open it.
2. Set your region to **us-east-1** (top-right of the console).
3. In the left sidebar, click **Model access**.
4. Find **Amazon Nova 2.5 Sonic** in the list, click **Request model access** (or **Modify model access**), check the box next to it, and submit. Access is granted instantly.

### Configure AWS credentials on your laptop

Your Python code needs AWS credentials so it can call Nova Sonic. The easiest way is a long-lived access key for a dedicated IAM user.

1. In the AWS Console, go to **IAM → Users → Create user**, name it `voice-agent-dev`, choose **Attach policies directly**, and attach **`AmazonBedrockFullAccess`**.
2. Finish creating the user, open it → **Security credentials → Create access key → Command Line Interface (CLI)**. Save the **Access Key ID** and **Secret Access Key** somewhere safe.
3. Create two plain-text files in your home folder:

**`C:\Users\YOUR_USERNAME\.aws\credentials`** (Windows) or **`~/.aws/credentials`** (macOS/Linux):

```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
```

**`C:\Users\YOUR_USERNAME\.aws\config`** (Windows) or **`~/.aws/config`** (macOS/Linux):

```ini
[default]
region = us-east-1
output = json
```

> **PowerShell tip for API keys.** If you'd rather not leave your secret key in a plain-text file, use `Read-Host -AsSecureString` to paste it into a PowerShell session environment variable and skip the `credentials` file for the day. For a workshop, the file is fine.

That's it for setup. On to the fun part.

---

## 2. Phase 1: Set up Kiro and your project

### 2.1: Create the project folder

Open a terminal and create a fresh folder for your agent:

**Windows (PowerShell):**

```powershell
mkdir jozi-voice-buddy
cd jozi-voice-buddy
```

**macOS / Linux:**

```bash
mkdir jozi-voice-buddy && cd jozi-voice-buddy
```

> **Pick your own name.** I'm calling mine `jozi-voice-buddy` because we're at AWS Community Day Johannesburg, but you can name yours anything. The code in later phases uses *Jozi Buddy* as the agent's personality — if you want yours to be a cooking assistant, a workout coach, or a bedtime-story narrator, just swap the system prompt when we get there.

### 2.2: Open the folder in Kiro

Launch Kiro, choose **File → Open Folder…**, and select the folder you just created. Kiro will trust the workspace and open a chat panel on the right — that's where we'll do most of our building.

### 2.3: Create a Python virtual environment

Virtual environments keep this project's packages isolated from the rest of your system. In Kiro's terminal (**Terminal → New Terminal**), run:

**Windows (PowerShell):**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**macOS / Linux:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

You should see `(.venv)` appear at the start of your prompt. Great.

> **On Windows, trust the venv's Python directly.** If you ever hit a weird "wrong Python" issue, run the venv's interpreter explicitly: `.\.venv\Scripts\python.exe your_script.py` instead of just `python your_script.py`.

### 2.4: Install the Strands Bidi SDK

Inside the venv, install the Strands SDK with the bidirectional-voice extras:

```powershell
pip install "strands-agents[bidi-all,bidi-pyaudio]"
```

What this installs:

- `strands-agents` — the Strands Agents SDK.
- `[bidi-all]` — the bidirectional-streaming support plus every voice-model provider (we need the Amazon Nova Sonic one).
- `[bidi-pyaudio]` — PyAudio bindings so Strands can read from your microphone and write to your speakers.

> **Heads up on PyAudio.** On Windows with Python 3.11 or 3.12, `pip install` just works — PyAudio ships prebuilt wheels. If you're on an older Python version and the install fails with a build error, switch to Python 3.11 or 3.12 and try again.

### 2.5: Tell Kiro what we're building (a steering file)

Kiro behaves a lot better when it knows the project's intent. Let's give it a one-page steering file that it will read on every turn.

In Kiro's chat panel, type:

> Create a steering file at `.kiro/steering/project.md` that captures what this project is: a local voice AI agent built with Strands Bidi Agents and Amazon Nova Sonic. It runs on the user's laptop and uses the microphone + speakers. The target model id is `amazon.nova-2-5-sonic` in `us-east-1`. The agent's personality is a friendly Johannesburg-based buddy named "Jozi Buddy" that can chat casually and recommend things to do in the city. The user is going to publish this project on GitHub Pages when done. Keep the file short — just the facts Kiro needs to make good decisions.

Kiro will create `.kiro/steering/project.md` and from here on it has the right context.

> **Why steering files matter.** Kiro reads everything in `.kiro/steering/` on every chat turn. A short, factual steering file prevents the agent from drifting ("don't you want to deploy this on AgentCore?") and keeps every code suggestion aligned with your actual plan.

---

## 3. Phase 2: Your first talking agent

Let's build the simplest useful voice agent — one that just listens and talks back.

### 3.1: Ask Kiro to scaffold `voice_agent.py`

In Kiro's chat, paste this prompt:

> Create `voice_agent.py` at the project root. It should be a Strands Bidi Agent powered by Amazon Nova Sonic (`amazon.nova-2-5-sonic` in `us-east-1`) that uses the laptop's microphone and speakers. Use the GA `strands.bidi` imports (not `strands.experimental.bidi`). Enable `AudioIO(audio_processor=True)` for echo cancellation so the agent doesn't cut itself off through open speakers. The agent is "Jozi Buddy", a friendly Johannesburg-based voice companion — keep its system prompt short, warm, conversational, and ask it to keep each spoken reply to one or two sentences. The script should run until the user presses Ctrl+C and print a clear message when the conversation ends.

Kiro will produce something close to the code below. Review it against this — if anything differs materially, ask Kiro to reconcile.

### 3.2: What `voice_agent.py` should look like

```python
"""Jozi Buddy — a Strands Bidi voice agent powered by Amazon Nova Sonic."""

import asyncio

from strands.bidi.agent import BidiAgent
from strands.bidi.io import AudioIO
from strands.bidi.models import BedrockNovaSonicModel


SYSTEM_PROMPT = """You are Jozi Buddy, a warm and friendly voice companion
based in Johannesburg, South Africa. You chat casually with the user, help
them think through small decisions, and share tips about the city when it
comes up. Keep spoken replies to one or two short sentences — this is a
voice conversation, not an essay. If the user says goodbye, say goodbye
back warmly."""


async def main() -> None:
    model = BedrockNovaSonicModel(
        model_id="amazon.nova-2-5-sonic",
        region="us-east-1",
    )

    agent = BidiAgent(
        model=model,
        system_prompt=SYSTEM_PROMPT,
    )

    # audio_processor=True enables echo cancellation, noise suppression, and
    # automatic gain control. Essential for laptop speakers; harmless with
    # headphones.
    audio = AudioIO(audio_processor=True)

    print("Jozi Buddy is listening. Start talking — press Ctrl+C to end the chat.\n")

    await agent.run(
        inputs=[audio.input()],
        outputs=[audio.output()],
    )


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\nConversation ended. Goodbye!")
```

### 3.3: Run it

Make sure your venv is active, then:

```powershell
python voice_agent.py
```

You should see **"Jozi Buddy is listening..."**. Say "Hi" out loud. After a short pause the agent will respond through your speakers. Try:

- *"What's a good spot in Sandton to grab breakfast?"*
- *"I'm feeling nervous about my talk later — any advice?"*
- *"Tell me a short joke about load-shedding."*

Press **Ctrl+C** when you're done.

> **"Speak..." is a prompt, not a hang.** When Strands shows `Speak...` in the terminal, it's telling you the agent is listening for your voice. If your laptop stays quiet, Jozi Buddy stays quiet too. Say something.

> **No sound coming out?** Check your OS output device — Strands uses the default output device. Also check that your microphone isn't muted at the hardware level. If the agent cuts itself off a few words into every reply, you don't have echo cancellation enabled; make sure `AudioIO(audio_processor=True)` is in your code and that you installed `strands-agents[bidi-all,bidi-pyaudio]` (the `bidi-aec` extra lives inside `bidi-all`).

### 3.4: Change the voice (optional, but try it!)

By default, Jozi Buddy speaks with **Matthew**, Nova Sonic's en-US masculine-sounding voice. You can swap that out in one line — just pass a `voice=` argument to `BedrockNovaSonicModel`:

```python
model = BedrockNovaSonicModel(
    model_id="amazon.nova-2-5-sonic",
    region="us-east-1",
    voice="tiffany",   # <-- try different voice IDs here
)
```

Save, re-run `python voice_agent.py`, and your agent now sounds different. Here are all the voice IDs Nova 2 Sonic supports today, by language and locale:

| Language | Locale | Feminine-sounding | Masculine-sounding |
| --- | --- | --- | --- |
| English (US) | en-US | `tiffany` | `matthew` *(default)* |
| English (UK) | en-GB | `amy` | — |
| English (Australia) | en-AU | `olivia` | — |
| English (Indian) | en-IN | `kiara` | `arjun` |
| French | fr-FR | `ambre` | `florian` |
| Italian | it-IT | `beatrice` | `lorenzo` |
| German | de-DE | `tina` | `lennart` |
| Spanish (US) | es-US | `lupe` | `carlos` |
| Portuguese (BR) | pt-BR | `carolina` | `leo` |
| Hindi | hi-IN | `kiara` | `arjun` |

> **Tiffany and Matthew are polyglots.** Both en-US voices can speak and code-switch mid-sentence across Nova Sonic's seven supported languages: **English, French, Italian, German, Spanish, Portuguese, and Hindi**. The other voices (like `amy` or `olivia`) are tuned to a single locale and don't code-switch.

> **Heads up for South African builders.** isiZulu, isiXhosa, Afrikaans, Sesotho and the rest of SA's official languages are **not** on Nova Sonic's supported list yet. Jozi Buddy will pronounce words like *"yoh"*, *"eish"*, *"lekker"*, or *"sawubona"* the way a US English speaker reading them cold would — close-ish, but not native. If native SA-language speech is a hard requirement, that's a gap to flag with AWS, not something we can fix in code today.

> **Ask Kiro to switch it for you.** If you'd rather not touch the code yourself, drop this in the Kiro chat: *"In `voice_agent.py`, change the Nova Sonic voice to `tiffany`."* Kiro will update the `BedrockNovaSonicModel` call and leave everything else alone.

Full details and sample prompts for each voice are on the [AWS voices-and-languages docs page](https://docs.aws.amazon.com/nova/latest/nova2-userguide/sonic-language-support.html).

### 3.5: Add a `requirements.txt`

Before we push to GitHub, let's capture the dependencies so anyone cloning your repo can reproduce the environment. Ask Kiro:

> Create a `requirements.txt` at the project root listing the exact packages this project needs: `strands-agents[bidi-all,bidi-pyaudio]`. Keep the extras in the specifier so a `pip install -r requirements.txt` picks up the voice stack and PyAudio.

Resulting file:

```text
# requirements.txt
strands-agents[bidi-all,bidi-pyaudio]
```

### 3.6: Add a `README.md`

GitHub Pages renders your README when it generates a landing page, so a decent README does double duty. Ask Kiro:

> Create a `README.md` at the project root for the Jozi Buddy voice agent. Include: a one-line description, a 2–3 sentence "what it does" section, how it works (Strands BidiAgent + Amazon Nova Sonic), prerequisites, setup steps (clone, venv, `pip install -r requirements.txt`, `aws configure`, enable Nova Sonic access in Bedrock), how to run (`python voice_agent.py`), how to end the chat (press Ctrl+C), and a short section on how to make it your own by editing the system prompt. Use plain Markdown and keep it friendly.

Review what Kiro produces and tweak the tone to sound like you.

### What did we do here?

- We created a `BedrockNovaSonicModel` pointing at Amazon Nova Sonic in `us-east-1`.
- We wrapped it in a `BidiAgent` with a short personality.
- We created an `AudioIO` with echo cancellation turned on and handed it to the agent as both input (microphone) and output (speakers).
- We ran the agent until the user pressed Ctrl+C.

That's a real-time, speech-to-speech voice agent in about 30 lines of Python. The *bidirectional* bit means audio flows in both directions at the same time — you can interrupt Jozi Buddy mid-sentence and it will stop talking and listen.

---

## 4. Phase 3: Push your code to GitHub

Time to put this on the internet.

### 4.1: Add a `.gitignore`

Ask Kiro:

> Create a `.gitignore` at the project root that ignores `.venv/`, `__pycache__/`, `*.pyc`, and `.aws/`. We're a Python project that uses a local virtualenv and may have local AWS creds we never want to commit.

Resulting file:

```gitignore
# Python
__pycache__/
*.py[cod]
*.pyc
.venv/
venv/
env/

# Local AWS credentials — never commit
.aws/

# Editor / OS
.DS_Store
Thumbs.db
```

> **Why this matters.** `.gitignore` is your safety net against accidentally pushing secrets. The `.aws/` line is paranoia-in-depth: your real credentials live in `~/.aws/` outside the project, but if you ever symlink or copy them in, this line keeps them off GitHub.

### 4.2: Create a repository on GitHub

1. Go to [github.com/new](https://github.com/new).
2. **Repository name**: `jozi-voice-buddy` (or whatever you named your project).
3. **Visibility**: **Public** — GitHub Pages on free accounts needs the repo to be public.
4. **Do NOT** initialize with a README, `.gitignore`, or license — you already have those locally.
5. Click **Create repository**.

GitHub will show you a page with setup commands. Keep that tab open.

### 4.3: Initialize git locally and push

In Kiro's terminal, from the project root:

```powershell
git init
git add .
git commit -m "Initial commit: Jozi Buddy voice agent"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/jozi-voice-buddy.git
git push -u origin main
```

Replace `YOUR_USERNAME` with your GitHub handle. On your first push, git may prompt you to authenticate — follow the browser flow.

Refresh the GitHub page for your repo. Your code is now online.

> **Double-check what landed.** Open the GitHub page and skim the file list. If you see a `.venv/` folder or a `.aws/` folder, stop and fix your `.gitignore` before continuing — those shouldn't be there.

---

## 5. Phase 4: Publish a landing page with GitHub Pages

Your Python voice agent runs on your laptop, not in the browser. But a public landing page is still useful: it's your project's shop window, and it's a nice thing to drop into a tweet, a conference slide, or a job application. GitHub Pages will host it for free.

We'll serve a single `index.html` from a `docs/` folder on the `main` branch.

### 5.1: Ask Kiro to build the landing page

In Kiro's chat:

> Create `docs/index.html` — a single-file, self-contained landing page for the Jozi Buddy voice agent project. No external dependencies, no frameworks, just HTML + CSS + a tiny bit of vanilla JS. Design brief: a modern dark hero with a warm gradient (deep purple to amber works well), a big project title "Jozi Buddy", a tagline that says it's a voice AI agent powered by Amazon Nova Sonic and Strands Bidi Agents, three short feature cards (Real-time voice, Powered by Nova Sonic, Built with Kiro), a "How to run it locally" section pulling the key commands from the README, a Tech Stack list (Python, Strands Agents, Amazon Nova Sonic, Kiro IDE), and a prominent "View the code on GitHub" button linking to the repo (use a placeholder URL `https://github.com/YOUR_USERNAME/jozi-voice-buddy` that the user will replace). Make it responsive and accessible: good contrast, semantic HTML, visible focus styles, works on mobile.

Kiro will produce a `docs/index.html` with styling, copy, and the GitHub link. Open it locally (right-click → **Open with Live Server** in Kiro, or just drag it into your browser) and give it a once-over.

> **Replace the placeholder.** Open `docs/index.html` and swap `YOUR_USERNAME` in the GitHub URL for your actual GitHub handle. If Kiro's default copy doesn't match your agent's personality, this is a great moment to rewrite it.

### 5.2: Commit and push the landing page

```powershell
git add docs/
git commit -m "Add GitHub Pages landing page"
git push
```

### 5.3: Turn on GitHub Pages

1. On GitHub, go to your repo and open **Settings → Pages** (left sidebar).
2. Under **Source**, choose **Deploy from a branch**.
3. Under **Branch**, choose **`main`** and the **`/docs`** folder, then click **Save**.
4. Wait 30–60 seconds. Refresh the page. GitHub will show a banner like **"Your site is live at `https://YOUR_USERNAME.github.io/jozi-voice-buddy/`"**.

Open that URL in a new tab. There it is — your voice agent's public landing page.

### 5.4: Add the Pages URL to your repo

While we're here, let's make the landing page easy to find from the repo.

1. Back on your repo's main page, click the **⚙️ gear icon** next to the **About** section (top right).
2. Paste your Pages URL into the **Website** field.
3. Add a short description like *"A voice AI agent built with Strands Bidi Agents and Amazon Nova Sonic."*
4. Tick **Use your GitHub Pages website**, save.

Now when someone lands on your repo they see a direct link to the live page.

> **Shareable in one line.** `https://YOUR_USERNAME.github.io/jozi-voice-buddy/` is now a real URL you can send to anyone. Nice work.

---

## 6. What's next

You've built a real-time voice agent, pushed it to GitHub, and published a landing page for it. Everything you'd want to do from here is one Kiro chat away:

- **Swap the personality.** Replace the system prompt and the agent becomes a cooking coach, a study buddy, a rubber-duck debugger, whatever. Keep replies short — this is a voice conversation.
- **Try different voices.** Nova Sonic ships 15+ voices across seven languages — see [Section 3.4](#34-change-the-voice-optional-but-try-it) for the full list and the one-line change to switch between them. `tiffany` and `matthew` can even speak multiple languages in the same sentence.
- **Switch model providers.** The Strands Bidi API is provider-agnostic — swapping `BedrockNovaSonicModel` for `OpenAIRealtimeModel` or a Gemini Live model is a one-line change. Great for comparing which model handles your use case best.
- **Record a demo clip.** A short screen recording of you talking to the agent, embedded on your GitHub Pages landing page, makes the project click for anyone who clicks the link.

### Resources

- [Strands Agents docs](https://strandsagents.com/)
- [Strands Bidi Agents announcement (GA)](https://strandsagents.com/blog/bidi-agents-now-ga/)
- [Amazon Nova Sonic on Bedrock](https://docs.aws.amazon.com/nova/latest/nova2-userguide/sonic-overview.html)
- [Kiro IDE](https://kiro.dev)
- [GitHub Pages docs](https://docs.github.com/pages)

Enjoy your talking agent — and hamba kahle. 👋
