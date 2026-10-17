CLAUDE.md
This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Project Overview
ShramMitra AI — A multilingual, voice-first assistant for gig and informal workers (delivery partners, drivers, domestic workers, daily-wage workers). Provides GenAI-powered guidance on earnings tracking, safety, welfare schemes (e-Shram, PMJAY), document help, and grievance drafting in simple Indian languages.

Core principle: Generative AI is the reasoning layer, not the authority. All monetary calculations and eligibility checks run on deterministic code; the LLM only interprets, explains, and drafts.

Tech Stack
Layer	Technology
Frontend	Flutter (Android-first, iOS secondary)
Backend	Firebase Cloud Functions (Dart)
Database	Cloud Firestore
File storage	Firebase Cloud Storage (requires Blaze plan)
Voice (STT/TTS)	Sarvam AI — Saaras (STT), Bulbul (TTS)
LLM	Sarvam-105B or OpenAI-compatible model
Shared logic	Dart package packages/shared/ (used by both app and functions)
Primary channel: WhatsApp (via WhatsApp Business API). The mobile app is a companion client for advanced operations (ledger history, draft storage, document upload).

Repository Structure
text
shram_mitra/
├── CLAUDE.md                    # This file
├── packages/
│   └── shared/                  # Dart shared package: models + deterministic logic
│       ├── lib/src/models.dart
│       ├── lib/src/ledger.dart  # Wage calculation — MUST be deterministic
│       └── lib/src/templates.dart  # Grievance draft templates
├── functions/                   # Firebase Cloud Functions (Dart)
│   ├── bin/server.dart          # Entry point
│   └── lib/src/
│       ├── sarvam_client.dart   # STT/TTS/LLM API wrappers
│       ├── voice_handler.dart   # Voice message processing
│       ├── grievance.dart       # Draft generation
│       └── rag.dart             # Scheme knowledge base retrieval
├── app/                         # Flutter app
│   └── lib/
│       ├── main.dart
│       ├── screens/             # Home, ledger, grievance, scheme Q&A
│       └── services/            # Cloud Functions callers
└── knowledge/                   # RAG source documents (scheme rules, legal sections)
Commands
Flutter App
bash
cd app
flutter pub get
flutter run                     # Run on connected device
flutter test                    # Run unit tests
flutter build apk --release     # Build Android release
flutter analyze                 # Lint check
Firebase Functions (Dart)
bash
cd functions
dart pub get
dart run bin/server.dart        # Run locally
firebase emulators:start        # Start local emulator suite
firebase deploy --only functions
Shared Package
bash
cd packages/shared
dart pub get
dart test
Full deploy
bash
firebase deploy --only functions,firestore:rules,storage:rules
Critical Rules — Do Not Violate
1. Ledger-first money rule (NON-NEGOTIABLE)
The LLM must never generate a rupee amount. Every monetary figure shown to the worker must originate from packages/shared/lib/src/ledger.dart.

Correct pattern:

dart
final result = WageLedger(rate: 800, days: 5, paid: 3000).calculate();
// result.owed == 1000 — this is the only allowed source of ₹1000
final prompt = "Explain to the worker in Hindi that they are owed ₹${result.owed}. "
               "Do not compute or change this number.";
If the LLM response contains a rupee figure not present in the ledger output, the response MUST be rejected and regenerated from ledger facts only.

2. Citation discipline
Every scheme-related answer must cite its source (PIB release, Karnataka Act section, e-Shram portal). If RAG retrieval confidence is low, respond with:

"I don't have verified information on this. Please call the labour helpline 1800-425-1200."

Never fabricate a scheme rule, deadline, or eligibility criterion.

3. Draft disclaimer
Every generated document (grievance letter, work agreement, wage demand notice) must include:

"यह एक मसौदा है, कानूनी सलाह नहीं। गंभीर विवाद के लिए वकील या श्रम हेल्पलाइन से संपर्क करें।"
("This is a draft, not legal advice. For serious disputes, contact a lawyer or the labour helpline.")

4. Escalation triggers
If the worker's message mentions threat, assault, non-payment below minimum wage, or medical emergency, the response MUST prioritize emergency contacts (police 112, labour helpline, NGO partners) over the standard grievance path. Do not bury these under procedural steps.

5. API key handling
The Sarvam API key lives in Firebase Functions environment config, never in the Flutter client. All Sarvam calls go through Cloud Functions. If real-time voice streaming is added later, use short-lived tokens issued by Functions.

6. e-Shram limitation
There is no public e-Shram API. The app can only:

Guide workers step-by-step through manual registration (e-Shram portal, UMANG, CSC/SSK centres)

Ask the worker to report their UAN if they have one

Display PMJAY eligibility information (which is linked to e-Shram status)

Do not write code that pretends to query e-Shram status directly. Do not scrape the e-Shram portal.

Architecture Notes
Voice flow
text
Worker records audio (Flutter)
  → upload base64 to Cloud Function `processVoice`
  → Sarvam STT (Saaras, mode="codemix", language_code=worker's language)
  → Intent extraction (LLM with function-calling)
  → Branch:
      ├─ wage_calc  → shared WageLedger (deterministic)
      ├─ grievance  → template + LLM fill
      ├─ scheme_q   → RAG retrieval + LLM answer
      └─ safety     → curated checklist + helpline numbers
  → Sarvam TTS (Bulbul, worker's language)
  → return { transcript, replyText, replyAudio } to Flutter
Shared package purpose
packages/shared/ exists so wage calculation logic is written once and used by both the Flutter app (for instant preview) and Cloud Functions (for authoritative result). Never duplicate ledger logic in either the app or functions.

RAG knowledge base
Source documents live in knowledge/. Curate from:

e-Shram / PMJAY official FAQs (PIB releases)

Code on Social Security 2020 (gig/platform worker sections)

Karnataka Platform Gig Workers Act 2025 (Section 22 — grievance timelines: IDRC 14-day action report, 45-day final order)

Central Social Security Rules 2026

State labour helpline directories

Update knowledge/ when rules change; re-index via the rag.dart ingestion routine.

Language support
Initial languages: Hindi, Marathi, Tamil, Telugu, Bengali. Add more by extending the supportedLanguages list in packages/shared/lib/src/models.dart and verifying Sarvam STT/TTS support for the language code.

Development Conventions
Dart style
Follow dart format (default 80-col) and flutter analyze clean

Prefer final over var; use const constructors where possible

Models: immutable classes with copyWith, fromJson, toJson

Errors: throw typed exceptions from shared package (LedgerException, GrievanceException); catch and translate to worker-friendly messages in the function layer

Naming
Files: snake_case.dart

Classes: PascalCase

Functions/variables: camelCase

Firestore collections: snake_case plural (workers, ledgers, grievance_drafts)

Firestore data model
text
workers/{workerId}
  - phoneHash: string       # hashed, not raw phone
  - preferredLanguage: string
  - eshramUan: string?      # only if worker provides it
  - createdAt: timestamp

workers/{workerId}/ledgers/{ledgerId}
  - employer: string
  - rate: number
  - days: number
  - paid: number
  - owed: number            # computed by WageLedger, stored for audit
  - createdAt: timestamp

workers/{workerId}/drafts/{draftId}
  - type: "grievance" | "agreement" | "demand_notice"
  - text: string
  - language: string
  - filed: boolean
  - createdAt: timestamp
Firestore security rules must restrict every workers/{workerId}/** path to request.auth.uid == workerId.

Testing
Shared package: unit tests for WageLedger covering rate × days, partial payment, rounding, zero/negative inputs

Functions: emulator-based integration tests for the voice flow with mocked Sarvam responses

Manual: test with real workers in real environments (roadside, noisy), not just in a quiet office

What Claude Should NOT Do
Do not generate rupee amounts in LLM prompts or responses — always source from WageLedger

Do not add direct e-Shram or government API calls — no public API exists

Do not hardcode the Sarvam API key in any client-side code

Do not skip the draft disclaimer on any generated document

Do not treat safety/emergency messages as routine grievances

Do not add languages without verifying Sarvam STT/TTS support for that language code

Do not commit firebase_options.dart, .env, or any file containing API keys

Do not promise real-time voice streaming unless the Sarvam WebSocket SDK path has been explicitly implemented and tested

Useful References
Sarvam AI docs: docs.sarvam.ai

Firebase Functions (Dart): firebase.google.com/docs/functions (Dart support via firebase_functions package)

e-Shram portal: eshram.gov.in (manual registration only)

Karnataka IPGRS: state gig worker grievance portal (link surfaced to workers, not called by the app)

National labour helpline: 1800-425-1200

Open Questions / Decisions Pending
□ WhatsApp channel: build first via py-whatsapp-cloudbot or use a BSP? (Decision affects whether Functions need a webhook handler in addition to the callable processVoice)
□ Real-time voice streaming via Sarvam WebSocket SDK: needed for MVP, or is record-and-respond acceptable? (Latency vs. complexity trade-off)
□ Blaze plan upgrade timing: Cloud Storage requires it; confirm budget before MVP launch
□ Legal review: all scheme/legal content in knowledge/ must be reviewed by a licensed Indian lawyer before public launch
□ Partnership path for e-Shram data: explore aggregator or NGO partner status to potentially access aggregated (not individual) e-Shram data in the future
