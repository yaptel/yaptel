# yaptel

the yaptel ecosystem is deliberately not just one thing, but split into separate sister projects on the side of research side and application with a sharing of underlying philosophy while still being separate independent projects in the same ecosystem with major overlapping work.

---

## the problem 

most ai voice/chat agents have exactly one personality, and it's the same personality no matter who's talking to it or how. that's not how real speech works. the way you text your best friend is not the way you text your boss. the way you talk on the phone is not the way you post on twitter. people shift constantly — vocabulary, tone, formality, even which language they reach for — depending on the medium and the audience, usually without noticing they're doing it. 

**yaptele** measures that shift instead of ignoring it. **regent** takes the result and gives it somewhere real to live — a phone number the agent actually owns, reachable by call, text, or whatsApp, remembering you the same way across all of them.

---

## yaptele — the research side

*yap (yapping) + tele (telephony)*

yaptele's job is to answer one question properly: **how does this specific voice actually change depending on where and to whom it's speaking?**

most tools skip this and just write a system prompt like "act like a witty, sarcastic 25-year-old" — a handful of adjectives, applied identically everywhere. yaptele instead tries to actually *measure* someone's (or something's) voice, the way a linguist would, and keep that measurement broken into parts instead of mashing it into one vibe.

### the three layers

yaptele splits "voice" into three separate, independently-measured layers. the point of keeping them separate is that if a reconstructed voice ever "sounds off," you can point to *which* layer is wrong instead of just shrugging.

**1. lingvist — which language(s), and how they move between them**

this is the layer for people (or characters) who speak more than one language, or who blend languages together in speech. the clearest example is code-switching — when a bilingual speaker drops into a second language mid-sentence, usually for a reason: emphasis, a joke that only works in that language, a phrase with no clean translation, or as a signal of who they're talking to. "dios mio, how did this happen" isn't an accident — it's a specific, patterned move. lingvist tracks:
- which language(s) someone primarily speaks
- how often and *why* they switch between them (emphasis? quoting someone? no good translation exists?)
- transliteration habits (writing one language's sounds using another language's alphabet)
- go-to loanwords — words borrowed from another language that show up as a default, not a stylistic choice

**2. lexica — the actual words**

this is vocabulary, specifically. not tone, not grammar — just which words someone reaches for. this layer tracks:
- slang that shows up regularly
- generational markers (words/phrases that peg someone to an age range or era)
- filler words ("um," "like," "you know") — and *which* fillers, since these vary a lot person to person
- hedges (softening language — "kind of," "I guess," "maybe")
- idioms or turns of phrase someone defaults to

**3. lingvica — delivery: tone and register**

this is *how* something is said, separate from *what* words are used. it shifts depending on medium and audience even when the vocabulary barely changes. this layer tracks:
- formality level, and how it changes by medium (a text to a friend vs. a work slack message)
- warmth, sarcasm, playfulness, confidence — scored, not just described
- how punctuation is used to convey tone (a period at the end of a text can read as cold; "..." can read as hesitation; excessive exclamation points read as forced enthusiasm)
- emoji density and habits

### why three layers instead of one

a single collapsed "personality" makes debugging impossible. if a reconstructed voice sounds wrong, "the personality is a bit off" tells you nothing actionable. but if you know code-switching frequency is accurate, vocabulary is accurate, but the *tone* register is too formal for the context — now you know exactly what to fix. every tuning run in yaptele reports a **delta vector**: a breakdown showing which specific layers moved and by how much when a change was made, so you can confirm a tone adjustment didn't accidentally also change someone's slang usage. a tool that can't move one trait without dragging three others along with it isn't actually under control yet — it's still guessing.

### where the source material comes from

the corpus (the body of writing/speech being studied) can come from a few different kinds of targets, and each has a different consent situation attached:

- **a real public figure** — someone with a large, genuinely public body of speech (social media, interviews, public writing). the bar here is that the corpus has to already be public and the person has to be enough of a public figure that studying their public speech is a reasonable thing to do.
- **a fictional or public-domain character** — sherlock holmes, or any character with enough established written dialogue to actually measure a voice from, rather than guess at one. no living-person consent question here, but the same "don't invent what isn't there" rule still applies.
- **yourself** — if you want an agent tuned to sound like you specifically, using your own writing/texting history as the source.

what stays constant across all three: only public, already-published material goes in. nothing private, nothing behind a login wall, nothing scraped by getting around access controls. and if a given medium (say, a public discord presence) doesn't exist for the target, that's recorded as **absent** — not filled in with a plausible-sounding guess. a missing data point is a correct, honest result. an invented one is not.

### how it actually gets built

roughly:
1. gather the target's public speech across at least two different mediums (say, twitter and one other place they've spoken publicly).
2. extract signal from that raw material into the three layers above — turning "a pile of tweets" into structured data about code-switching rate, vocabulary habits, and tone patterns.
3. test the result by generating synthetic speech in mediums where there's no direct real-world sample to check against (say, generating a plausible slack message in that voice, even with no real slack messages from the target to compare it to).
4. score the result — either against real samples where they exist, or by having another model judge how consistent and plausible it is.
5. package all of that into a **voice profile** — a structured, traceable object, not a paragraph of adjectives.

that voice profile is the handoff point to regent.

### the honesty rule

every reconstruction discloses that it's a reconstruction — not the real person or the original source — if asked, mid-conversation, no exceptions. this isn't a nice-to-have, it's the thing that separates "a research project studying how voice shifts across mediums" from "a tool for impersonating someone to deceive people." those are very different projects, and yaptele is explicitly the first one.

---

## regent — the delivery side

*reach + agent*

yaptele produces a voice profile. regent is what gives that profile somewhere to actually live.

### the gap regent fills

most "ai phone agent" products give you a call handler — something that answers calls, or places them, and that's the whole product. that's useful, but it's not how a person is actually reachable. a real contact isn't "the phone rings and something picks up" — it's a person you can call, text, or message on whatever app is convenient in the moment, and who remembers the conversation regardless of which one you used last.

regent's goal is to give an agent that same kind of reachability: **one phone number, working across every channel that number implies**, with a single continuous memory underneath all of them.

### the channels, and the order they're being built in

1. **regular text messages (SMS)** — first, because it's the simplest infrastructure and works everywhere without needing platform approval.
2. **whatsApp** — second. a known, well-documented integration path, with much larger reach than SMS internationally.
3. **imessage** — deliberately later. there's no public, official way to send/receive iMessage programmatically — doing it at all requires a real workaround (a relay device, or a third-party service standing in for one), so it's not something to depend on early.
4. **voice calls** — not "above" the other channels, just one more way in. the agent isn't a voice bot that can also text; it's one identity reachable multiple ways, of which a phone call is one option, not the default.
5. **voicemail** — part of the same identity, not a separate fallback system. if a call isn't picked up, the greeting and the "leave a message" prompt are delivered in the target's own voice profile, not a generic recording. messages left get folded into the same shared memory as everything else, so a voicemail is just another kind of turn in the ongoing conversation, referenceable later on any channel.

### the agent can reach out first

this isn't purely reactive. the agent can text someone first — a follow-up on something discussed earlier, a reminder, something worth surfacing before the person asks. that's a different situation than someone calling in and the agent disclosing what it is on the spot — here, the agent is the one initiating contact. the same disclosure rule from yaptele applies regardless of who started the conversation.

### the actual point: it doesn't forget itself between channels

here's the thing that makes this worth building instead of just shipping "a voice bot" and "a texting bot" that happen to share a name: **continuity.** if someone's on a call and says "yeah, I just sent you that on WhatsApp" — the agent needs to already know what "that" refers to, or be able to go check, the same way a person would say "oh yeah, let me pull that up" instead of "sorry, what are you talking about?"

that only works if conversation memory is tied to *the person*, not to *the channel*. a call and a text thread with the same person need to be treated as one ongoing conversation with two different kinds of turns in it — not two separate conversations that just happen to sound similar.

concretely, that shows up as a real, live capability: mid-call, if someone references something from another channel, the agent can actually go check — pausing briefly the way anyone does when they're pulling something up, not faking that pause for effect. if there's no real lookup happening, there's no fake pause either — disclosure applies to *that* too, not just to "am I a reconstruction." the agent is allowed to be visibly doing a real lookup. it's not allowed to perform the appearance of one.

### what regent deliberately isn't

- it doesn't invent a presence on a channel the target has no real footprint on — same absent-is-correct principle yaptele uses for mediums.
- it doesn't relax the disclosure rule on any channel. "this is a reconstruction, not the real thing" applies everywhere the agent is reachable, not just on calls.
- it's not a general-purpose messaging bot. the point isn't "an AI that happens to text you" — it's one consistent identity, reachable several ways, that never loses track of itself moving between them.

---

## how the two fit together

```
yaptele                                   regent
──────────────────────                    ──────────────────────
gather public speech
        ↓
measure lingvist / lexica /
lingvica, separately
        ↓
package as a voice profile   ────────→    render per channel:
                                            - spoken voice for calls
                                            - texting register for SMS
                                            - texting register for WhatsApp
                                                     ↓
                                           one phone number, all channels,
                                              one shared memory of
                                                  the person
```

yaptele answers: *what does this voice actually sound and read like, and how sure are we?*
regent answers: *where can you reach it, and does it remember you when you do?*

see `research.md` for the full detail on how yaptele's measurement actually works, and `regent.md` for the full detail on the delivery/memory architecture.
