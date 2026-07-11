# Kisan Vaani — Multilingual Voice Advisory Agent for Farmers

**One agronomist's table of rules + live weather + Sarvam's full voice stack → the right advisory, to the right farmer, in the right language, over the right channel — every morning, automatically.**

---

## The problem

Agri-input companies, FPOs, and state agriculture departments need to reach millions of farmers with time-critical guidance — spray windows, irrigation calls, weather warnings. Today that means mass SMS in English/Hindi (which many farmers can't read), or expensive field-officer visits and call-centre agents. Advice that arrives late, in the wrong language, or as unreadable text is advice wasted. The gap isn't knowing what to tell farmers — it's delivering it to each one, in time, in a form they can actually use. That form is **voice, in their mother tongue.**

## What this solution does

Two n8n workflows share one localize-and-voice pipeline, driven two different ways.

**Workflow A — Daily Advisory (scheduled).** Every morning, on schedule:

1. Reads the farmer register (mock CRM — Google Sheet; **phone number is the farmer's identity**, as in the real world).
2. Fetches the **live 48-hour weather forecast** for each farmer's district (Open-Meteo API).
3. **Reasons:** classifies the weather condition (heavy rain / hot-dry / normal) and picks the single matching row from the expert-curated **advisory rules table** for that farmer's crop. It never invents agronomy — it only localizes expert content. Every message logs the rule ID used, so it's fully auditable.
4. Composes a short, warm, spoken-style advisory (Sarvam **sarvam-105b**).
5. Localizes it (Sarvam **Translate / Mayura**) into the farmer's language — Kannada and Marathi in this demo.
6. Voices it (Sarvam **TTS / Bulbul v3**) as a natural voice note.
7. Delivers it per the farmer's **channel** — WhatsApp voice note for smartphone users, outbound voice call for feature-phone users — and logs rule, timestamp, and preview back to the register.

**Workflow B — Campaign Broadcast (on demand).** A field officer records one ~20-second voice memo; Sarvam **STT (Saaras v3)** transcribes it, and the same translate-and-voice pipeline localizes and voices it for every farmer. One spoken sentence reaches the entire base in their own languages.

### Note on delivery in this PoC

The farmer register contains phone numbers only (no emails — farmers don't have them). In this proof-of-concept, deliveries are routed to a **demo inbox**, with the subject line declaring the simulated channel and target phone (e.g. `[SIMULATED WHATSAPP → +91-98XXX…]`) and the generated voice note attached. This keeps every send visible and audible for review. Production delivery is the **WhatsApp Business API** and **telephony** (e.g. Exotel / Sarvam Samvaad / Twilio) with TRAI/DND-compliant consent — see the 90-day rollout below.

## Architecture

![Architecture](Farm%20Advisory%20Workflow%20Architecture.png)

## Sarvam APIs used, and why they're core

| API | Where | Why it's load-bearing |
|---|---|---|
| Chat Completions (sarvam-105b) | Advisory composition | Classifies the weather condition, selects the rule, writes spoken-style copy — remove it and there's no message |
| Translate (mayura:v1) | Localization | Kannada / Marathi output that keeps the natural code-mixed terms farmers actually use |
| Text-to-Speech (bulbul:v3) | Voice notes | The product *is* voice — literacy-independent delivery |
| Speech-to-Text (saaras:v3) | Campaign workflow | Lets a non-technical field officer feed the system by simply speaking |

## Repo contents

```
Kisan Vaani A - Daily Advisory.json              ← main scheduled agent (import into n8n)
Kisan Vaani B - Campaign Broadcast (STT).json    ← STT campaign broadcast (import into n8n)
kisan_vaani_postman_collection.json              ← every Sarvam API call, runnable standalone
farmers.csv                                      ← mock CRM: farmer register (import into Google Sheets)
advisory_rules.csv                               ← expert advisory rules table (import into Google Sheets)
tts_player.html                                  ← offline player for Sarvam TTS base64 audio
env.example                                      ← credential placeholders (no real keys)
Farm Advisory Workflow Architecture.png          ← system architecture diagram
Prashant Patil_SarvamAI_Solution Architect Assignment.pdf   ← business write-up / solution overview
README.md
```

## Setup (≈15 minutes)

1. Get a Sarvam API key at dashboard.sarvam.ai (free signup credits).
2. Create a Google Sheet named `Kisan_Vaani`; import `farmers.csv` into a tab named `Farmers` and `advisory_rules.csv` into a tab named `Advisory_Rules` (File → Import → Insert new sheet, then rename the tabs exactly).
3. In n8n (cloud or self-hosted): Workflows → Import from File → import both workflow JSONs.
4. Create credentials: Google Sheets OAuth2, Gmail OAuth2, Google Drive OAuth2 (guided sign-in), and one **Header Auth** credential named `Sarvam Header Auth` — Name: `api-subscription-key`, Value: your key. Attach them where nodes show warnings.
5. Replace `PASTE_YOUR_SHEET_URL` (both workflows) and `PUT_YOUR_EMAIL_HERE` (Gmail nodes) with your values.
6. For Workflow B: upload a short voice memo (wav/mp3, ≤25 s) to Google Drive and select it in the Download Memo node.
7. Open Workflow A → Execute workflow. Advisories arrive in the demo inbox; the sheet logs rule IDs and timestamps.

*Tip: to validate the four Sarvam APIs standalone before running n8n, import `kisan_vaani_postman_collection.json` into Postman, set the `sarvam_api_key` collection variable, and run requests 1–4.*

## Why Sarvam

Indic-native STT / TTS quality (including Kannada and Marathi); natural code-mixing; data residency in India for a government-adjacent sector; and a single vendor across LLM, translate, and voice. The same voice-automation stack is already proven in production for higher-compliance BFSI use cases (collections, loan-servicing), which de-risks an agri advisory rollout.

## Limitations & 90-day path to production

**PoC gaps (deliberate):** simulated WhatsApp/telephony delivery to a demo inbox; mock farmer register instead of a CRM / FPO database integration; sample advisory rules (production rules come from the customer's agronomy team, per crop stage and region); no consent / DND management; single fixed voice; two languages voiced (Konkani is translate-supported but has no Bulbul voice yet, so Goa farmers receive Marathi voice); one-way broadcast (no farmer replies); weather fetched per farmer (cache per district at scale).

**Rollout:**
- **Days 0–30 —** WhatsApp Business API pilot in 2 districts, real agronomy rule base, consent / DND capture, CRM integration replacing the sheet.
- **Days 31–60 —** telephony for feature-phone farmers (Exotel / Sarvam Samvaad / Twilio), IVR replay option ("press 1 to hear again"), per-district weather caching, listen-through analytics.
- **Days 61–90 —** inbound farmer voice questions (Sarvam STT + intent routing to agronomists), scale to 6+ languages, A/B advisory formats against an SMS control group to prove the lift.
