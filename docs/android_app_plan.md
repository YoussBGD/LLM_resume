# Android Lecture Companion App Plan

## Goal
Build an Android application that can record lectures even when the screen is off, transcribe the audio accurately, extract text from lecture PDFs (including scanned images), and summarize the combined content using an LLM (GPT-5 Mini via user-provided API key). The app should gracefully handle permissions, power management, and process restarts.

## Core User Journeys
1. **Lecture capture**
   - User opens the app, grants microphone and storage permissions.
   - Starts a recording session that continues in the background with a persistent notification.
   - Recording automatically pauses/resumes on phone calls or headset removal.
   - User stops the session and receives a transcription plus summary.
2. **Lecture + slides workflow**
   - User selects an existing recording and attaches a lecture PDF.
   - App extracts text (with OCR fallback) per page and merges it with the transcription.
   - User receives a structured summary with section highlights and key concepts.

## Feature Breakdown
- **Background audio recorder**: Foreground service with MediaRecorder or AudioRecord, persistent notification, restart via `WorkManager` if killed.
- **Transcription pipeline**: On-device buffering to FLAC/WAV, upload to Whisper API (or local Whisper.cpp) for accurate transcription. Support chunked processing for long recordings and display progress.
- **PDF ingestion**: Use `PdfRenderer` for native text extraction; integrate ML Kit Text Recognition or Tesseract for OCR on image-only pages. Cache per-page results with Room database.
- **Summarization service**: Kotlin coroutine that composes the combined transcript and PDF text, sends prompts to GPT-5 Mini via HTTPS (user API key stored securely with EncryptedSharedPreferences).
- **Result presentation**: Compose UI showing timeline, searchable transcript, per-page notes, and generated summary with sections (Overview, Key Points, Action Items, Questions).
- **Resilience & energy**: Foreground service to avoid Doze kill, schedule periodic saves, handle low-battery events, and resume encoding after process death using `WorkManager` and `ForegroundServiceStartNotAllowedException` fallbacks.
- **Permissions & onboarding**: Guided flow explaining why microphone, storage, notification, and battery optimization exemptions are required.

## High-Level Architecture
- **Presentation layer**: Jetpack Compose screens (Home, Recording, Session Detail, Settings). Navigation Compose handles flow.
- **Domain layer**: Use cases for starting/stopping recordings, processing PDFs, running summaries. Kotlin sealed results for success/error.
- **Data layer**: Repositories encapsulating audio storage, transcription API, PDF parsing/OCR, summary API, and session metadata stored in Room.
- **Background work**: Foreground service for active recordings, `WorkManager` for long-running processing (transcription, OCR, summarization).
- **Dependency injection**: Hilt for providing repositories, API clients, and settings.

## Data Model Sketch
- `RecordingSession`: id, title, createdAt, audioFilePath, duration, status.
- `TranscriptSegment`: sessionId, startTime, endTime, text, confidence.
- `PdfDocument`: sessionId, filePath, pageCount.
- `PageExtraction`: pdfId, pageIndex, rawText, ocrConfidence.
- `Summary`: sessionId, combinedTextHash, overview, bulletPoints, actionItems, jsonMetadata.
- `UserSettings`: apiKey, transcriptionMode, batteryOptimizationsEnabled.

## External Integrations
- Whisper or alternative speech-to-text API.
- GPT-5 Mini summarization endpoint (HTTP client with retries, exponential backoff, streaming support).
- Local secure storage via Android Keystore + EncryptedSharedPreferences for API keys.

## Security & Privacy Considerations
- Explicit user consent for recordings and uploads.
- Local encryption of stored audio and transcripts.
- Allow users to delete sessions and purge remote data.
- Offline mode when no network; queue uploads when connectivity returns.

## Testing Strategy
- Unit tests for use cases and repositories with Fake implementations.
- Instrumented tests for recording service lifecycle, permission flows, and PDF parsing.
- Snapshot tests for Compose UI states.
- End-to-end smoke test to ensure recording → transcription → summary pipeline works on real device.

## Milestones
1. **MVP Recording**: Foreground service, simple UI, manual transcription trigger.
2. **PDF & OCR**: Add PDF ingestion, per-page extraction, caching.
3. **Summaries & UX polish**: LLM integration, summary presentation, offline handling.
4. **Hardening**: Battery optimization guidance, crash recovery, analytics/telemetry (if desired).

## Open Questions
- Final choice of transcription backend (latency vs cost).
- Target minimum Android API level (suggest 26+ for modern APIs).
- Localization requirements and accessibility (TalkBack, captions).

## Next Steps
- Validate API budget for transcription + LLM usage.
- Prototype recording service on reference devices to measure power impact.
- Evaluate OCR accuracy on sample course PDFs.
- Draft prompt templates for GPT-5 Mini summarization.
