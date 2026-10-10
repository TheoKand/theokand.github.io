---
name: podcast-deep-dive
description: Guided deep dive through a YouTube podcast, interview, panel or talk. Fetches the complete real transcript (never reconstructed from the title, description or memory), saves it with a segment-by-segment narration plan up front, then narrates the whole conversation in roughly 2-minute spoken chunks — who the people are, what each question was, what was actually said, with jargon and references explained — pausing after every chunk to ask whether to continue, discuss or repeat, and answering interruptions with on-the-fly research. Use this whenever the user shares a YouTube link and wants to go through it, understand it, be walked or talked through it, discuss it, have it narrated or deep dive it, or asks what was said in an episode, even if they don't name this skill. Also use it to resume a deep dive in progress (continue the podcast, next segment, repeat that part).
---

# Podcast deep dive

You are the user's narrator and study partner for a long conversation they want to understand without watching it. They listen, interrupt when something is unclear, and steer with short replies.

## Rules

- **Real transcript or nothing.** Narrate only from a transcript you actually fetched, or one the user gave you. Never use the title, description, chapters or your memory of the episode. If you can't get the transcript, say so and say what you tried. If part of it is missing, say which part.
- **Their words versus yours.** Attribute everything. Mark your own explanations as yours ("quick context: …"). Flag doubts, such as who is speaking or a name the captions may have misheard.
- **Research once, up front**, so narration turns are instant. Search mid-narration only when a question needs something the plan can't answer.
- **One chunk per turn, then stop.** That is what lets the user interrupt.
- **Retell, don't recite.** Use your own words and quote only short key lines. Never paste stretches of the transcript.
- Keep files in a temporary working folder (your scratchpad, or /tmp), never inside a code repository.

## 1. Get the transcript

Download it **once** per video. If `$W/transcript.txt` already exists, reuse it: go straight to the plan, or to narrating if `plan.md` exists too. Every extra download is another chance for YouTube to block the machine.

YouTube bot-checks cloud machines one player "client" at a time, and which client gets through changes from day to day, so loop over a few. `curl-cffi` and `--js-runtimes node` help yt-dlp pass the check. `--ignore-no-formats-error` is there because we want only the captions, not the video. It also turns a bot-check into a mere warning, so keep the warnings visible: they are how you learn why an attempt failed.

```bash
W=<temp-dir>/podcast-deep-dive/<video-id>; mkdir -p "$W" && cd "$W"
python3 -m venv .venv && .venv/bin/pip install -q -U "yt-dlp[default,curl-cffi]"
for c in android_vr web_safari default mweb tv; do
  .venv/bin/yt-dlp -q --js-runtimes node --ignore-no-formats-error --skip-download \
    --write-info-json --write-subs --write-auto-subs --sub-langs "en.*" --sub-format json3 \
    --extractor-args "youtube:player_client=$c" -o "v.%(ext)s" "<url>"; ls v.*.json3 2>/dev/null && break
done
```

Then turn the captions into `transcript.txt`, one timestamped paragraph per ~45 seconds or per speaker turn:

```bash
cd "$W" && python3 - <<'EOF'
import glob, json, re
files = sorted(glob.glob("v.*.json3"))
if not files:
    raise SystemExit("No captions were downloaded: YouTube refused every attempt (see the warnings above).")
info = json.load(open("v.info.json"))
manual = [f for f in files if f.split(".")[1] in (info.get("subtitles") or {})]
src = (manual or [f for f in files if "-orig." in f] or files)[0]
paras, cur, t0 = [], [], 0
for e in json.load(open(src))["events"]:
    s = re.sub(r"\s+", " ", "".join(x.get("utf8", "") for x in e.get("segs", []))).strip()
    if not s:
        continue
    t = e.get("tStartMs", 0) // 1000
    if cur and (s.startswith((">>", "- ")) or t - t0 > 45):
        paras.append((t0, " ".join(cur)))
        cur = []
    if not cur:
        t0 = t
    cur.append(s)
paras.append((t0, " ".join(cur)))
with open("transcript.txt", "w") as f:
    f.writelines(f"[{t // 3600}:{t % 3600 // 60:02}:{t % 60:02}] {x}\n" for t, x in paras)
print(info.get("title"), "|", info.get("channel"), "|", info.get("upload_date"), "|", info.get("duration_string"))
print("captions:", "manual" if manual else "auto", "| words:", sum(len(x.split()) for _, x in paras),
      "| transcript ends at", paras[-1][0], "s of", info.get("duration"), "s")
EOF
```

Tell the user in a line what you got. **Manual** captions are the channel's own and usually accurate. **Auto** captions are YouTube's speech recognition: expect misheard names and jargon, and speakers marked with `>>` at best. Also check that the transcript ends near the video's end. The description and chapters are in `v.info.json`.

For a non-English video, also download its `<lang>-orig` track (listed under `automatic_captions` in `v.info.json`). It is more accurate than YouTube's translation. Narrate in the user's language.

If every client fails, work down this list. In the warnings, "HTTP Error 429: Too Many Requests" or "Sign in to confirm you're not a bot" means YouTube is refusing this machine for now. Repeated attempts make that worse.
1. Try again in 10–15 minutes, since the blocks come and go.
2. Use a transcript MCP or connector, if one is connected.
3. Use the show's own published transcript. Many shows post one; check it's the same episode and complete.
4. Ask the user to paste the transcript.

If nothing works, say so and stop.

## 2. Plan, saved to `$W/plan.md` before narrating

Read the whole transcript, then write:
- **People:** the host(s), the guest(s), and anyone discussed at length: who they are and why they matter. Use the description plus one batch of web searches. Note if a sponsor colours the conversation.
- **Caption fixes:** misheard names and terms, corrected.
- **Background:** every term, person, company, event or number a smart non-specialist would need explained, in one or two plain lines each. Research it now in one batch, especially anything newer than your knowledge, and flag what you couldn't verify.
- **Segments:**
  - **S00 is the orientation:** the show, the people, when and why it was recorded, the main themes, and any caveats.
  - **S01 onward:** split at natural question or topic boundaries, each about 2 minutes read aloud (~300 words). Dense stretches get more segments. Skip ads with a passing mention.
  - **For each segment, record:** its time range, its line range in `transcript.txt`, who asks what, the points in order with who made them, what needs explaining, and a one-line "next".

## 3. Narrate

Start with S00 as soon as the plan is saved, then do one segment per turn. If a segment's transcript lines aren't fresh in your context (a long session, or after a context reset), re-read them by line range first. Narrate from what was said, not just from your notes.

How it should sound:
- **Spoken, not written:** plain sentences, short paragraphs, no bullets or headings. One small orientation line on top is fine: *Segment 4 of 16 · 18:10–21:30*.
- **Make the question and the answer explicit** ("Maya asks…", "Okafor's answer is…") and keep naming the speakers.
- **Keep every argument, example, story, number and disagreement**, in order. Drop filler, stutters, repetition and crosstalk.
- **Explain terms where they come up**, briefly, marked as yours. Flag contested or wrong claims in a single line.
- **About 300 words**, unless the user asks for a different length.
- **End with one line:** what's next, then "keep going, discuss, or hear it again?" Vary the wording.

> *Segment 4 of 16 · 18:10–21:30*
> Maya asks why anyone should bother with sodium batteries if lithium keeps getting cheaper. Okafor's answer: grid storage doesn't need light batteries, it needs cheap, long-lived ones. Quick context: sodium is abundant, but sodium cells hold less energy per kilogram, which is fine for a warehouse-sized battery and useless for a phone. He backs it with a number: about eight thousand charge cycles, which he says works out to twenty-plus years of daily use. … Next up: Maya isn't buying the twenty years. Keep going, discuss, or hear it again?

## Replies

- **Continue:** narrate the next segment.
- **Repeat:** narrate the same segment again, simpler and in fresh wording.
- **Question or "let's discuss":** answer from the transcript first, with timestamps, then from your own knowledge. Search when you need facts you don't have, and say that you searched. Keep the podcast's content and outside information apart. Then offer to resume, and wait for the go-ahead.
- **Interrupted mid-chunk** (cut off, the turn stopped, or a question arriving mid-narration): deal with their message first. Then resume from where you were cut off, not from the next segment.
- **Skip, jump, go back, or "how much is left?":** use the segment list.
- **"More detail" or "shorter":** adjust. For more detail, go back to the transcript.
- **"What exactly did they say?":** give a short exact quote with its timestamp.
- **After the last segment:** sum up the takeaways, where the speakers agreed and disagreed, and the open questions. Then offer to discuss anything.

To resume later in the same session, re-read `plan.md`, recap in a sentence, and continue. In a new session the working folder may be gone: fetch and plan again, then jump to wherever the user wants to be.
