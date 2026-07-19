---
name: Sai-ai-video-drama
description: Use this skill whenever the user wants to plan, script, or generate an AI-generated video drama, short film, or multi-scene narrative video — especially TikTok-style vertical video built from reference images and a reference-to-video model (e.g. Seedance, or similar image-to-video / reference-to-video APIs). Trigger on requests to turn a script or concept into a "production bible," build character sheets or location reference images for consistency across scenes, orchestrate multi-clip AI video generation pipelines, or assemble/QA a sequence of AI-generated video clips into a finished video. Also trigger on mentions of "reference-to-video," "character sheet consistency," "scene continuity," or planning a multi-scene AI video project, even if the user doesn't use the word "skill" or name a specific tool.
---

# AI Video Drama Generation Workflow

An end-to-end pipeline for turning a script/concept into a finished AI-generated video: production bible → reference images → per-scene character variants → clip generation → assembly → music → export.

Follow the steps below in order. Each step lists its checkpoints — do not skip a checkpoint even if the user seems eager to move fast; bad assets compound expensively downstream.

## Pre-Production: Confirm Settings

Before Step 1, confirm with the user (ask if not already provided):
- **Generation provider & API key (ask first)** — by default ask which provider to use for image/video/music generation: **FAL.AI** or **Atlas Cloud** (both host the same underlying models — Seedance 2.0, GPT Image 2, MiniMax Music 2.6). Then obtain the provider's **API key securely**: request it via `requestApproval({ type: 'user-input' })` with an `inputType: 'password'` field, store it as a session **environment variable**, and reference it only by env var in shell/API calls. **Never** ask for the key in plain chat, never echo it back, never write it to a file or artifact, and never paste it into prompts, logs, or the conversation. If the user already supplied a provider/key earlier in the project, reuse it and don't re-ask.
- **Resolution** (e.g. 720p)
- **Aspect ratio** (e.g. 9:16 vertical for TikTok-style)
- **Expected total runtime** (e.g. 2 min)
- **Frame rate** (e.g. 24fps)
- **Background music style/mood** (genre, tempo, reference tracks) — captured now so Step 6 doesn't need a second round-trip
- **Script or concept** — if the user hasn't provided one, ask before proceeding

### Models & reference docs

The pipeline uses three models, hosted on **both** FAL.AI and Atlas Cloud — use the endpoints matching the provider the user chose above. **Read the relevant model page before first use** to confirm the current request/response schema, parameters, and exact slug (these APIs change):

| Role | Model | FAL.AI | Atlas Cloud |
|---|---|---|---|
| Reference-to-video (clips) | ByteDance **Seedance 2.0** (reference-to-video) | https://fal.ai/explore/seedance-2.0 | https://www.atlascloud.ai/sv/models/bytedance/seedance-2.0/reference-to-video |
| Image generation (references) | **GPT Image 2** (`text-to-image` / `edit`) | https://fal.ai/models/openai/gpt-image-2 | https://www.atlascloud.ai/sv/models/gpt-image-2 |
| Background music (Step 6) | **MiniMax Music 2.6** | https://fal.ai/models/fal-ai/minimax-music/v2.6 | https://www.atlascloud.ai/sv/models/minimax/music-2.6 |

Provider-specific notes:
- **Atlas Cloud** — image: `POST /api/v1/model/generateImage` → poll `GET /api/v1/model/prediction/{id}` for the hosted output URL. Video: `POST /api/v1/model/uploadMedia` (get hosted URL) → `POST /api/v1/model/generateVideo` → poll the same prediction endpoint. GPT Image 2 requires the **task-suffixed slug** (`openai/gpt-image-2/text-to-image` or `openai/gpt-image-2/edit`) — the bare slug fails. Confirm base URL and exact paths against the model pages above.
- **FAL.AI** — use the `fal-ai/...` / `openai/...` model slugs from the pages above; upload reference images to fal's CDN and pass **hosted URLs** (see Step 4's base64 warning).
- Whichever provider is chosen, keep it **locked for the whole production** — don't switch providers or models mid-project (see the locked-settings rule below), since that shifts the look.

## Global Rules (apply throughout)

**Scene & clip structure**
- A *clip* is the smallest unit sent to the video model. **~15 seconds is the recommended length for most clips** — long enough to carry real content and conversation — but it's a guideline, not a fixed rule: some beats, especially **opening and closing shots or brief reaction moments, are better shorter**. Size each clip to its content. When a clip does run to full length, fill it with enough content to justify it — multiple beats/shots, ongoing action, substantial dialogue — never a single thin moment stretched out (see Step 1).
- A *scene* may contain multiple clips, but must take place in exactly one location. All clips within a scene share that same location reference **and** the same scene-specific character reference(s) for every character appearing in it — never mix reference versions across clips in one scene.
- Locations may repeat across scenes — reuse the same reference image rather than regenerating it.

**Audio**
- There are **two** sources of unwanted background music, and both must be prevented during clip generation (Steps 1–4):
  - **(a) The operator adds a track** — never do this before Step 6. Music is added once, at the end, in Step 6.
  - **(b) The video model generates its own score** — reference-to-video models with audio generation enabled will often synthesize a full soundtrack (dialogue + ambient + SFX **and** a background musical score of their own), especially when the prompt uses cinematic/emotional wording ("cinematic," "emotional," "sweeping," "orchestral mood"). To prevent this, **every clip's video-generation prompt must explicitly instruct the model NOT to generate any background music or musical score** — only diegetic dialogue, ambient/room tone, and natural SFX. Be aware that cinematic/emotional prompt language nudges the model toward adding a score, so pair such wording with an explicit no-music instruction (see Step 4).
- Do include natural sound effects, ambient sound, and accurately lip-synced dialogue/vocalizations per clip.
- Every character needs a voice profile: casting reference, tone, delivery style, vocal characteristics. Voice profiles are authored as a structured artifact in Step 1 (`voice-profiles.json`) and must be embedded into every clip's video-generation prompt for each speaking character (see Step 1 and Step 4).
- **Clip-level "music-free" ≠ film-level "music-free."** Each clip carries no music, but the final assembled film gets exactly one music track, added once in Step 6. Removing music from the clips is *because* it's added properly at the end — not instead of it. Never conflate "no music per clip" with "no music in the final video," and never skip Step 6.

**Visual style — cinematic film drama**
- The target look is an authentic **cinematic film drama** — real movie-scene quality, not "AI-pretty." Bake this into every image-generation and video-generation prompt using the concrete recipe below — vague instructions like "make it cinematic" or "make it look realistic" are not enough and reliably drift back toward the glossy, plastic, over-polished look that reads as obviously AI-generated.
  - **Real camera / lens / film-stock language** in the prompt itself — name a specific camera, lens, and film stock (e.g. "shot on ARRI Alexa, 40mm anamorphic lens, Kodak Vision3 stock") with subtle **film grain**. This grounds the model in photographic reality instead of a generic render.
  - **Motivated, naturalistic lighting** — describe light like a cinematographer: key / fill / practical sources with real motivation (window, desk lamp, monitor glow), soft falloff and believable shadow — not generic "dramatic" or "epic AI glow."
  - **Photographic realism cues** — visible skin texture and pores, natural facial asymmetry, natural motion blur, and subtle lens character (slight vignette, faint chromatic aberration). These are what separate a real lens from an AI render.
  - **Keep backgrounds clean and in focus (default) — avoid heavy background blur.** In BOTH image generation and video generation, do NOT default to a strong shallow depth-of-field / bokeh look that melts the background into a blur. In most clips the environment should read clearly and stay legible — the location references were built precisely so the setting is visible and consistent, and a blurred-out background wastes that work and hides continuity. Prefer a **deeper depth of field with the background in acceptable focus.** Add a negative instruction to every image and video prompt: *no heavy background blur, no strong bokeh, no melted/mushy background — keep the setting clean, sharp, and readable.* Reserve shallow focus for the **few** shots that genuinely need it (e.g. an intimate close-up where isolating the subject is the point), never as the default across clips.
  - **Real color-grade references** — name an actual filmic grade/look (e.g. "teal-and-amber cinematic grade," "muted naturalistic grade," "warm film-print look") rather than just "cinematic."
  - **Naturalistic performance** — micro-expressions, natural blinks, believable weight and momentum in movement; avoid stiff, floating, or unnaturally smooth motion.
  - **Camera movement — choose per beat, and keep it restrained.** Decide each clip's camera independently: either a **deliberate locked-off (static) shot** when stillness serves the moment (a held emotional pause, a composed frame), or **a single slow, motivated move** (a gentle push-in to intensify, a slow dolly to reveal, subtle handheld for intimacy) chosen to serve that beat. Do NOT default every clip to motion — a film that moves in every shot feels restless and unintentional; aim for a mix where stillness and movement each earn their place. **Do not overuse the far-to-near push-in / dolly-in / zoom-in specifically.** It is fine *occasionally*, when a beat genuinely needs to intensify or reveal — but it must NOT be the default move applied to clip after clip. Repeating a distance-crunching push-in across most clips both looks monotonous AND compounds morphing/warping artifacts, because the model has to re-invent the subject as its on-screen scale grows. When a push-in isn't clearly earned, instead pick the framing up front (already close for a close-up, already wide for a wide) and either hold it locked-off or use a small move at a roughly constant distance (slow lateral drift, gentle handheld). Reserve the push-in for the few beats that truly call for it. **Never stack multiple moves in one clip** (e.g. pan + push-in + tilt) and never use fast, erratic, or continuous motion. Excess and stacked camera movement also amplifies morphing/warping artifacts, which reads as more AI, not less.
  - **Prefer higher fidelity WITH film grain** — for cinematic drama, do NOT drop resolution to hide the AI look; render at full fidelity and let real film grain plus photographic imperfections defeat the digital-AI tell. Grain, not softness, is the lever here.
  - **Explicitly avoid the AI tells** — add a negative instruction to every image and video prompt: no waxy / plastic / over-smooth skin, no poreless faces, no uncanny symmetry, no over-saturation or over-polishing, no floaty / morphing / too-smooth motion, and **no heavy background blur / strong bokeh (keep the background clean and in focus).**
  - **No background music under any clip** (see Audio rules) — score is added only in Step 6.
- Location reference images are wide/panoramic purely so the video model understands what's off-frame — this is a spatial reference, not a literal frame, and doesn't conflict with a 9:16 final output.

**Visual coherence across clips — lock the look, don't re-describe it**
- Separately generated clips drift apart mainly because each clip *re-describes* the intended look in slightly different words — eleven independently-worded descriptions of "cinematic grade" yield eleven different grades. Make consistency come from **identical tokens**, not from re-expressing the same intent.
- **Author one canonical "Look Bible" (a locked visual-style block) in Step 1** — an exact, reusable string covering: camera + lens set, film stock, named color grade/LUT, contrast & black levels, film-grain amount, lighting philosophy, and a **defined color palette (specific named hues / hex)**. Write it once, get it approved by the user, then **paste it verbatim — character-for-character — into every image-generation and video-generation prompt.** Do NOT paraphrase, summarize, reorder, or "improve" it per clip; the whole point is that every prompt carries the *same words*. This is how the "prompt self-containment" rule's lighting/grade/lens items must be satisfied — by pasting the locked block, not by rewriting it each time.
- **Lock generation settings for the entire production.** Choose the provider, image model, video model, resolution, aspect ratio, and frame rate once and reuse them for every clip — and use a **fixed/consistent seed strategy** wherever the model supports it. Changing any of these mid-project silently shifts the look and breaks coherence.
- **Establish a master "look anchor."** The first approved hero image defines the exact grade and palette; where the model supports an image style/reference input, pass it on subsequent generations so the look anchors on an image, not only on text.

**Clip continuity**
- Never rely on one clip's final frame as the next clip's starting frame — each clip must be independently generatable.
- Write shots/actions so they don't require exact frame-to-frame continuity between separately generated clips.

**Lead with image generation for in-frame detail**
- Reference-to-video models render text, logos, screen content, and precise props poorly — garbled text, fake-looking logos, mushy or wrong UI are among the biggest authenticity tells. **Do not leave these to the video model to invent.** Anything in frame that requires authentic, precise detail — **brand assets (logos, wordmarks), on-screen app/product UI (phone & computer screens), text on computers/phones, documents, signage, packaging, hero props** — must be created as an **image first** (generated or composited) and passed to the video model as a **reference**, not described and hoped for. Two failure modes to avoid: passing a **fragment** (e.g. a bare logo) makes the model invent — and garble — the rest around it (UI, text, layout); and baking the asset into a **pre-composed dramatic shot** (character + hand + action + camera angle) and animating that makes the model warp/morph what it can't physically reason about. So build the **complete, finished, isolated** asset and let the video model only place and animate it (see Step 2 for how).
- **Real brand assets are composited as-is, never AI-generated.** If the user supplies a real logo or an actual app/product screenshot, use it directly (place it on a screen, a wall, a device via image compositing/editing) — never let the model regenerate or "reinterpret" it. AI-generated brand marks and UIs read as fake and are often legally/factually wrong.
- **Bias toward generating MORE reference images, not fewer.** Extra angles, states, and detail images (a product in several UI states, a prop from two angles, a screen mid-interaction) give the video model more to anchor on and measurably reduce drift. When in doubt, make another reference image rather than asking the video prompt to carry the load alone.

**Prompt self-containment & detail**
- Each clip's image-generation prompt and video-generation prompt is consumed by the model **in isolation** — the model never sees the production bible, the other clips, or even which scene this clip belongs to. A thin, shorthand prompt (e.g. "Michael looks tired at his desk") gives the model nothing to anchor on, so it drifts, invents details, or produces generic output. Under-specified prompts are a primary cause of model confusion and inconsistency across clips.
- Therefore **every** clip prompt must independently restate the full context it needs, even when that context is "obvious" from the scene or repeated verbatim across sibling clips. At minimum, each prompt must specify: **character appearance/wardrobe** (who is in frame and exactly how they look/are dressed), **location & spatial context** (where this is, and what's around/off-frame), **lighting/mood/grade** (time of day, light sources, color grade, emotional tone), **camera & lens** (shot size, angle, lens feel, movement), **blocking/action** (what physically happens, beat by beat), **dialogue + delivery** (exact lines and how they're spoken), and **SFX/ambient** (foley and background sound). Never rely on the bible or neighboring clips to "carry over" any of these — re-embed them in-prompt every time.

**Asset naming** — use this structure so scene/costume/clip variants stay organized at scale:
```
/production-bible.md
/references/look-bible.md
/references/look-anchor.png
/references/locations/{location_name}.png
/references/characters/general/{character_name}.png
/references/characters/scene-variants/{scene_id}/{character_name}.png
/references/characters/voice-profiles.json
/references/assets/brand/{asset_name}.png
/references/assets/screens/{screen_name}.png
/references/assets/props/{prop_name}.png
/clips/{scene_id}/clip-{n}.mp4
/audio/music/final-track.mp3
/output/final-video.mp4
```
Location images are keyed **per unique location**, not per scene.

## Step 1 — Rewrite the Script into a Production Bible

Expand the user's script/concept into a full production bible containing: character and location bibles, visual style guide (including the locked **Look Bible** — see below), shot-by-shot storyboard with timecodes, dialogue and performance direction, image-generation and video-generation prompts per clip, voice profiles, text overlays, master timeline, production checklist, and export requirements.

**Adapt to the input — detailed script vs. simple prompt (two situations, handled oppositely).**
- **The user wrote a detailed script.** Their script is the source of truth. Your job is to *enrich* it into fully-specified image/video prompts — adding the visual, camera, lighting, blocking, and delivery detail the models need — **without changing the user's intention**. Preserve their story beats, plot, character choices, dialogue meaning, tone, and ending exactly as written. Add craft detail *around* their content; do NOT rewrite lines into different meanings, invent new plot turns, cut or reorder scenes, or change how it resolves. If a beat is genuinely ambiguous and you must fill a gap, resolve it in the direction that best serves *their* stated intent — and flag any material interpretation at the checkpoint for confirmation rather than silently deciding.
- **The user gave only a simple prompt / logline.** Now you are the writer: invent the characters, the specific conflict, the scene-by-scene beats, the dialogue, and the resolution, then specify every clip prompt in full. The burden is on you to add sufficient detail at *both* levels — a rich production bible **and** fully-specified per-clip prompts — so nothing is left thin for the models to guess. Present the invented story at the checkpoint so the user can course-correct before any assets are generated.
- In **both** cases the per-clip prompts must end up fully specified (see the detail checklist below); the only difference is *who authors the story* — preserve it faithfully when the user gave detail, create it richly when they didn't.

**This is a *drama* — deliberately engineer emotion and conflict, fast.** A short drama lives or dies on making the audience *feel* something quickly; the production bible must engineer this on purpose, never leave it to chance. Apply ALL of the following concrete rules when building the beats, dialogue, and performance direction:

- **Start with conflict immediately.** Introduce the main problem within the first few seconds. Do NOT spend the opening on background, world-building, or slow character introductions — drop the audience straight into the tension. The very first clip should already show the problem, not set up to it.
- **Give the protagonist one clear goal.** The audience should quickly understand what the main character *wants* and exactly what is *stopping them* from getting it. One goal, one obstacle — legible within the first beat, not a diffuse mix of motivations.
- **Make the audience take sides early.** Use a strong action, line, or reaction to make viewers immediately like, dislike, support, fear, or feel sorry for a character. Engineer a specific "take a side" moment up front (a sympathetic underdog to root for, an antagonist to resent) — don't wait for it to emerge.
- **Include at least one genuinely unlikable character (the "asshole").** Every drama needs someone the audience can actively resent. At least one character MUST behave in a clearly *unacceptable, unreasonable, or unfair* way — belittling, dismissive, arrogant, smug, cruel, petty, or entitled — not merely holding "an opposing viewpoint" or being a mild skeptic. Make the bad behavior concrete and **on-screen** (a cutting insult, a public humiliation, taking credit for someone's work, an unfair decision) and let it land hard enough that viewers genuinely dislike them. This friction is what gives the protagonist someone to overcome and makes the payoff satisfying. Keep it *believable*, not cartoonish — a real jerk, not a moustache-twirling villain — but do NOT soften them into someone reasonable; the antagonism must be real and felt. Reinforce it across multiple beats, and (if the story calls for it) let them get their comeuppance or be shown up at the payoff.
- **Continuously reinforce each character's emotional role.** Do not rely on a single introductory moment. Keep using dialogue, behavior, reactions, and decisions throughout the story to strengthen how the audience should feel about each character — the underdog stays sympathetic, the antagonist keeps earning resentment, beat after beat.
- **Keep the emotional arc simple.** Use this structure explicitly: **Struggle → Pressure → Setback → Turning Point → Transformation → Payoff.** Map the scenes/clips onto these stages so the arc is unmistakable; don't blur or overcomplicate it.
- **Raise the stakes clearly.** Give the protagonist something meaningful to lose — job, reputation, relationship, opportunity, dignity, or confidence. Name the stake early and make its threat concrete and escalating, so the audience feels what a loss would cost.
- **Keep the number of characters limited.** For a short drama, **two to four important characters** are usually enough. Every character must have a clear, distinct purpose — cut anyone who doesn't change the story.
- **Make every scene change the situation.** Each scene must do at least one of: increase the pressure, reveal new information, create a setback, or move the protagonist closer to the ending. No scene may leave the situation where it started — if a scene doesn't change things, rewrite or cut it.
- **Create a strong turning point.** At the pivot, the protagonist makes a decision, discovers something, or takes an action that clearly changes the direction of the story. Make this moment sharp and unmistakable — it's the spine of the arc.
- **End with a satisfying payoff.** The ending must deliver real emotional satisfaction — surprise, irony, justice, revenge, redemption, or transformation. Pay off the goal and stakes established at the start; don't let it fizzle or resolve limply.

When **expanding a simple prompt**, *invent* the conflict, the goal/obstacle, the love/hate dynamics, the stakes, and the turning point deliberately, using the rules above. When **enriching a detailed script**, *heighten and sharpen* the emotional beats the user already wrote to satisfy these rules — without changing their story, intent, or ending.

**Drama self-check before the checkpoint.** After drafting the beats, verify the bible against every rule above, in order: Does the first clip open on the problem? Is the protagonist's single goal + obstacle clear immediately? Is there an explicit early "take a side" moment? Is there at least one genuinely unlikable character who behaves unacceptably/unreasonably **on-screen** (a real jerk, not just a mild skeptic)? Is each character's emotional role reinforced repeatedly, not just once? Do the scenes map cleanly onto Struggle → Pressure → Setback → Turning Point → Transformation → Payoff? Are the stakes concrete and escalating? Are there only 2–4 purposeful characters? Does every scene change the situation? Is there one unmistakable turning point? Is the ending a genuine payoff? Fix any rule that fails before presenting to the user.

Organize the script into scenes, clips, and shots per the Global Rules above. Each scene may contain multiple clips; all clips within a scene must share one location; most clips should run approximately **15 seconds** — a guideline, not a hard rule (opening/closing shots and brief reaction beats can be shorter; size each clip to its content) — may contain multiple shots, and **must be independently generatable without relying on the final frame of the previous clip** — design the shots and actions with this constraint in mind now, while writing the script, not as an afterthought to fix later.

When rewriting the script, **expand rather than compress** — write out the concrete visual and performance detail explicitly; never leave it implied or assumed. If a beat says "they argue," spell out the wardrobe, the room, the light, the blocking, the exact lines, and the delivery. The bible's job is to make every downstream prompt richly specified, not terse.

**Author the locked "Look Bible" (canonical visual-style block).** As part of the bible, write a single, exact, reusable **Look Bible** string that will be pasted verbatim into every image and video prompt (see "Visual coherence across clips" in the Global Rules). It must pin down, in fixed wording: camera + lens set, film stock, named color grade/LUT, contrast & black levels, film-grain amount, lighting philosophy, and a **specific color palette (named hues / hex values)**. Persist it as `references/look-bible.md` and present it to the user as its own item at the checkpoint — this string is the single biggest lever for cross-clip coherence, so it must be locked before any image is generated. Also record the **locked generation settings** here (provider, image model, video model, resolution, aspect ratio, frame rate, and seed strategy) so every downstream step reuses exactly the same configuration.

**Fill every ~15-second clip with real content and conversation.** Fifteen seconds is a long shot — each clip must carry enough to sustain it: multiple beats, continuing action, and **substantial dialogue/conversation** so the viewer always understands what's happening and why. Do NOT stretch a single thin moment ("he sits down, looks tired") across 15 seconds — that reads as empty and slow. Every clip should advance the story and include enough spoken lines — real exchanges between characters, or meaningful delivery — to give the audience context; a near-silent clip is acceptable only when silence is a deliberate dramatic choice for that specific beat. When you divide scenes into clips, budget the dialogue and action so each clip has its own beginning-middle-end worth of content, not a fragment padded out to length.

**Land dialogue inside the clip — never cut on a mid-sentence line (invisible clip boundaries).** A clip that ends while a character is still mid-sentence produces a chopped word and an obvious, jarring cut when it joins the next clip. Time each clip's dialogue so **all spoken lines complete before the clip ends**, then close the clip on a brief **non-speaking beat** — a look, a breath, a gesture, a reaction — so the boundary falls on silence/action, not inside a word. Likewise, **don't open a clip mid-sentence**; start on action or a line's beginning. In short, place clip boundaries in the pauses *between* lines, not inside them — this keeps cuts landing on natural beats and makes the seams between separately generated clips effectively invisible. Design this into the clip division now (it also reinforces the rule that each clip must be independently generatable without frame-to-frame continuity).

**Plan transitions per boundary (mostly clean cuts).** In the master timeline, mark how each clip joins the next — don't leave it to "add transitions" later. Default to **clean hard cuts between clips *within* a scene** (they share one location and use the invisible-boundary landing above, so a fade there only makes continuous action feel choppy). Reserve a **deliberate transition for scene boundaries** — especially the **last clip of a scene into the first clip of the next scene**, where location/time changes — typically a **fade out / fade in (dip to black)** or a gentle **cross-dissolve** (a short fade to black suits a larger time jump). Do NOT schedule a transition on every clip; overusing fades reads as amateur. Record the chosen transition (or "hard cut") for each boundary here so Step 5 executes exactly this plan.

Per clip, include: duration, camera/lens, framing, dialogue, voice direction, facial expressions, image-generation prompt, I2V motion prompt, and foley/SFX notes. The **image-generation prompt** and **video-generation prompt** for each clip are the assets that get sent to the models in isolation (see "Prompt self-containment & detail" in the Global Rules), so each one must be a **mandatory detail checklist**, independently satisfying ALL of the following — no item omitted, even if it repeats what a sibling clip already stated:
- **Character appearance/wardrobe** — every character in frame, named, with their exact look and clothing for this clip.
- **Location & spatial context** — where this takes place and what surrounds it / sits off-frame.
- **Lighting/mood/grade** — time of day, light sources, color grade, emotional tone.
- **Camera & lens** — shot size, angle, lens character, and camera movement. State **either** a deliberate locked-off/static hold **or** the single slow, motivated move for this clip (and its pace) — never a list of several moves. Choose static vs. move per beat; don't make every clip move, and **don't default to a far-to-near push-in/zoom on most clips** — reserve it for the few beats that genuinely need to intensify or reveal (see the camera-movement rule in the Global Rules).
- **Blocking/action** — the physical action beat by beat.
- **Dialogue + delivery** — the exact lines and how they're performed (for video prompts).
- **SFX/ambient** — foley and background sound (for video prompts).

**Re-embed shared scene context into every clip's prompt.** Context that is constant across a scene — location, wardrobe, character look, time-of-day, color grade — must be written out in full inside *each* clip's prompt, NOT stated once at the scene level and assumed for the rest. Each clip is generated independently, so "same as previous clip" is invisible to the model.

**Per-prompt quality bar:** each image-generation and video-generation prompt should read as a **fully-specified paragraph**, not a one-line summary. If a prompt is a single terse sentence, it is under-specified — enrich it until every checklist item above is explicitly present.

**Voice profiles (mandatory, structured).** Author a voice profile for **every** character and persist them as a structured file `references/characters/voice-profiles.json`, keyed by character name, so each profile is reusable and can be injected verbatim into clip prompts downstream. Prose scattered through the bible is not enough — it must be this machine-readable object. Each character entry must define at minimum: `character`, `casting_reference`, `gender_age_read`, `pitch_register`, `accent`, `pace_cadence`, `tone`, `delivery_style`, `vocal_texture`, `emotional_range`, and `sample_line_reads`. Example:
```json
{
  "Michael": {
    "casting_reference": "weary mid-40s middle manager, think a subdued Bryan Cranston",
    "gender_age_read": "male, 40s",
    "pitch_register": "medium-low",
    "accent": "neutral American",
    "pace_cadence": "slow, halting when tired; steadier when confident",
    "tone": "tired and strained early, warm and assured by the end",
    "delivery_style": "understated, few words, lets pauses land",
    "vocal_texture": "slightly gravelly, breathy on exhales",
    "emotional_range": "exhaustion → quiet hope → confidence",
    "sample_line_reads": ["'Three days.' — flat, defeated", "'Reviewed, and ready.' — calm, self-assured"]
  }
}
```

**Pass voice profiles along to every clip.** For each clip, embed the full voice profile of **every speaking character in that clip** — pulled verbatim from `voice-profiles.json` — directly into the clip's video-generation prompt (this is the voice half of the "Dialogue + delivery" checklist item above). Consistent with prompt self-containment, re-state the profile in-prompt every time the character speaks; never assume the model remembers a voice from a previous clip.

List every character who appears in the script and note which scenes each one appears in — this is what Step 3 uses to know which scene-specific variants to generate for each character.

Do not write music-generation prompts here — that's Step 6 only.

Sanity-check: does the sum of clip durations (most ~15s, some shorter for openers/closers/reaction beats) roughly equal the confirmed Expected Total Runtime? Flag any mismatch.

**Story-clarity self-reflection (do this before presenting the bible).** The audience only ever experiences what is *inside the clips* — the visible on-screen action, the spoken dialogue, and any on-screen text. They never read the bible's scene descriptions or director's notes. So before the checkpoint, read through **only the per-clip content — the dialogue and the visible action — in sequence**, deliberately ignoring the surrounding scene prose and the camera/grade/lens notes, and ask honestly: *does the storyline come through clearly from the clips alone?* If any beat is unclear — a character acts with no established motivation, the story jumps without explanation, a character appears without being introduced, or a plot point is only understandable from prose the viewer will never see — the clips are under-specified. Revise those clips (add clarifying dialogue, an establishing beat, or visible action) until the clips-only read is coherent on its own. This is the concrete test of the "prompt self-containment" and "enough dialogue/context per clip" rules; passing it is what ensures the finished video actually tells the story rather than relying on context only you can see.

**Checkpoint**: present the production bible to the user for confirmation before Step 2.

## Step 2 — Generate Master Reference Images

*(Use GPT Image 2, or the user's preferred image-generation model)*

- **Locations**: a single **multi-angle location composite sheet per unique location** (not per scene) — one image showing the same environment from several viewpoints (a wide establishing view **plus 2–4 additional angles** of the same space, e.g. from the doorway, the window side, the opposite corner). This is a spatial "turnaround" for the environment, the location equivalent of a character sheet: it gives the video model multi-view spatial information so the room stays consistent (furniture placement, wall colors, window positions, light direction) across clips framed from different angles — a single panorama only covers one viewpoint and forces the model to invent the unseen sides, which is where environments drift.
  - **Generate the sheet reference-linked**, not all-at-once-blind: create the establishing angle first, then generate the other angles using it as a reference/edit input, so every panel is genuinely the *same* room (matching layout, palette, props). A sheet whose panels disagree with each other injects inconsistency instead of fixing it.
  - **Use it as ONE reference slot per clip.** A composite keeps the location to a single reference image, preserving the model's limited reference slots (e.g. Seedance ~9) for the clip's character(s), props, and brand/screen assets. In the clip prompt, always declare its role (see Step 4) so the model treats it as a reference, not a layout to reproduce.
  - **Validate on the first clip, keep a fallback.** On the first clip generated for a location, confirm the model reads the sheet as spatial reference and renders a single continuous scene. If instead the output shows panels/split-screen or gets confused, fall back to **separate per-angle images** for that location (each angle its own image, attaching the angle closest to the clip's framing). Note this in the bible so the fallback isn't a surprise.
  - Map each scene to its location composite (locations may repeat across scenes — reuse the same sheet). Note: multi-view references sharply reduce environmental drift but don't give the model a true 3D model of the space — expect much better consistency, not architectural perfection.
- **Characters**: a 5-angle character sheet for every character in the script — front, three-quarter front, side profile, three-quarter back, back. No exceptions or simplified variants — every character gets the same full treatment.

**Defeat the reference-to-video face filter now, at sheet-generation time (realistic faces get blocked downstream).** Seedance 2.0 (and similar reference-to-video models) run a partner face-validation filter that **rejects the whole clip job** in Step 4 (`content_policy_violation` / `partner_validation_failed` / "may contain likenesses of real people") whenever a reference image contains a photorealistic human face. The filter cannot tell an AI-generated face from a real photo, so **the character sheets you generate here are exactly what trigger it** — every face-bearing sheet is at risk. A visible "AI-generated / synthetic" watermark does **not** help (the filter keys off the face itself, not any label or metadata). Prepare the workaround as part of generating the sheet, so a clean *and* a gridded version exist before Step 4:
1. After generating each character sheet, overlay a **dense, strong geometric grid** on it — the **proven default recipe is 8px cells, pure black at 0.65 opacity**: ffmpeg `drawgrid=width=8:height=8:thickness=1:color=black@0.65` — and save it alongside the clean sheet (e.g. `{character_name}-grid.png`). This *gridded* version is what gets uploaded as the reference in Step 4. **Do NOT start with a faint/light grid.** Light grids (15px @ 12%, 12px @ 16%) were tested repeatedly and reliably FAILED the filter — starting light just wastes attempts and credits' worth of time. Begin at the strong 8px / black / 0.65 recipe on the very first attempt.
2. This dense grid breaks the classifier's contiguous-real-face signal — the small 8px cells fragment the face so no region reads as a continuous photograph, while enough structure survives for identity lock — and the video model reconstructs clean skin beneath it — **no grid lines appear in the output** — provided the Step 4 clip prompt tells it to ignore the grid ("the reference images contain an alignment grid — ignore it, do not render any grid lines, reconstruct clean high-fidelity photorealistic skin beneath it").

**The filter is stochastic and aggregate across all reference images in a clip** — the more face sheets attached to one clip, the higher the block chance (and identical references can pass on one clip, fail on another). **Start strong, and if the first round is blocked, escalate HARDER — never lighter.** Do not creep up from a faint grid; that path was tested and fails. The escalation ladder (top is the default first attempt):
1. **Strong grid — DEFAULT first attempt: 8px cells, pure black @ 0.65 opacity.** This is the proven recipe; begin here, never warm up with anything lighter.
2. **Denser/darker grid** — if a clip is still blocked, push HARDER: smaller cells (8px → 6px) and/or higher opacity (0.65 → 0.7 → 0.75). Escalate toward more grid, never less.
3. **Fewer face references per clip** (in Step 4) — drop redundant/secondary character sheets and let those characters render from the text prompt instead. Keep **≤2 faces per clip** as a rule; ideally 1.
4. **Single face sheet** — the reliable fallback that almost always passes.

**Filter-probe first (cheap):** before batching, send one shortest-duration test per unique gridded reference to confirm it passes — failing fast on a 2s probe is far cheaper than discovering a block mid-batch. Blocked jobs cost **no output credits**, so escalating/retrying is free — there is no reason to be timid.

**Tradeoff:** a denser grid degrades reference fidelity slightly (the model hallucinates more skin underneath), which can add minor warping. That is an acceptable price — **prioritize passing the filter over preserving maximum fidelity.** Only if a specific hero shot genuinely needs more fidelity may you test a lighter grid **on that one shot**; the pipeline default is always the strong recipe first.
- **In-frame detail assets**: image-generate or composite a reference for every element that needs authentic, precise detail and that the video model would otherwise render poorly (see "Lead with image generation for in-frame detail" in the Global Rules) — **brand assets** (logos, wordmarks), **screen/product content** (phone & computer app UIs, in-world text, dashboards), **key props**, and **hero graphics**. Real supplied brand assets and actual app/product screenshots are used/composited **as-is, never AI-generated**. Build each as a **complete, finished, isolated asset**: decide per shot what in-frame element is actually needed and create that *whole* element (the full app/phone UI, the entire document, the complete label or dashboard) — not a fragment like a bare logo, and **never** pre-composed into a dramatic shot (character + hand + action + camera angle). The video model garbles anything it has to invent around a fragment, and warps anything it must animate from a complex composed frame — so hand it the finished, isolated asset and let it only place and animate it. Generate **multiple variants where useful** — e.g. an app screen in several states (idle, mid-task, approval prompt, result), a prop from more than one angle — so downstream clips have a precise reference for each moment rather than relying on the video prompt alone. Scan the production bible for every clip whose prompt mentions a logo, a screen, readable text, a product, or a specific prop, and make sure each has a reference image here.

## Step 3 — Generate Scene-Specific Character Variants

*(Same image model as Step 2)*

For each character in each scene they appear in, generate a scene-specific variant of their reference sheet, reflecting any costume/styling/appearance changes while keeping the core character design consistent.

**Checkpoint**: present all location + character reference images (general and scene-specific) to the user for approval before Step 4. Catching a bad reference here is far cheaper than discovering it after clips are generated from it.

## Step 4 — Generate Video Clips

*(Use Seedance, or the user's preferred reference-to-video model/API)*

### Critical: image inputs must be hosted URLs, never base64

**Never pass base64-encoded images as reference inputs to the reference-to-video call.** This is the #1 failure mode: base64 inputs cause videos to morph to black or produce random/garbage output partway through the clip. Always upload reference images (character sheets, location images) to fal's CDN (or the equivalent hosted storage for whichever platform is in use) first, and pass the resulting hosted URL to the API — not the raw image bytes. Before generating a single clip, confirm the upload step is wired up and producing real URLs; don't discover this the hard way on the first clip.

### Face filter blocks character sheets — apply the grid overlay from Step 2

Upload the **gridded** character sheet from Step 2 (never the clean one) as the reference from the **first** attempt — the strong 8px / black / 0.65 grid is the default, not something you reach for only after a block. On the prompt side, add the instruction that pairs with it: "the reference images contain an alignment grid — ignore it, do not render any grid lines, reconstruct clean high-fidelity photorealistic skin beneath it." If a clip is *still* blocked with the strong grid, **escalate HARDER, never lighter** — denser/darker grid → fewer faces per clip → single face sheet (see Step 2's ladder). Light grids reliably fail, so never step down. Blocked jobs cost **no output credits**, so escalating/retrying is free.

### `end_image_url` usage

If the API/model supports an `end_image_url` parameter (an image to steer the clip toward its final frame — useful for landing on a specific end card, a freeze-frame, or a precise handoff point), keep in mind:
- It only works if the **prompt text also describes the ending frame** — passing `end_image_url` alone without matching language in the prompt is unreliable. Explicitly describe in the prompt what should be happening/visible at the end of the clip, consistent with the image supplied.
- This is the mechanism to use when a clip needs to land on a specific end card or title/text-overlay frame — don't rely on the model inferring it from the reference images alone.

For every clip:
- Attach the scene's character reference(s) and location reference as **hosted URL** inputs to the reference-to-video call (see above — never base64). **All clips within the same scene must use the same scene-specific character references and the same location reference** — don't mix reference versions (e.g. an earlier costume variant, or a different location shot) across clips in one scene.
- Use the clip's prompt from the production bible, explicitly stating how each reference image should be used, and describing the ending frame in the prompt if `end_image_url` is also being used.
- **Paste the locked Look Bible verbatim into every clip's prompt, and use the locked generation settings.** Insert the canonical Look Bible string (from Step 1) into each prompt character-for-character — never paraphrased or re-summarized per clip — so all clips share identical style tokens. Generate every clip with the same locked provider/model/resolution/aspect/frame-rate and the same seed strategy chosen in Step 1; don't vary settings between clips. Where the model supports an image style/reference anchor, attach the approved look-anchor frame so the grade and palette hold visually as well as textually.
- **Declare each reference sheet's role in the prompt** so the model uses it as a reference, not a layout to reproduce. Reference-to-video models weight prompt text heavily when deciding *how* to use an attached image — an unlabeled multi-panel sheet is sometimes copied literally as a grid/split-screen. Prevent this explicitly: for a **location composite**, say something like "@imageN is a multi-angle reference sheet of the [location] showing the same space from several viewpoints — use it for spatial/layout/palette/lighting consistency only; render a single continuous scene; do NOT reproduce the panels, grid, or split-screen." For a **character sheet**, say "@imageN is a character turnaround reference — use it to keep the character's identity and appearance consistent; do NOT render multiple copies, a grid, or a turnaround." Make this declaration for every reference-sheet image attached to the clip.
- **Attach the in-frame detail assets** (brand assets, screen/product UI, on-screen text, key props — from Step 2's "in-frame detail assets") as **hosted-URL** references for any clip that shows them, and state in the prompt how each should appear (e.g. "the laptop screen shows @image3, the Sai app UI"). This is how logos, screens, text, and products render faithfully — never rely on the video model to generate them from a text description. If a clip's output shows garbled text, a wrong/fake logo, or mushy UI, the fix is to supply (or improve) an image reference for that element, not to reword the prompt. Attach an in-frame detail asset (logo, app/UI screen, product, prop) **only to the specific clip where it is actually on screen** — a reference-to-video model incorporates every reference you attach, so brand/app assets attached to clips that don't show them will bleed into frame. Attach only what the clip needs.
- Embed the voice profile of every speaking character in the clip — pulled verbatim from `references/characters/voice-profiles.json` (authored in Step 1) — directly into this clip's video-generation prompt, then include lip-synced dialogue, natural vocal reactions, ambient sound, and SFX consistent with that profile. Re-state the profile in-prompt for each clip; never rely on the model carrying a voice over from a prior clip.
- **No background music of any kind.** This means both: don't add a music track (that's Step 6), and explicitly instruct the video model not to generate one. Every clip's video-generation prompt must carry an explicit no-music / no-score instruction — e.g. "no background music, no musical score — only ambient room tone, dialogue, and natural SFX." If the API/model exposes a granular audio flag that separates score from dialogue/ambient, prefer disabling model-generated music there as well. Remember cinematic/emotional prompt wording nudges the model toward adding a score, so always pair it with the explicit no-music instruction.

**QA loop**: after each clip generates, check it against the bible — character/costume consistency, location match, lip-sync, action correctness — before moving to the next. Regenerate immediately if a clip has drifted (wrong face, wrong costume, broken lip-sync), rather than waiting to catch it at assembly. If a clip morphs to black or produces garbage/random output partway through, that's almost always a base64-input bug, not a prompting issue — check that all reference images were passed as hosted URLs before retrying with a new prompt. Track pass/fail per clip.

**Cross-clip coherence check (compare clips against each other, not just the bible).** Per-clip QA above catches a clip that's wrong on its own; it does NOT catch look *drift between* clips, which is only visible by comparison. After clips are generated (and incrementally as you go), lay them **side by side** and check that color grade, palette, contrast/black levels, skin tone, and light direction/temperature match across all clips. Flag outliers — a clip that's greener, warmer, brighter, higher-contrast, or plasticier than its neighbors — and regenerate it with the verbatim Look Bible and locked settings/seed. This clip-vs-clip pass is what keeps the whole film feeling like one production; don't rely on per-clip checks alone to deliver coherence.

## Step 5 — Assemble and Review

Combine all clips per the master timeline; add subtitles and text overlays per the bible.

**Transitions — selective, not on every clip.** Execute the per-boundary transition plan from the bible's master timeline. Within a scene, join clips with **clean hard cuts** — they're continuous and designed with invisible boundaries (dialogue lands before the cut; see Step 1), so added fades would only make the action feel choppy. Reserve deliberate transitions for **scene boundaries**, especially the **last clip of a scene into the first clip of the next scene**: a **fade out / fade in (dip to black)** or a gentle **cross-dissolve** smooths the location/time shift (use a brief fade to black for larger time jumps). Keep transitions subtle and consistent, and apply them sparingly — a transition on every cut reads as amateur.

**Apply a unifying color grade across the whole timeline (coherence safety net).** Even with a locked Look Bible and matched references, small grade/exposure differences can remain between separately generated clips. As a final normalizer, apply **one consistent color grade / LUT across the entire assembled timeline** — match exposure, white balance, contrast, and palette so every clip reads as the same film stock — and nudge any remaining outlier clip toward the group before locking picture. This is a safety net on top of the cross-clip coherence check in Step 4, not a substitute for regenerating a badly drifted clip.

Do one final full-video pass against the bible (timing, clip order, dialogue, lip-sync, continuity, audio, subtitle styling, visual consistency, and that transitions land only where planned) — this is an assembly-level check, since per-clip QA already happened in Step 4.

## Step 6 — Generate and Add Background Music

**Do not skip this step.** The clips are deliberately music-free (Steps 1–4); this is where the film gets its single score. "No music per clip" is NOT "no music in the final video" — never conflate the two. Always perform Step 6 unless the user has explicitly said they want no music at all.

Using the confirmed style/mood/tempo from pre-production, generate one continuous background-music track for the whole assembled video (not per-clip). Add it only in post; balance levels so it never masks dialogue, vocal reactions, ambient audio, or SFX.

## Step 7 — Export

Export the final MP4 using the resolution, aspect ratio, frame rate, and runtime confirmed during pre-production.
