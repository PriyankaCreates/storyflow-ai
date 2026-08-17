# Research Report: Deterministic Multi-Scene Story Generation

## 1. Purpose

This project investigates whether a short natural-language story request can be transformed into a coherent narrated video through a deterministic orchestration layer built around separate generative services. The goal was not simply to generate four unrelated clips. It was to preserve story order, character identity, scene meaning, narration timing, media compatibility, and a consistent visual language across the final video.

The original test case used a familiar hare-and-tortoise narrative. Later tests expanded to a robot and firefly, a fox and bird, a sailboat and whale, a child carrying a lantern, Cinderella, a red panda, an elephant and butterfly, and other short client-style requests. These tests exposed both the strengths of structured orchestration and the limitations of independently sampled generative media.

## 2. System under study

The evaluated workflow is:

```text
Story request
  -> structured storyboard and invariant descriptions
  -> scene image prompts
  -> scene images
  -> scene narration and TTS duration
  -> image-to-video prompts and frame counts
  -> scene videos
  -> FFmpeg audio/video assembly
  -> final narrated MP4
```

The application uses a language model as the planning layer, a prompt-to-image endpoint for scene keyframes, a general TTS endpoint for narration, an image-to-video endpoint for motion, and FFmpeg for deterministic media assembly.

## 3. Research questions

The work focused on five questions:

1. Can a language model reliably decompose a short request into four to eight connected scenes?
2. Can explicit character and environment descriptions keep independently generated images consistent?
3. Can narration be kept semantically aligned with visible scene content?
4. Can video duration follow natural speech without speeding up the voice?
5. Which failures can be detected automatically, and which require semantic visual review?

## 4. Deterministic orchestration approach

### 4.1 Structured planning

The planner is instructed to return strict scene JSON containing a story action, narration, visible-character list, required objects, allowed objects, continuity constraints, and motion intent. A character bible defines invariant anatomy, colors, materials, clothing, accessories, locomotion, and feature counts. An environment bible defines recurring geography and lighting.

The planner also receives narrative-shape constraints: the first scene establishes the situation, middle scenes advance obstacles or attempts, the penultimate scene contains the decisive action, and the final scene resolves the story. Consecutive scenes must not repeat the same title, action, staging, or camera distance.

### 4.2 Fixed visual recipe

A fixed house style is inserted into every image prompt. It specifies the rendering medium, lens, depth of field, lighting, material treatment, color behavior, contrast, and forbidden alternative styles. This reduced large art-direction changes caused by the planner returning inconsistent or malformed style values.

### 4.3 Scene-level cast and object constraints

Every scene uses an exhaustive visible-character list and exact instance count. Missing or not-yet-revealed characters are omitted from early prompts. Required story objects are explicitly listed, while unnecessary animals, people, props, text, and background duplicates are forbidden.

This was added after observing duplicated hares, duplicated Cinderella characters, unexpected cats, missing stepping stones, extra trunks, and unrelated background objects.

### 4.4 Restrained animation

Image-to-video prompts emphasize small, physically plausible motion. They preserve the source composition and prohibit character translation, redesign, duplication, extra anatomy, disappearing objects, new subjects, aggressive camera movement, and scene transitions. Narration duration is measured first so the number of video frames can be selected before generation.

### 4.5 Natural audio assembly

Narration is generated independently for every scene. FFmpeg maps the original video stream and narration audio without accelerating speech. Scene clips are normalized for compatible playback and concatenated into a final MP4. This replaced an earlier approach in which audio appeared sped up to match a fixed video duration.

## 5. Quality controls implemented

The prototype includes bounded retries rather than unlimited regeneration.

Storyboard preflight checks:

- exact scene count;
- presence of character and environment bibles;
- unique scene titles and progressive actions;
- visible-character whitelist and instance counts;
- required story objects;
- narration presence and basic alignment;
- prevention of overly similar consecutive scenes.

Media checks:

- file presence and minimum size;
- successful decode;
- expected image/video resolution;
- unusable image exposure;
- unreadable or mostly silent narration;
- extended black video segments;
- final MP4 decode verification.

Only the failed asset is regenerated during a retry. Failed or rejected runs are not used as trusted visual references.

## 6. Experimental observations

### What improved

- Structured scene JSON produced clearer story progression than free-form prompt generation.
- Exact visible-character lists reduced, but did not eliminate, duplicate subjects.
- Removing hidden characters from early scene prompts reduced premature character appearances.
- A fixed house style improved art-direction stability.
- Short, scene-specific prompts performed better than extremely long contradictory prompts.
- Restrained micro-motion generally produced more stable video geometry than walking, flying, lifting, or complex interaction prompts.
- Narration-first duration planning improved audio pacing.
- Technical validation caught corrupt video, black frames, silent audio, FFmpeg decode failures, and unusable outputs.
- Some test stories produced good end-to-end results, demonstrating that the orchestration design is viable.

### What continued to fail

- Character appearance sometimes changed between scenes: body shape, facial features, limbs, wheels, clothing, or materials.
- Models occasionally duplicated a main character or introduced unrelated animals and props.
- A required object could be absent even when stated in the prompt.
- Narration and imagery could diverge, such as narration describing stepping stones that were not visible.
- Text or subtitle-like marks occasionally appeared inside generated images.
- Complex image-to-video movement caused deformation, duplication, unstable anatomy, and object transformation.
- Independent samples could switch art style despite repeated style descriptions.
- Technical validation could confirm that a file decoded, but not that it depicted the correct story.

## 7. Root-cause assessment

The primary unresolved issue is architectural, not merely prompt quality. The prompt-to-image endpoint generates every scene independently and does not accept an approved character reference, identity embedding, pose reference, or structural conditioning input. Text can describe identity but cannot force the sampler to reproduce exactly the same visual subject across several independent generations.

The second gap is semantic evaluation. File-level checks can detect corruption, resolution problems, silence, or black frames. They cannot reliably determine whether there are two Cinderellas, whether a flower became a cat, whether stepping stones are present, or whether the same robot has wheels in one scene and legs in another. Those checks require a multimodal vision model or human review.

## 8. Production-readiness assessment

The prototype is suitable for demonstrating orchestration, API integration, retry logic, media assembly, and the value of structured prompts. It is not yet suitable for unattended production where every generated story must be visually correct.

Production readiness requires:

1. Reference-conditioned image generation using image-to-image, IP-Adapter, InstantID, ControlNet, identity embeddings, or an equivalent provider capability.
2. A vision evaluator that compares every scene with the approved character reference and storyboard.
3. Explicit acceptance thresholds for identity, count, required objects, text artifacts, style, and narration alignment.
4. Candidate generation and ranking rather than trusting the first sample.
5. Persistent job queues, idempotent webhook handling, authenticated callbacks, cancellation, observability, and durable storage.
6. Provider-specific rate-limit scheduling and cost ceilings.
7. A stable hosted deployment instead of an account-less temporary tunnel.
8. A human escalation path when bounded retries fail.

## 9. Recommended production pipeline

```text
Request
  -> policy and story validation
  -> structured storyboard
  -> storyboard semantic QA
  -> approved character/style reference
  -> generate multiple image candidates per scene with reference conditioning
  -> vision-based ranking and rejection
  -> TTS generation and pronunciation QA
  -> image-to-video candidate generation
  -> sampled-frame vision QA and motion QA
  -> deterministic FFmpeg normalization and assembly
  -> final audiovisual QA
  -> publish or human review
```

Retries should remain bounded. A system that promises to regenerate “until error-free” risks infinite cost because generative correctness is probabilistic and no automatic evaluator is perfect.

## 10. Conclusion

The research shows that deterministic orchestration substantially improves a multi-model story pipeline: it creates clearer plans, safer prompts, natural narration timing, compatible media, and recoverable jobs. It does not make independently generated visual samples deterministic. The decisive next step is to combine the orchestration layer with reference-conditioned generation and semantic multimodal QA. Until then, the system should be presented as an advanced prototype with measurable safeguards and known visual-consistency limitations.

