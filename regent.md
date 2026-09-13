# regent - how the delivery layer actually works

*reach + agent.* this doc assumes `readme.md` for context but doesn't require `research.md` to be read first, the parts of a voice profile regent actually consumes are re-explained here where relevant.

---

## what problem regent is solving

yaptele (see `research.md`) produces a **voice profile** aka a structured description of how a specific voice sounds and reads across different mediums. on its own, that's just data. regent is what turns it into something reachable.

the comparison worth drawing: most "ai phone agent" or "ai calling" tools give you a **call handler** aka a thing that answers or places calls, full stop. that's a real capability, but it's not how a person is actually reachable in day-to-day life. nobody experiences a friend as "the thing that answers when i call a number." a real contact is someone you can reach several different ways depending on what's convenient right now, who remembers the last conversation no matter which way you reached them last time.

regent's job is to give an agent that same shape of reachability: **one phone number, multiple channels, one continuous memory underneath all of them.**

---

## the channels

### why a phone number is the anchor

a single phone number is the natural anchor for this because it's already the thing that ties sms, whatsapp, and voice calling together in real-world telephony aka the same number can receive a text, a whatsapp message, or a call. that makes number-based identity resolution (knowing "this text and this call are from the same person") mostly free, rather than something that has to be built from scratch.

### channel 1: sms (regular text messages)

built first, for practical reasons: sms infrastructure is mature, works over standard telecom rails, doesn't require platform-specific business approval, and works across essentially every phone regardless of what apps someone has installed. it's the lowest-friction way to prove the "agent has a real, texting-capable identity" concept before adding platform-specific complexity.

### channel 2: whatsapp

built second. whatsapp has an official business api path, meaning the integration is well-documented and supported, rather than requiring a workaround. it also has far larger reach than sms in large parts of the world, which matters if the agent's reachability is meant to feel real rather than region-limited.

### channel 3: imessage (deliberately later)

imessage has no public, official api for sending or receiving messages programmatically. getting an agent onto imessage requires either a relay device (an actual mac acting as a bridge) or a third-party service standing in as an intermediary — both are real technical dependencies with their own reliability and maintenance concerns, not a clean integration. because of that, imessage support is explicitly sequenced *after* the two channels that don't require a workaround, so the core "one identity, multiple channels, shared memory" concept can be proven and stable before adding a fragile dependency on top of it.

### voice calling

voice sits alongside these channels, not above them. it's one more way to reach the agent, not the primary mode with texting as an add-on. in practice this matters for how the product should be described: not "a voice bot that also texts," but "one identity, reachable several ways, one of which happens to be a phone call."

### voicemail

a real phone number has a voicemail behind it, and regent's does too - this isn't a side feature, it's part of what makes the number feel like it belongs to an actual reachable contact rather than a call-only bot that just doesn't answer sometimes.

two directions this goes:

**when someone calls and the agent doesn't pick up** - the caller reaches a voicemail greeting rendered in the target's own voice profile, not a generic "please leave a message after the tone." the greeting uses the same spoken rendering yaptele's `lingvica` layer already produces for calls (see `research.md`) - tone, warmth, phrasing habits - so the greeting sounds like a continuation of the same identity, not a different, flatter system taking over the moment the "real" agent isn't available. asking the caller to leave a message is itself delivered in-voice ("leave a message and i'll get back to you" rendered in the target's register), not a scripted line that breaks character.

**when a message gets left** - that voicemail isn't a dead-end recording sitting unheard. it gets transcribed and written into the same shared memory graph that a text or a call would write into (see the memory architecture section below), so it's just another kind of turn in the ongoing conversation with that person - something the agent can reference the next time it talks to them, on any channel, the same way it'd reference something sent over whatsapp. "you left me a voicemail about x, want to talk through it now?" is the kind of continuity this is meant to produce.

**agent-initiated outbound calls that hit voicemail** - the reverse case: when the agent calls someone and gets *their* voicemail, it should leave a message that's genuinely useful (not a dead "call me back") and, same as any other channel, disclose what it is if that's relevant to the message being left, consistent with the disclosure rule below.

### outbound-initiated contact

the agent isn't purely reactive at all, it can text first. examples: following up on something raised in an earlier call, sending a reminder, surfacing something relevant before being asked. this is a meaningfully different situation from someone calling in and getting disclosure on the spot, because here the agent is the one initiating contact rather than responding to it. the same disclosure obligation from yaptele (see `research.md`) applies regardless of which side started the conversation. the fact that the agent reached out first doesn't loosen the requirement to be honest about what it is if asked.

---

## rendering one profile across channels

yaptele's voice profile already contains what's needed for this - specifically the **lingvica** layer's `register_by_medium` field, which separately scores how tone/formality shifts for different mediums (twitter register vs. slack register vs. imessage register, etc., per `research.md`). that means regent isn't solving a new research problem per channel - it's a **rendering** problem: given one profile, pick the right register for the channel currently being used.

- a phone call uses the profile's spoken rendering - full lingvist code-switching behavior, lingvica's medium-shift corrections translating written tics (emoji density, all caps, trailing "...") into actual vocal equivalents (expressiveness, stress, real pauses), since a voice call carries none of those written signals directly.
- an sms or whatsapp message uses the profile's texting register directly - punctuation-as-tone, emoji habits, and message length patterns apply mostly as-is, since text is the medium the profile's `lingvica.register_by_medium` values were largely built to describe in the first place.

the underlying voice profile doesn't change between these — only which slice of it gets used, and how it's rendered for that channel's actual medium.

---

## the memory architecture: the actual hard part

### the failure mode this exists to prevent

without shared memory, regent is indistinguishable from running two separate bots that happen to share a voice profile — a voice bot and a texting bot that sound alike but don't know about each other. that defeats the entire point. the value of "one identity, multiple channels" only exists if the agent actually behaves like one continuous entity across them.

### memory has to be scoped to the person, not the channel

concretely: a phone call and a text thread with the same person need to be treated as **one ongoing conversation with two different kinds of turns in it** — not two independent conversations that happen to sound similar. that means the underlying memory store needs to be keyed by the person (or contact) being talked to, not by an individual call session or an individual text thread.

this has a real, visible consequence: if someone says mid-call "yeah, i just sent you that on whatsapp," the agent should either already know what "that" is (because the text got processed into shared memory when it arrived) or be able to go check right now (a live lookup into the same shared memory store) — either way, it should not respond as if it's hearing about the message for the first time, the way it would if the call and the text thread were two disconnected systems.

### two different architectures for getting there

there are two real ways to make this work, and they're meaningfully different:

**continuous ingestion** - every text that arrives gets processed into the shared memory graph immediately, regardless of whether a call is happening at that moment. by the time a call starts, anything sent by text earlier is already sitting in memory, ready to be referenced. this means "remembering" during a call is just a normal read from memory (tentatively mem0 use) nothing special has to happen at call time.

**on-demand cross-channel lookup** - the agent doesn't pre-ingest anything; instead, during a call, if a text-channel reference comes up, the agent performs a live lookup across channels at that moment (falkordb) the same way it might pause to check its own memory of the call in progress. this is architecturally simpler (no background ingestion pipeline needed) but means the "let me check" moment happens live, in front of whoever's on the call, rather than being invisible because the information was already there.

these aren't mutually exclusive. a real system likely wants continuous ingestion as the default (so memory doesn't visibly lag behind reality) with on-demand lookup as a fallback for anything that hasn't been ingested yet or needs a fresher check. but they're worth deciding deliberately rather than defaulting into one by accident, since they produce different live behavior.

### the honesty rule extends to lookups themselves

when a real lookup is happening, whether it's checking the shared memory graph (we're planning for falkordb), or reaching out to confirm something across channels — the agent is allowed to sound like it's doing that: a brief pause, filled with something that sounds like the target's own natural filler words (not a generic "please hold"), because there's a real wait happening and covering it with dead air would be worse, not more honest.

what's not allowed is faking that pause when no real lookup is happening, inserting a "let me check" moment purely for effect, with nothing actually being checked. that would mean performing a fake tell about doing ai work that isn't actually occurring, which works directly against the disclosure principle from `research.md` rather than supporting it. the boundary is simple: **a real wait can be covered. a fake wait cannot be manufactured.**

---

## what regent is not trying to be

- **not a way to claim presence on a channel that doesn't actually exist for the target.** the same "absent is a correct answer, not a gap to fill" principle from yaptele's medium coverage applies directly here. so if there's no real way to reach the agent on a given channel yet (imessage, currently), that's stated plainly, not implied to exist.
- **not a loophole in the disclosure rule.** "this is a reconstruction, not the real thing" is not a call-only obligation. it applies on every channel the agent is reachable through a text conversation deserves the same honesty a phone call gets.
- **not a general-purpose messaging bot.** the interesting part isn't "an ai that can text you" bcs  that exists already, broadly. the interesting part is a single identity that stays consistent and remembers itself across every way of reaching it, which is a continuity problem, not a channel-integration problem.

---

## summary

regent takes one thing (a voice profile from yaptele) and gives it three things it wouldn't otherwise have: a real, persistent point of contact (the phone number with calls, texts, whatsapp, and voicemail all included, not just whichever channel is easiest to answer), the ability to render that one profile appropriately across different channels instead of needing a separate profile per channel, and the part that actually matters aka a shared memory that makes those channels feel like one ongoing relationship instead of several disconnected tools that happen to sound alike.
