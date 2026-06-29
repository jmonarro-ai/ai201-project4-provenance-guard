# Provenance Guard

A Flask API for AI content attribution. Any creative sharing platform can plug into this system to classify submitted text as human-written or AI-generated, return a confidence score, display a transparency label to readers, and handle appeals from creators who believe they've been misclassified.

---

## Portfolio Walkthrough
[Watch the walkthrough video here](https://www.loom.com/share/f32f166b8f31446a8757d1aa8557efd6)

## Architecture Overview

When a creator submits text via `POST /submit`, it passes through two independent detection signals — a Groq LLM classifier and a stylometric heuristics analyzer. Their scores are combined into a single weighted confidence score (60% LLM, 40% stylometrics). That score drives a transparency label generator which produces one of three plain-language labels. The full decision is written to a structured JSON audit log before the response is returned to the caller.

When a creator submits an appeal via `POST /appeal`, the system looks up their original submission by `content_id`, updates its status to `under_review`, logs the appeal reasoning alongside the original decision, and returns a confirmation.

```
SUBMISSION FLOW
===============
Creator
  |
  | {text, creator_id}
  v
POST /submit
  |
  | raw text
  v
Signal 1: Groq LLM -----> llm_score (0.0-1.0)
  |                              |
  | raw text                     |
  v                              |
Signal 2: Stylometrics --> style_score (0.0-1.0)
                                 |
                    [llm_score, style_score]
                                 |
                                 v
                    Confidence Scorer
                    (60% LLM + 40% Style)
                                 |
                        combined_score (0.0-1.0)
                                 |
                                 v
                    Transparency Label Generator
                    <= 0.25  --> "Likely Human-Written"
                    0.26-0.69 -> "Uncertain"
                    >= 0.70  --> "Likely AI-Generated"
                                 |
                           label_text
                                 |
                                 v
                          Audit Logger
                    {content_id, timestamp,
                     llm_score, style_score,
                     confidence, label, status}
                                 |
                                 v
                    JSON Response to Creator

APPEAL FLOW
===========
Creator
  |
  | {content_id, creator_reasoning}
  v
POST /appeal
  |
  | look up content_id
  v
Update status --> "under_review"
  |
  v
Audit Logger + Confirmation response
```

---

## API Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/submit` | POST | Submit text for attribution analysis |
| `/appeal` | POST | Contest a classification decision |
| `/log` | GET | View structured audit log entries |
| `/dashboard` | GET | View analytics metrics (stretch feature) |

---

## Detection Signals

### Signal 1: Groq LLM Classification

**What it measures:** Semantic and stylistic coherence — whether the text reads as AI-generated based on phrasing patterns, structural uniformity, generic transitions, and absence of personal voice.

**Why I chose it:** LLMs can evaluate text holistically in a way that pure statistics cannot. They recognize overused AI phrases ("it is important to note that"), unnaturally smooth transitions, and the absence of personal idiosyncrasies that characterize human writing. This is the strongest single signal available without training a custom classifier.

**What it misses:** Highly polished human writing — academic papers, professional essays, non-native English speakers writing carefully — can be misclassified as AI-generated. It also struggles with AI text that has been substantially rewritten by a human.

**Output:** A float between 0.0 (human) and 1.0 (AI), returned alongside a one-sentence reasoning explanation.

### Signal 2: Stylometric Heuristics

**What it measures:** Three statistical properties of the text:
- **Sentence length variance** — standard deviation of sentence lengths. Human writing varies more; AI writing is uniform.
- **Type-token ratio (TTR)** — unique words divided by total words. Human writing uses more diverse vocabulary.
- **Punctuation density** — informal punctuation marks (!, ?, ;, —, …) per word. Human writing uses more expressive punctuation.

**Why I chose it:** These structural properties are entirely independent of the LLM's semantic judgment. A text can fool the LLM with careful phrasing but still exhibit statistical uniformity. Combining both signals makes the system more robust than either alone.

**What it misses:** Formal human writing (legal documents, academic papers, technical manuals) is naturally uniform and will score similarly to AI text. This signal cannot capture meaning — only structure.

**Output:** A float between 0.0 (human-like irregularity) and 1.0 (AI-like uniformity), broken down into three sub-scores.

---

## Confidence Scoring

Both signals are combined using a weighted average:

```
confidence = (llm_score × 0.60) + (style_score × 0.40)
```

The LLM is weighted higher (60%) because semantic analysis is a stronger signal than surface statistics. Stylometrics provides an independent structural cross-check.

**Why this approach:** The two signals are genuinely independent — one is semantic, one is structural. Weighting them rather than averaging equally reflects the difference in their reliability. A 60/40 split was chosen as a starting point that could be tuned with real data in production.

**Validation — two example submissions with noticeably different scores:**

High-confidence AI example:
```json
{
  "text": "Artificial intelligence represents a transformative paradigm shift...",
  "llm_score": 0.92,
  "style_score": 0.4915,
  "confidence": 0.7486,
  "label": "Likely AI-Generated"
}
```

Lower-confidence human example:
```json
{
  "text": "Ive been thinking a lot about remote work lately...",
  "llm_score": 0.42,
  "style_score": 0.3311,
  "confidence": 0.3844,
  "label": "Attribution Uncertain"
}
```

The spread between clearly AI text (0.7486) and uncertain/human text (0.3844) demonstrates meaningful variation in the scoring system.

---

## Transparency Labels

All three variants are written in plain language. No technical jargon. A non-technical reader can understand what each label means.

### High-Confidence AI (confidence ≥ 0.70)
```
⚠️ Likely AI-Generated
Our system has analyzed this content and found strong indicators that it was
produced by an AI writing tool. This label is based on automated analysis and
may not be perfect. If you are the creator and believe this is incorrect,
you can submit an appeal to have your work reviewed by a person.
Confidence: High
```

### Uncertain (confidence 0.26 – 0.69)
```
🔍 Attribution Uncertain
Our system analyzed this content but could not confidently determine whether
it was written by a person or generated by AI. This sometimes happens with
formal writing styles or edited content. The creator can submit an appeal
to provide additional context.
Confidence: Low — treat this label with caution
```

### High-Confidence Human (confidence ≤ 0.25)
```
✅ Likely Human-Written
Our system analyzed this content and found strong indicators that it was
written by a person. This label is based on automated analysis. No system
is perfect — if you have concerns about this content, you can flag it for review.
Confidence: High
```

---

## Rate Limiting

The `/submit` endpoint is limited to **10 requests per minute and 100 requests per day** per IP address.

**Reasoning:** A real writer submitting their own creative work might submit 2–5 pieces in a session, occasionally bursting to 10 if they are testing or submitting a batch. A limit of 10 per minute accommodates legitimate power users while making automated flooding expensive. The 100 per day cap prevents a script from cycling through the per-minute limit repeatedly throughout a day. These numbers were chosen to reflect realistic human usage patterns on a creative writing platform, not arbitrary defaults.

**Rate limit behavior — evidence:**
```
200
200
200
200
200
200
200
200
200
200
429
429
```
Requests 1–10 succeed. Requests 11–12 return HTTP 429 Too Many Requests.

---

## Appeals Workflow

Any creator who receives a `content_id` from `/submit` can contest their classification by calling `POST /appeal` with their `content_id` and a `creator_reasoning` explanation.

When an appeal is received the system immediately updates the content's status to `under_review` in the audit log, records the creator's reasoning alongside the original classification decision, and returns a confirmation. No automated re-classification occurs — a human reviewer reads the audit log and makes the final judgment.

**Example appeal response:**
```json
{
  "message": "Your appeal has been received and is under review.",
  "content_id": "6cf48452-92ce-4590-a039-9b50e4833013",
  "status": "under_review",
  "appeal_timestamp": "2026-06-29T03:01:58.619291+00:00"
}
```

---

## Audit Log

Every attribution decision is written to `audit_log.json` as a structured entry. The log captures: timestamp, content ID, creator ID, attribution result, confidence score, both individual signal scores, stylometric breakdown, transparency label, status, and appeal reasoning if filed.

Call `GET /log` to view the most recent 20 entries.

**Sample entries (3 shown):**

```json
{
  "content_id": "6cf48452-92ce-4590-a039-9b50e4833013",
  "creator_id": "test-user-1",
  "timestamp": "2026-06-29T00:25:59.406239+00:00",
  "attribution": "likely_human",
  "confidence": 0.34,
  "llm_score": 0.34,
  "style_score": null,
  "label": "Attribution Uncertain",
  "status": "under_review",
  "appeal_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical."
}
```

```json
{
  "content_id": "4cc489b9-14ec-4653-bde9-d17e0e5049f6",
  "creator_id": "test-user-2",
  "timestamp": "2026-06-29T00:27:12.213262+00:00",
  "attribution": "likely_ai",
  "confidence": 0.85,
  "llm_score": 0.85,
  "style_score": null,
  "label": "Likely AI-Generated",
  "status": "classified",
  "appeal_reasoning": null
}
```

```json
{
  "content_id": "bffe884d-3183-4e99-8ac4-ab3dd2365c82",
  "creator_id": "test-user-3",
  "timestamp": "2026-06-29T00:29:41.997232+00:00",
  "attribution": "likely_human",
  "confidence": 0.12,
  "llm_score": 0.12,
  "style_score": null,
  "label": "Likely Human-Written",
  "status": "classified",
  "appeal_reasoning": null
}
```

---

## Analytics Dashboard (Stretch Feature)

`GET /dashboard` returns three metrics computed from the audit log:

- **AI vs Human verdict ratio** — proportion of submissions classified as AI-generated vs human-written
- **Appeal rate** — proportion of submissions that have been appealed
- **Average confidence score** — mean confidence across all submissions

**Example response:**
```json
{
  "total_submissions": 38,
  "verdicts": {
    "likely_ai": 27,
    "likely_human": 11,
    "ai_ratio": 0.7105,
    "human_ratio": 0.2895
  },
  "appeal_rate": {
    "total_appeals": 1,
    "rate": 0.0263
  },
  "average_confidence": 0.6198
}
```

---

## Known Limitations

**Non-native English speakers writing formally** will likely be misclassified as AI-generated. A writer who carefully avoids contractions, uses conservative punctuation, and constructs uniform sentence structures — because they are being precise in a second language — will score high on both signals. The stylometric signal penalizes low punctuation density and low sentence variance, both of which are natural in careful formal writing. The LLM signal will interpret formal phrasing as AI-like coherence. This is the most predictable source of false positives in the current system and the primary reason the appeals workflow exists.

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants in `planning.md` before building anything forced a decision about what confidence thresholds meant to a real user, not just to the algorithm. When it came time to implement `get_label()`, the function almost wrote itself because the spec had already answered the hard question — what does a 0.60 score mean to someone who just had their poem flagged?

**One way implementation diverged from the spec:** The confidence thresholds in `planning.md` were originally set at 0.85 (AI) and 0.14 (human). In practice, the stylometric signal pulled scores toward the middle — no submission reached 0.85 with both signals combined, and very few dropped below 0.14. The thresholds were adjusted to 0.70 (AI) and 0.25 (human) to reflect the actual score distribution the two-signal system produces. This was a calibration decision that could only be made after seeing real outputs, not before.

---

## AI Usage

**Instance 1: Generating the Flask app skeleton and Groq signal function**
I provided Claude with the Detection Signals section of `planning.md` and the architecture diagram and asked it to generate the Flask app skeleton with a `/submit` route stub and the Groq LLM signal function. It produced a working skeleton but the confidence scoring function used `style_score=None` as a placeholder with a conditional branch — I kept this structure in M3 as intended but replaced it entirely in M4 once the stylometric signal was working, removing the conditional rather than leaving dead code in place.

**Instance 2: Generating the stylometric signal and confidence scoring logic**
I provided the Detection Signals and Uncertainty Representation sections and asked Claude to generate the `get_style_score()` function and the weighted confidence scorer. The generated TTR normalization formula produced `ttr_score: 0.0` for all inputs because the normalization range assumed a TTR distribution that did not match the actual texts. I diagnosed this by examining the individual sub-scores, identified the formula as the problem, and replaced the normalization with a simpler linear scale that produced meaningful variation across test inputs.

---

## Setup

```bash
git clone https://github.com/YOUR-USERNAME/ai201-project4-provenance-guard.git
cd ai201-project4-provenance-guard
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file:
```
GROQ_API_KEY=your_key_here
```

Run:
```bash
python app.py
```

---

## Requirements

```
flask>=3.0.0
flask-limiter>=3.5.0
groq==0.15.0
python-dotenv==1.0.1
```