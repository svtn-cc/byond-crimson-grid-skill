# Player-facing text

Rules for everything a player reads in game: `to_chat`, `balloon_alert`, `visible_message`, `name`, `desc`, window and button text, examine output.

**All of it is written in Russian.** This document is in English; every example string below is Russian because that is what ships.

The game is a roleplaying game. Its text is not written by a programmer explaining a system but by a **game master** telling the player what is happening to their character. SS13 does not say "an alien was implanted in you and will hatch in 30 seconds". It says "you feel something stirring inside you".

## 1. Two layers of text

| Layer | Where | How to write it |
|---|---|---|
| **In-game** | everything a player sees during a round | the game master's voice, by the rules below; Russian |
| **Service** | logs, admin messages, code comments, the module's `readme.md` | literal and exact, with numbers; English |

Exact conditions and numbers are for developers and the wiki. They belong in the module's `readme.md` and never reach the in-game layer.

The dice roll readout itself (the "N dice vs. difficulty" box) is drawn by the roll system. It is a separate interface: do not duplicate it or paraphrase it in words.

## 2. The game master's voice

- **Second person, present tense.** «Вы замираете», «тени сходятся вокруг вас».
- **Sensation instead of mechanics.** What the character sees, hears, feels on their skin, what they crave. The player will infer the cause.
- **Leave things unsaid.** «Кажется», «будто», «на миг» — the character does not know the rules the world runs on.
- **Short.** One or two sentences. An event, not a paragraph of lore.
- **The setting is the World of Darkness.** Blood, the Beast, hunger, shadows, an alien will, the cold of a dead body. No officialese and no game terms.

### Banned from the in-game layer

- numbers and units: seconds, tiles, percentages, points, levels, successes;
- stat names and rules vocabulary: «бросок», «проверка», «сложность», «кулдаун», «перезарядка», «эффект», «способность», «активирована», «баф», «стан», «радиус»;
- interface words given as a cause: «нажмите», «кликните» — except as the label of the button itself;
- explanations of how the power is built: «спадает, если…», «действует, пока…».

A condition the player must know in order to use a power is given as a property of the world, not as a rule: not «движение снимает эффект» but «стоит шевельнуться — и тень отпустит вас».

## 3. Do not leak what is hidden

Text tells only what the character could have perceived.

- The victim does not see the Discipline's name or the user's name unless they could have understood it.
- A failed attempt against someone must not tell the target that an attempt was made.
- A message to an observer does not reveal what the mechanic hides from them (an invisible person, a mask, a true name).
- "Nobody is there" and "failed to notice" read identically.

## 4. Point of view: three messages for one event

A noticeable event is seen from three sides, each with its own text:

| To whom | What it describes |
|---|---|
| the one acting | their own sensations and intent |
| the one acted upon | what they felt, without knowing the cause |
| bystanders (`visible_message`) | only what is outwardly visible |

A side with nothing to perceive gets no message.

## 5. RP directives with no mechanic behind them

Some consequences the code does not enforce: the character forgot what they saw, came to trust someone, froze in terror, must obey a word. Playing it out is on the player, and the player **must notice it**. An ordinary chat line drowns in the scroll.

Every such directive goes out on three channels at once:

1. **Large conspicuous chat text.** For influence on the mind — `span_hypnophrase` (shimmering, as Dominate uses); for a command or terror — `span_phobia`; for bodily threat — `span_userdanger`.
2. **A `balloon_alert` over the character** — two to four words, lowercase first letter, no full stop. This is what the player sees without looking at the chat.
3. **A screen flash** — `flash_color(mob, colour, 2 SECONDS)`. The colour is tied to the school of influence; for influence on the mind it is the purple `#b066ff` Dominate uses.

A flash-plus-balloon sample in code: `dominate_feedback()` in `modular_darkpack/modules/powers/code/discipline/dominate/dominate.dm`.

### How to phrase a directive

The game master does not order the player around («отыгрывайте, что забыли»). They **state as fact what the character now believes** — and from that it is clear how to play.

- A statement, not a request. «Здесь никого не было» rather than «вы должны забыть, что здесь кто-то был».
- It describes the character's state from the inside: a thought, a certainty, a gap in memory, a pull.
- Unambiguous in meaning: having read it, the player knows what their character knows and does not know.
- No «отыгрыш», «РП», «по правилам», «вы обязаны».
- When the state ends, a separate message on the same channels says that memory, will or clarity has returned.

Example — a witness forgets a vampire who vanished:

```dm
flash_color(witness, "#b066ff", 2 SECONDS)
witness.balloon_alert(witness, "здесь никого не было")
to_chat(witness, span_hypnophrase("Вы моргаете. Перед вами пустое место — и, кажется, всегда было пустым. С кем вы только что говорили?.. Ни с кем."))
```

Bad — literal and in the imperative:

```dm
to_chat(witness, span_warning("Вы провалили проверку Воли и должны забыть, что видели вампира."))
```

## 6. Names and descriptions (`name`, `desc`)

- `desc` is one or two sentences about how the thing looks or feels. For a power — what it gives the character, in the world's words.
- A limit without which the power cannot be used is woven in as an image, without numbers.
- No lists of conditions, successes or costs in `desc`.

## 7. Refusal and failure

A "that did not work" message is still the game master speaking: what the world lacked, not which check failed.

- not «недостаточно крови (нужно 2)» but «в жилах слишком пусто, чтобы удержать чужой облик»;
- not «цель вне радиуса» but «слишком далеко — тени туда не дотянуться».

## 8. Balloon text

- two to four words, lowercase first letter, no full stop; `!` and `?` are fine;
- a sound, a sensation or a brief thought: «скрип половицы!», «здесь никого не было», «облик для него»;
- it does not repeat the chat line word for word; it gives the essential.

## 9. Examples: before → after

Every "before" is a real string from this project that was written too literally.

| Before | After |
|---|---|
| «Идите сквозь толпу, не попадаясь никому на глаза. Шум шагов, вода, шёпот и возня с вещами могут вас выдать.» | «Взгляды соскальзывают с вас, как вода со стекла. Но мир помнит о вас: плеск, скрип, неосторожное слово — и чары рвутся.» |
| «Вас видят, но принимают за другого. 1–2 успеха — вас не узнают, 3 — чужой облик, 4+ — облик и голос.» | «Чужие глаза дорисовывают вам лицо, которого у вас нет. Чем крепче ваша воля, тем полнее обман — вплоть до голоса.» |
| «Вы сдвинулись с места - Плащ теней больше вас не скрывает!» | «Вы шевельнулись — и тени, обнимавшие вас, расступаются.» |
| «Свет падает прямо на вас - Плащ теней рассеивается!» | «Свет ложится на плечи, и укрывавший вас мрак тает.» |
| «Вы привлекли к себе внимание - Незримое присутствие рассеивается!» | «Чей-то взгляд цепляется за вас. Наваждение спадает.» |
| «Вы отошли слишком далеко - чужое Сокрытие спадает.» | «Чужая воля, укрывавшая вас, истончается с каждым шагом — и рвётся.» |
| «Вы не исчезли до конца: окружающие видят смутную призрачную фигуру.» | «Вы таете — но не до конца. В воздухе остаётся ваш смутный, дрожащий силуэт.» |
| «Не хватает крови, чтобы наложить Маску.» | «В жилах слишком пусто, чтобы удержать на себе чужое лицо.» |
| «Вы не замечаете ничего необычного.» | «Вы всматриваетесь в тени. Тени молчат.» |
| «Пелена спадает: вы замечаете то, что пытались от вас скрыть!» | «Воздух идёт рябью — и там, где только что было пусто, проступает чей-то силуэт.» |
| «Это иллюзия! Перед вами Имя — не тот, за кого себя выдаёт.» | «Лицо плывёт, как отражение в воде. Под ним — другое: Имя.» |
| «Кто-то только что исчез у вас на глазах. Голова идёт кругом!» | «Только что здесь стоял человек. Теперь — никого. Земля уходит из-под ног, мысли путаются.» |

The examples set the tone; they are not stock phrases. Write each text for its own event.

## 10. Formatting

- Russian throughout. Use the letter «ё» and «ёлочки» quotes in new text. Do not restyle existing upstream strings to match.
- Inflect with `declent_ru()`, gender with `ru_p_them()` and its siblings; do not drop a nominative name into any slot.
- `span_notice` — neutral; `span_warning` — something is off; `span_danger` — harm; `span_userdanger` — large, a threat to life; `span_hypnophrase`, `span_phobia` — only for influence on the mind and RP directives, or they stop standing out.

## 11. Checking a string before handing off

1. Is it Russian?
2. Does it contain a number, a unit, or a word from the rules or the interface? Rewrite it.
3. Could the character have perceived this? Does the string give away a name, a power or a fact the mechanic hides?
4. Is it a description of a sensation, or an explanation of how the code works?
5. If it is an RP directive with no mechanic — are all three channels there: large text, balloon, flash? Is it clear from the text what the character now believes?
6. Would a game master say this aloud at the table?
