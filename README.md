# languageme

A progressive language-immersion drip for Claude Code. It preprompts, forces
and monitors how much of each reply comes back in a target language, and ramps
that share up on evidence you're keeping up, not on a calendar.

Standalone: one Python file, stdlib only, no pip installs. A *blend* drip, not
a hard language switch.

## How it works

Three layers, wired as two Claude Code hooks plus a statusline segment:

- **Preprompt (SessionStart hook).** `languageme hook` injects "render ~N% of
  your prose in Swedish, keep the rest in your usual reply language" into every
  new session. N lives in state. The target language is confined to chat prose,
  never code, commits, filenames, tool arguments, or any clause where a wrong
  word breaks something.
- **Force.** The injected rule is explicit and self-limiting: meaning stays
  recoverable, precise technical claims stay in your usual language.
- **Monitor (Stop hook).** `languageme monitor` reads the finished transcript,
  measures the *real* target-language share you produced (char-weighted,
  sentence-level classifier, no ML dependency), tracks whether you asked for
  help or wrote the language yourself, and moves N per the ramp policy.

A **statusline segment** (`sv 12%↑ ~11`: target, current blend, last measured)
rides along so you always see where the dial is. `install` wires it
non-destructively: if you already have a statusline it wraps it and appends the
segment, if you don't it adds one. It never edits or replaces your statusline.

The blend only ever grows when the drip is landing and you're not drowning.

## Quick start

```sh
./languageme init sv          # seed Swedish at 10%
./languageme install          # wire hooks + statusline (edits ~/.claude/settings.json)
# start a new Claude Code session, the drip is live
./languageme status           # target, blend %, recent measurements, ramp state
```

Put it on your PATH if you like: `ln -s "$PWD/languageme" ~/bin/languageme`.
The hooks call it by absolute path, so a symlink is optional.

## Ramp policies (`languageme ramp <mode>`)

- **mastery** (default). Ramps one step after 3 consecutive sessions where you
  produced the target blend and asked for zero help. Writing the language
  yourself adds a bonus step. Asking for help resets the streak.
- **calendar.** One step per monitored session until the goal. Hands-off, blind
  to whether you're keeping up.
- **manual.** Never auto-moves. You bump it: `languageme bump +5`.

Natural-language overrides always win, both directions: say "trop de suédois"
(or "för mycket") in chat and the next monitor pass steps the blend *down*; say
"plus de suédois" and it steps up.

## Help modes (`languageme help <mode>`)

- **auto** (default). Glosses new words as `word (translation)` on first use,
  fades as they recur.
- **on.** Every target phrase glossed. zero ambiguity, slower learning.
- **off.** Raw, infer from context.

Add `languageme phonetic on` to layer a short pronunciation respelling on the
tricky words (the å/ä/ö vowels, sj/tj/rs clusters, silent letters), e.g.
`sköldpadda (turtle, "HYULD-pad-da")`. Plain learner respelling, not IPA, only
where the spelling hides the sound. Off by default; ignored when help is off.

## Pause and kill switch

Two levels, so you can always shut it off:

- **Persistent toggle.** `languageme off` pauses the drip (no preprompt, no
  measuring; the statusline shows `sv off`). `languageme on` resumes it. The
  state persists across sessions.
- **Hard kill (env).** `export LANGUAGEME_OFF=1` fully disables it for that
  shell or session, overriding state no matter what. The hook injects nothing,
  the monitor does nothing, and the statusline falls back to pure passthrough
  (your original line, no segment). The guaranteed off switch.

## Commands

```
init <lang>                 seed a target language (sv, lv, de, es, it, ...)
status                      full state readout
statusline [--wrap]         compact segment; --wrap runs the wrapped statusline too
hook                        [SessionStart] print the preprompt
monitor [--session P]       [Stop] measure a turn, ramp the %
        [--full] [--quiet]  re-measure whole transcript / silence the report
set pct|goal|floor|step <n> tune the numbers
bump [+n]                   nudge the blend now
goal <n>                    ceiling to ramp toward
help on|off|auto            inline-gloss policy
phonetic on|off             pronunciation cues on tricky words (å/ä/ö...)
ramp mastery|calendar|manual
lang [<code>]               show / switch target
off | on                    pause / resume the drip (persists)
install | uninstall         wire / unwire hooks + statusline
update                      self-update (git pull the tool's checkout)

Measurable targets: sv, es, lv. Others get the drip but no % yet.
Hard kill for a single shell/session (overrides everything): LANGUAGEME_OFF=1
```

## State

Lives in `$LANGUAGEME_HOME` (default `~/.languageme/state.json`). The code is
portable; the state is the only thing that mutates. `install` backs up your
settings to `settings.json.languageme.bak` before its first edit.

## Notes

- Measurable targets: **sv, es, lv** (plus fr/en as discriminators). Each ships
  a stopword lexicon and its distinctive characters in the `LEXICONS` /
  `LANG_CHARS` registries; the target is scored against all the others. Adding a
  language is those two entries, nothing else. The drip and preprompt work for
  any language out of the box, only the *measurement* half is lexicon-bound (an
  unmeasured target just holds the % at 0).
- `install` appends its hook entries and wraps (never replaces) your statusline;
  it leaves every existing hook and statusline untouched. `uninstall` removes
  only its own and restores the original statusline.
- Scales flat. State is bounded (history and session cursors are capped), each
  Stop reads only the new transcript tail via a byte seek (not the whole file),
  and the read-modify-write is `flock`-serialized so parallel Claude Code
  sessions can't clobber each other's measurements.
