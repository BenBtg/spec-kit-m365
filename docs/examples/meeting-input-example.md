# Example: Meeting Transcript -> Markdown Artifact

This example shows how a Teams meeting transcript is fetched and saved as Markdown.

---

## Source: Meeting Transcript (VTT format)

The following is a sample WebVTT transcript downloaded via `m365 teams meeting transcript get`:

```vtt
WEBVTT

00:00:05.000 --> 00:00:12.000
<v Sarah Chen>Okay, let's get started. The main topic today is the file upload redesign. 
We've been getting a lot of complaints about the current drag-and-drop experience.</v>

00:00:12.500 --> 00:00:25.000
<v Marcus Johnson>Right. The main issues are: no progress indicator, the 10MB limit is too 
low for our enterprise users, and there's no way to resume a failed upload. I've looked 
at the support tickets — these are consistently in our top 5 complaints.</v>

00:00:25.500 --> 00:00:38.000
<v Priya Patel>From the design side, I've mocked up two approaches. Option A keeps the 
current modal but adds a progress bar and retry button. Option B replaces it with an 
inline upload zone that shows progress directly in the document list. I can share the 
Figma link in chat.</v>

00:00:38.500 --> 00:00:48.000
<v Sarah Chen>I've seen both mocks. I think Option B is cleaner — the inline experience 
feels more modern and users don't lose context by having a modal pop up. What do you 
think, Marcus?</v>

00:00:48.500 --> 00:01:02.000
<v Marcus Johnson>Option B is more work on the frontend but I agree it's better UX. For 
the backend, we need to switch to chunked uploads to support larger files and 
resumability. I'd recommend using the tus protocol — it handles resume natively.</v>

00:01:02.500 --> 00:01:15.000
<v David Kim>From infrastructure, chunked uploads mean we need to rethink storage. We 
currently write directly to blob storage. With chunked uploads, we'd need a temporary 
staging area to assemble chunks before moving to final storage. I can set up an Azure 
staging container.</v>

00:01:15.500 --> 00:01:25.000
<v Sarah Chen>Makes sense. What should the new file size limit be? Our enterprise 
customers have asked for up to 500MB.</v>

00:01:25.500 --> 00:01:38.000
<v David Kim>500MB is doable with chunked uploads. I'd suggest 5MB chunks. We should 
also add virus scanning before moving from staging to final storage. That's a hard 
requirement from the security team.</v>

00:01:38.500 --> 00:01:48.000
<v Marcus Johnson>Agreed on virus scanning. What about file type restrictions? Currently 
we allow everything except executables.</v>

00:01:48.500 --> 00:01:58.000
<v Sarah Chen>Let's keep the current file type restrictions for now. We can revisit that 
separately. So to confirm — we're going with Option B, chunked uploads via tus protocol, 
500MB limit, virus scanning. Everyone good with that?</v>

00:01:58.500 --> 00:02:02.000
<v Marcus Johnson>Yes, sounds good.</v>

00:02:02.500 --> 00:02:04.000
<v Priya Patel>Confirmed.</v>

00:02:04.500 --> 00:02:06.000
<v David Kim>Agreed.</v>

00:02:06.500 --> 00:02:20.000
<v Sarah Chen>Great. For action items — Marcus, can you start on the tus protocol 
integration? David, set up the staging container and virus scanning pipeline. Priya, 
finalize the Option B designs and share them by Wednesday.</v>

00:02:20.500 --> 00:02:28.000
<v Marcus Johnson>Will do. One question — should we support concurrent uploads? Like 
uploading multiple files at once?</v>

00:02:28.500 --> 00:02:38.000
<v Sarah Chen>Good question. Yes, we should support at least 3 concurrent uploads. The 
inline UI in Option B actually lends itself well to that — you can show multiple 
progress bars.</v>

00:02:38.500 --> 00:02:48.000
<v Priya Patel>I'll add concurrent upload states to the design. What about drag-and-drop 
for folders? That came up in the user research.</v>

00:02:48.500 --> 00:02:58.000
<v Sarah Chen>Let's park folder upload for now. It adds a lot of complexity with 
directory structure preservation. We can tackle it in a follow-up if the single-file 
experience lands well.</v>

00:02:58.500 --> 00:03:08.000
<v David Kim>One more thing — we need to think about what happens when a user closes the 
browser mid-upload. With tus, the chunks are on the server, so we could show a 
"resume upload" option when they come back.</v>

00:03:08.500 --> 00:03:18.000
<v Sarah Chen>That's a great feature and one of the main reasons to use tus. Yes, let's 
include resume-on-return as a requirement. Any other items? ... Okay, I think we're 
good. I'll summarize and send out notes. Thanks everyone.</v>
```

---

## Generated Output: `transcript-MSo1N2Y5ZGFjYy03MWJm.md`

The following is a sample output from `/speckit.m365.fetch-transcript`:

```markdown
---
Source:
  type: "m365-transcript"
  meetingId: "MSo1N2Y5ZGFjYy03MWJm..."
  transcriptId: "MSMjMCMjNmU2OTc2OTUt..."
  fetchedAt: "2026-04-10T16:00:00Z"
---

# Meeting Transcript: File Upload Redesign

## Transcript

[00:00:05] Sarah Chen: Okay, let's get started. The main topic today is the file upload redesign...

[00:01:25] David Kim: 500MB is doable with chunked uploads. We should also add virus scanning...

[00:02:06] Sarah Chen: For action items - Marcus, start tus integration. David, set up staging and scanning...
```

## Summary

The fetch command stores transcript content as a reviewable Markdown artifact.

Next step:

```text
speckit specify .specify/specs/transcript-MSo1N2Y5ZGFjYy03MWJm.md
```
