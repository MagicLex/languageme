# languageme

A progressive language-immersion drip for Claude Code. It preprompts, forces
and monitors how much of each reply comes back in a target language, and ramps
that share up on evidence you're keeping up, not on a calendar.

Standalone: one Python file, stdlib only, no pip installs. Modeled on the
`lex-claude` lang mechanism, but a *blend* instead of a hard switch.

## How it works

Three layers, wired as two Claude Code hooks:

- **Preprompt (SessionStart hook).** `languageme hook` injects "render ~N% of
  your prose in Swedish, the rest in French" into every new session. N lives in
  state. Swedish is confined to chat prose. never code, commits, filenames,
  tool arguments, or any clause where a wrong word breaks something.
- **Force.** The injected rule is explicit and self-limiting: meaning stays
  recoverable, precise technical claims stay in the base language.
- **Monitor (Stop hook).** `languageme monitor` reads the finished transcript,
  measures the *real* target-language share you produced (char-weighted,
  sentence-level classifier, no ML dependency), tracks whether you asked for
  help or wrote the language yourself, and moves N per the ramp policy.

The blend only ever grows when the drip is landing and you're not drowning.

## Quick start

```sh
./languageme init sv          # seed Swedish at 10%
./languageme install          # wire the two hooks (edits ~/.claude/settings.json)
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

- **auto** (default). Glosses new words as `word (traduction)` on first use,
  fades as they recur.
- **on.** Every target phrase glossed. zero ambiguity, slower learning.
- **off.** Raw, infer from context.

## Commands

```
init <lang>                 seed a target language (sv, lv, de, es, it, ...)
status                      full state readout
statusline                  compact string, e.g. "sv 12%↑ ~11"
hook                        [SessionStart] print the preprompt
monitor [--session P]       [Stop] measure a turn, ramp the %
        [--full] [--quiet]  re-measure whole transcript / silence the report
set pct|goal|floor|step <n> tune the numbers
bump [+n]                   nudge the blend now
goal <n>                    ceiling to ramp toward
help on|off|auto            inline-gloss policy
ramp mastery|calendar|manual
lang [<code>]               show / switch target
install | uninstall         wire / unwire the Claude Code hooks
```

## State

Lives in `$LANGUAGEME_HOME` (default `~/.languageme/state.json`). The code is
portable; the state is the only thing that mutates. `install` backs up your
settings to `settings.json.languageme.bak` before its first edit.

## Notes

- The prose classifier ships Swedish / French / English lexicons. Measurement
  of another target needs its stopword set added to the script (the `SV_STOP`
  block). the drip and preprompt work for any language out of the box, only the
  *measurement* half is lexicon-bound.
- `install` appends its hook entries; it never touches existing hooks (e.g.
  lex-claude's). `uninstall` removes only its own.
