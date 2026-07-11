# sarvam-ai-farmer-advisory-workflow

# Kisan Vaani — Multilingual Voice Advisory Agent for Farmers

**One agronomist's table of rules + live weather + Sarvam's full voice stack → the right advisory, to the right farmer, in the right language, over the right channel — every morning, automatically.**

Demo video: **[PASTE YOUR LOOM LINK]**

---

## The problem

Agri-input companies, FPOs, and state agriculture departments need to reach millions of farmers with time-critical guidance — spray windows, irrigation calls, weather warnings. Today that means mass SMS in English/Hindi (which many farmers can't read), or expensive field-officer visits and call-center agents. Advice that arrives late, in the wrong language, or as unreadable text is advice wasted.

## What this agent does

Every morning, on schedule:

1. Reads the farmer register (mock CRM — Google Sheet; **phone number is the farmer's identity**, as in the real world).
2. Fetches the **live 48-hour weather forecast** for each farmer's district (Open-Meteo API).
3. **Reasons**: classifies the weather condition (heavy rain / hot-dry / normal) and picks the single matching row from the expert-curated **advisory rules table** for that farmer's crop. It never invents agronomy — it only localizes expert content. Every message logs the rule ID used, so it's fully auditable.
4. Composes a short, warm, spoken-style advisory (Sarvam **sarvam-105b**).
5. Localizes it (Sarvam **Translate / Mayura**) into the farmer's language — Kannada and Marathi in this demo.
6. Voices it (Sarvam **TTS / Bulbul v3**) as a natural voice note.
7. Delivers it per the farmer's **channel** — WhatsApp voice note for smartphone users, outbound voice call for feature-phone users — and logs rule, timestamp, and preview back to the register.

A second workflow handles **campaign broadcasts**: a field officer records one ~20-second voice memo; Sarvam **STT (Saaras v3)** transcribes it, and the same pipeline localizes and voices it for every farmer. One spoken sentence reaches the entire base in their own languages.

### Note on delivery in this PoC

The farmer register contains phone numbers only (no emails — farmers don't have them). In this proof-of-concept, deliveries are routed to a **demo inbox** with the subject line declaring the simulated channel and target phone (e.g. `[SIMULATED WHATSAPP → +91-98XXX…]`). Production delivery is WhatsApp Business API and telephony (with TRAI/DND-compliant consent). As proof the phone channel works end-to-end, the demo includes a **real outbound call via Twilio** playing the generated Kannada advisory to a verified number — see the video at [TIMESTAMP].

## Architecture

![Architecture](docs/architecture.png)

## Sarvam APIs used, and why they're core

| API | Where | Why it's load-bearing |
|---|---|---|
| Chat Completions (sarvam-105b) | Advisory composition | Classifies weather condition, selects the rule, writes spoken-style copy — remove it and there's no message |
| Translate (mayura:v1) | Localization | Kannada/Marathi output that keeps natural code-mixed terms farmers actually use |
| Text-to-Speech (bulbul:v3) | Voice notes | The product IS voice — literacy-independent delivery |
| Speech-to-Text (saaras:v3) | Campaign workflow | Lets a non-technical field officer feed the system by simply speaking |

## Repo map

```
/src
  workflow_A_daily_advisory.json   ← main agent (import into n8n)
  workflow_B_campaign.json         ← STT campaign broadcast (import into n8n)
  kisan_vaani_postman_collection.json  ← every API call, runnable standalone
  farmers.csv / advisory_rules.csv ← mock CRM data (import into Google Sheets)
  tts_player.html                  ← offline player for Sarvam TTS base64 audio
  .env.example
/docs
  business_writeup.pdf
  architecture.png
```

## Setup (≈15 minutes)

1. Get a Sarvam API key at dashboard.sarvam.ai (free signup credits).
2. Create a Google Sheet named `Kisan_Vaani`; import `farmers.csv` into a tab named `Farmers` and `advisory_rules.csv` into a tab named `Advisory_Rules` (File → Import → Insert new sheet, then rename tabs exactly).
3. In n8n (cloud or self-hosted): Workflows → Import from File → import both JSONs.
4. Create credentials: Google Sheets OAuth2, Gmail OAuth2, Google Drive OAuth2 (guided sign-in), and one **Header Auth** credential named `Sarvam Header Auth` — Name: `api-subscription-key`, Value: your key. Attach them where nodes show warnings.
5. Replace `PASTE_YOUR_SHEET_URL` (both workflows) and `PUT_YOUR_EMAIL_HERE` (Gmail nodes) with your values.
6. For Workflow B: upload a short voice memo (wav/mp3, ≤25 s) to Google Drive and select it in the Download Memo node.
7. Open Workflow A → Execute workflow. Advisories arrive in the demo inbox; the sheet logs rule IDs and timestamps.

## Limitations & 90-day path to production

**PoC gaps (deliberate):** simulated WhatsApp delivery; mock farmer register instead of a CRM/FPO database integration; sample advisory rules (production rules come from the customer's agronomy team, per crop stage and region); no consent/DND management; single fixed voice; no farmer replies.

**Rollout:** Days 0–30 — WhatsApp Business API pilot in 2 districts, real agronomy rule base, consent capture. Days 31–60 — telephony for feature-phone farmers (Exotel / Sarvam Samvaad), IVR replay option ("press 1 to hear again"), analytics on listen-through. Days 61–90 — inbound farmer voice questions (Sarvam STT + intent routing to agronomists), scale to 6+ languages, A/B advisory formats against SMS control group.
