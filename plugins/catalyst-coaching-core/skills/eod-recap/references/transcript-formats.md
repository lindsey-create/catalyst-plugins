# Transcript formats - per-recorder stripping (for the word-count floor)

Used by Step 3's no-show check and Step 2c. Strip the recorder's own formatting before counting spoken words - count only words a person actually said, never timestamps, cue numbers, speaker labels, or platform boilerplate.

## zoom

Native Zoom cloud-recording transcript (.vtt). Structure per cue:

```
1
00:00:09.180 --> 00:00:12.300
Lindsey Hammer: Back, Bernadette, I actually,
```

Strip: the `WEBVTT` header, the numeric cue index line, the `HH:MM:SS.mmm --> HH:MM:SS.mmm` timestamp line, and the leading `<Speaker Name>: ` on each spoken line. Count what remains. A cue with no speaker line (blank) contributes nothing.

## fathom

Inline per-line format:

```
[MM:SS](https://fathom.video/...) Speaker Name: text of what they said
```

Strip: the `[MM:SS](url)` prefix and the `Speaker Name: ` label on each line. Count the remaining text.

## otter

Header-per-turn format:

```
Speaker 1  02:25
Oh, I don't think stream day. Let me check.
```

Strip: the `<Speaker Name>  MM:SS` header line above each turn, and the `Transcribed by https://otter.ai` footer line. Count the remaining spoken text.

## Adding a new recorder

If a coach's `recorder` Brain row names a service not listed here, do not guess at its format - flag it in the final message ("no transcript-format entry for recorder: <name>") and skip the word-count floor for that run rather than mis-strip and miscount.
