# 🔥 PyroPlayer

**A non-repeating music player that keeps your songs fresh, built for Termux + MPD.**

PyroPlayer is a tiny but clever music player that never plays the same song twice.
It hooks into the popular [MPD](https://www.musicpd.org/) music daemon and turns it
into a worry-free playlist that keeps picking music for you — until you decide you've
heard everything and want to start the cycle over.

Whether you're a Terminal power user or just someone who wants to press "play" and
never think about song selection again, PyroPlayer has your back.

---

## Table of Contents

- [What is this? (for everyone)](#what-is-this)
- [What does it do?](#what-does-it-do)
- [How does it work? (the big picture)](#how-does-it-work)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Command Reference](#command-reference)
- [How it works under the hood](#under-the-hood)
  - [The two halves: Client & Daemon](#client-and-daemon)
  - [The secret to "no repeats"](#the-no-repeat-trick)
  - [A walk through the implementation](#implementation-walkthrough)
- [Project Layout](#project-layout)
- [Contributing](#contributing)
- [License](#license)

---

## What is this? <a name="what-is-this"></a>

Imagine a jukebox that remembers every song it has ever played for you. When you ask
it for music, it refuses to replay anything you've already heard — so you always get
something *new-ish*. That's PyroPlayer.

It sits on top of **MPD**, a well-known, free music server ("daemon") that runs in the
background and knows about every music file on your device. PyroPlayer is the brain that
tells MPD *which* song to play, *when*, and *why* — with a strong rule: **no repeats**.

The target audience is people using **Termux** (a terminal emulator that brings a Linux
environment to Android) — but the ideas apply to any computer with MPD installed.

## What does it do? <a name="what-does-it-do"></a>

- **Plays your music automatically, non-stop.** When one song finishes, it picks another
  one for you.
- **Never repeats within a "cycle."** Every song you play is tracked. PyroPlayer will not
  choose it again until you reset the cycle.
- **Lets you hand-pick songs** with a friendly fuzzy-finder menu, or just go fully random.
- **Plans ahead** — queue up songs to play "next."
- **Rewinds** — go back to the previous song or replay a favourite from your history.
- **Pauses, restarts, stops** — all the basics you expect from a player.
- **Runs quietly in the background** as a daemon, so your music keeps going even as you
  do other things.

## How does it work? (the big picture) <a name="how-does-it-work"></a>

If you're not technical, here's a simple way to picture it:

> MPD is a **music library** — a giant list of all your songs. PyroPlayer is a **DJ** who
> stands in front of it with a clipboard. The clipboard has two lists: **"already played"**
> and **"coming up next."** Each time a song ends, the DJ strikes one off the "already
> played" list, checks the "coming up next" list first — and if that's empty, grabs any
> song *not* already on either list. Only when the whole library has been heard does the
> DJ tear up the sheet and start fresh.

That's the entire magic. Everything else is just the plumbing that makes it smooth,
reliable, and easy to control from your Terminal.

---
---

## Requirements

- **Python 3** (any modern version)
- **MPD** — the music daemon
- **mpc** — the command-line client used to talk to MPD
- **fzf** — a command-line fuzzy finder (used for the interactive song-selection menu)
- A **Termux** (or other Linux) environment

> These are used as command-line *tools* by PyroPlayer; it does not import any
> third-party Python libraries — everything it needs comes from Python's standard library.

## Installation

1. Install MPD, mpc, and fzf in your environment, e.g. on Termux:

   ```bash
   pkg install mpd mpc fzf
   ```

2. Make sure MPD is set up and knows where your music lives (see MPD's own docs for
   configuring `~/.mpdconf`).

3. Grab the `pyroplayer` script into a folder on your `PATH` and make it executable:

   ```bash
   chmod +x pyroplayer
   ```

4. (Optional) create an alias so it's easy to type:

   ```bash
   echo 'alias pyroplayer="python3 /path/to/pyroplayer"' >> ~/.bashrc
   ```

That's it. PyroPlayer creates all of its own data files on first run — no manual setup.

## Quick Start

```bash
# Start PyroPlayer and choose how the first song is picked
pyroplayer start

# Show what's currently playing
pyroplayer c

# Skip to a random new song
pyroplayer n
```

Once started, when songs finish, PyroPlayer keeps choosing new unplayed songs
automatically — you don't have to tell it anything.

---

## Command Reference

Here is every command you can run. In the list, "✔ interactive" means it opens an
fzf menu for you to pick from.

| Command | What it does |
|---------|--------------|
| `pyroplayer start` | Start the background daemon and choose the first song (interactive) |
| `pyroplayer c` | Show the current song and status |
| `pyroplayer pause` | Pause / resume playback |
| `pyroplayer n` | Immediately play a random unplayed song |
| `pyroplayer p` | Play the previous song |
| `pyroplayer r` | Restart the current song from the beginning |
| `pyroplayer q` | Stop PyroPlayer and reset the cycle |
| `pyroplayer s` | Choose an unplayed song and play it now (✔ interactive) |
| `pyroplayer next` | Choose an unplayed song to play later (✔ interactive) |
| `pyroplayer queue` | Show the planned "up next" queue |
| `pyroplayer h` | Choose a previously played song to replay (✔ interactive) |
| `pyroplayer reset` | Reset the current non-repeat cycle |
| `pyroplayer` or `--help` | Show the in-app help |

---

## Under the hood <a name="under-the-hood"></a>

Now for the technical walkthrough. If this is your first time reading code, don't worry —
I'll explain every step in plain language.

### The two halves: Client & Daemon <a name="client-and-daemon"></a>

The whole program is a **single Python file**, but it plays two completely different roles
depending on how you launch it:

1. **The Client** — what you run when you type `pyroplayer c`, `pyroplayer next`, etc.
   It figures out what you asked for, talks to the daemon, and shows you the result.

2. **The Daemon** — launched in the background (`pyroplayer --daemon`). It is the always-on
   worker that:
   - keeps track of what's been played,
   - picks the next song,
   - and listens for your commands.

They talk to each other over a **Unix socket** — think of it as a private telephone line
between two programs on the same machine.

```
┌──────────────┐   command (JSON)   ┌──────────────────────┐
│   CLIENT     │ ──────────────────▶│       DAEMON         │
│  (you type)  │                    │  listens + decides   │
└──────────────┘ ◀───────────────── └──────────────────────┘
                   response (JSON)       │
                                         ▼
                                    ┌──────────┐
                                    │   MPD    │
                                    │ (music)  │
                                    └──────────┘
```

### The secret to "no repeats" <a name="the-no-repeat-trick"></a>

The non-repeat behaviour comes from **three simple text files** the daemon maintains in
`~/.config/pyroplayer/`:

- **`history`** — a fresh list of songs PyroPlayer has played in this cycle.
- **`queue`** — songs you've asked to play "next."
- **`state.json`** — a little note about where you are (used for the "previous song"
  feature).

When PyroPlayer needs a new song, it:

1. Asks MPD for a list of **all** songs.
2. Removes anything already in `history`.
3. Removes anything already in `queue`.
4. Picks randomly from whatever is left.

```
ALL SONGS   ─▶  minus HISTORY  ─▶  minus QUEUE  ─▶  pick one at random
```

Because history grows with every song, the pool shrinks — until it's empty. When that
happens, PyroPlayer wipes both lists and starts a brand-new cycle. No song repeats until
you've genuinely heard everything.

### A walk through the implementation <a name="implementation-walkthrough"></a>

Let's trace what actually happens in the code when you run `pyroplayer start`:

1. **`main()`** (the client) checks the first argument and sees `start`. It calls
   `start_daemon()`.

2. **`start_daemon()`** checks whether the daemon is already alive by sending it a
   `ping`. If not, it launches a detached background copy of itself with `--daemon`,
   then waits (up to 3 seconds) for it to come online.

3. **`run_daemon()`** (now running in the background) does three important things:
   - **Single-instance lock** — it takes an exclusive file lock (`pyroplayer.lock`). If
     another daemon already holds it, it refuses to start twice.
   - **Creates the socket server** — a threaded server that can handle many commands at
     once.
   - **Starts the event watcher** — a background thread that keeps an eye on MPD.

4. The client sends a `status` command. The server's `handle_command()` runs the right
   logic and replies. Since nothing is playing yet, you're asked how to pick the first
   song.

5. If you pick **random**, the client sends `start_random`, which runs `play_random()` →
   `choose_random()` → `play_song()`.

6. **`play_song()`** tells MPD to clear, add, and play your song — but it doesn't trust
   that MPD obeyed. It *polls* MPD up to 20 times over a second to confirm the requested
   file really started. Only then does it record the song in `history`. This careful
   verification is what makes the tracking accurate.

7. Meanwhile, the **`mpd_event_loop()`** thread is blocked on `mpc idle player` — it's
   telling MPD "wake me up whenever the player changes." When a song finishes, MPD wakes
   it, and PyroPlayer notices:

   - If the change was caused by PyroPlayer's own command (tracked by a flag called
     `commanded_change`), it ignores it — no point choosing a new song when *we* just
     asked for this one.
   - If MPD naturally *stopped*, that means the song ended. So it calls `play_next()`,
     which drains the `queue` first, and falls back to a random unplayed song if the
     queue is empty.

That loop is the heart of the "hands-free" experience: **song ends → new song chosen →
repeat** — forever, with no repeats, until `reset`, `q` (stop), or `shutdown`.

A few details worth appreciating:

- **Thread safety.** Two threads (the command server and the event watcher) access the
  same history/queue files. PyroPlayer guards these with a re-entrant lock so changes
  can't collide, and it writes files *atomically* (write to a temp file, then swap it in)
  so a crash mid-write can't corrupt data.
- **No third-party dependencies.** It only uses Python's built-in `socket`, `threading`,
  `fcntl`, `subprocess`, and friends.
- **MQ interplay.** The `mpc idle` call intentionally does *not* hold the MPD lock while
  waiting, so it never blocks other commands. Clever and correct.

---

## Project Layout

```
pyroplayer/
├── pyroplayer        # The entire program (client + daemon, ~1,400 lines)
├── LICENSE           # MIT license
├── README.md         # This file
└── ~/.config/pyroplayer/   # Auto-created at runtime (never committed)
    ├── history       # songs played this cycle
    ├── queue         # songs queued for later
    ├── state.json    # playback position
    ├── pyroplayer.lock   # single-instance lock
    ├── pyroplayer.log   # daemon log
    └── pyroplayer.sock  # live socket (delete-and-recreate on start)
```

> The files under `~/.config/pyroplayer/` are created automatically the first time the
> program runs. **Don't commit them** — they're machine-specific runtime data.

---

## Contributing

PyroPlayer is a small, friendly project, and every bit of help makes it better. Here is
how you can get involved — no matter your skill level.

### Ways to contribute

- **🐛 Report bugs.** Found something that doesn't work as expected? Open an issue and
  tell us what you did, what happened, and (if possible) any output/log from
  `~/.config/pyroplayer/pyroplayer.log`.
- **💡 Suggest features.** Have an idea — like shuffle *without* the non-repeat rule, or
  support for playlists? Share it with a clear description of what you'd expect to happen.
- **📝 Improve the docs.** Good documentation is hard. Typos, clearer examples, or a
  better explanation are all welcome.
- **🧪 Write tests.** Right now the code is verified by hand. Automated tests would make
  it far more robust — this is a big area to help.
- **🔧 Fix bugs or add features.** Dive into the code and send a pull request.

### Development tips

- The whole program is one file, `pyroplayer`. Read it top to bottom — it's organised
  into clearly labelled sections (CONFIG, FILE HELPERS, MPD HELPERS, PLAYBACK, QUEUE,
  STATUS, DAEMON, CLIENT, CLI).
- Keep changes **small and focused** with a clear commit message describing *what* and
  *why*.
- Matching the existing style makes reviews smoother: plain, well-commented Python,
  no dependencies beyond the standard library, and atomic file writes.
- Test your changes in a real (or simulated) MPD environment before opening a pull
  request. The hardened `playback verification` logic is especially important to
  preserve — don't break the promise that "history only records songs that actually
  played."

### Contribution guidelines

1. Fork the repo and create a branch for your work.
2. Make your change, keeping it focused on one thing.
3. Verify it works and explain what you changed in the pull-request description.
4. Open a pull request — someone will review it and work with you to get it merged.

Every contributor — reviewer, tester, writer, or coder — is appreciated. Thank you! 🔥

---

## License

PyroPlayer is open source, released under the **MIT License**. You're free to use,
modify, and share it, even commercially, with attribution. See the [LICENSE](LICENSE)
file for the full text.
