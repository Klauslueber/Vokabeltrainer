# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projekt

Englisch-Vokabeltrainer für deutschsprachige Nutzer. Single HTML file mit eingebettetem CSS und JavaScript — kein Build-Tool, kein Framework, keine Dependencies.

**Hosted on GitHub Pages:** `https://klauslueber.github.io/Vokabeltrainer/`
**GitHub Repo:** `https://github.com/Klauslueber/Vokabeltrainer`

## Dateien

- `index.html` — Produktionsdatei, wird auf GitHub Pages deployt; immer diese bearbeiten
- `vokabeltrainer.html` — ältere Kopie mit hartkodiertem VOCAB-Array; **nicht mehr aktuell**, wird nicht synchronisiert
- `Vokabeln Englisch.rtf` — ursprüngliche Vokabeldaten (85 Einträge) aus macOS Notes; historisch, Daten leben in Supabase

## Entwicklung & Deployment

```bash
# Lokal testen: index.html direkt im Browser öffnen
# Internet für Supabase erforderlich; Spracherkennung: Chrome oder Safari

# Deployen
git add index.html
git commit -m "..."
git push   # GitHub Pages deployt automatisch (~1-2 Minuten)
```

## Architektur

### 4 Trainingsphasen pro Wort

| Phase | Karte | Funktion |
|-------|-------|----------|
| 1 Aussprache | `card1` | TTS via `SpeechSynthesis` → `pronounce()` |
| 2 Übersetzung | `card2` | Tippfeld, Fuzzy-Match → `checkTranslation()` |
| 3 Sprechen | `card3` | 2 Sätze via `SpeechRecognition` + editierbare Textareas |
| 4 Feedback | `card4` | Claude API oder lokales Keyword-Matching → `showFeedback()` |

Navigation: `goPhase(n)` blendet Karten und Dot-Indikatoren um, scrollt nach oben.

### State & Persistenz

`localStorage`:
- `vt_key` — Claude API-Key
- `vt_progress` — `{ queue, qIdx, useAPI, apiKey }` — wird bei jedem Wort gespeichert (`saveProgress()`)

Globale Variablen: `VOCAB[]`, `queue[]` (shuffled Indices), `qIdx`, `voc` (aktuelle Vokabel), `sentences[2]`

### Supabase

- **URL:** `https://duzfmlkjrfvkklvshptf.supabase.co`
- **Tabelle:** `vocab` — `id, word, translations TEXT[], example, key, hint, created_at`
- **RLS:** Public read/insert/update/delete (Anon-Key im HTML, kein Secret)
- Daten werden bei `init()` via PostgREST API geladen; `VOCAB` ist danach befüllt

### Claude API (Feedback-Phase)

- Modell: `claude-haiku-4-5`
- Direkter Browser-Call mit Header `anthropic-dangerous-direct-browser-access: true`
- Fallback auf `evaluateLocally()` wenn kein API-Key oder API-Fehler
- Lokales Matching: alle Tokens aus `voc.key` müssen im Satz vorkommen

## Wichtige JS-Funktionen

- `init()` — Einstiegspunkt: Speech-Support, Supabase laden, gespeicherten Fortschritt wiederherstellen
- `loadWord()` — UI zurücksetzen, `goPhase(1)` aufrufen
- `goPhase(n)` — Karte n (1–4) einblenden, Dots aktualisieren, nach oben scrollen
- `checkTranslation()` — Fuzzy-Match via `normalize()` (Umlaute → ASCII, Sonderzeichen entfernen)
- `startRecording(n)` — `SpeechRecognition` starten/stoppen; bei Stopp: `captureFromTranscript()` aufrufen
- `confirmFromTranscripts()` — liest Sätze aus `#transcript1/2` Textarea-`.value` (iOS-sicher)
- `onTranscriptInput(n)` — live `has-text`-Klasse aktualisieren; Satz-2-Button freischalten
- `pronounce()` — bevorzugt Stimmen Samantha/Alex/Daniel/Karen; bereinigt `[...]`, `sb.`, `sth.`

## iOS-Besonderheiten

- `SpeechRecognition.onend` liefert auf iOS keine zuverlässigen Ergebnisse → Sätze immer via `.value` aus Textarea-DOM-Elementen lesen, nie aus `finalText`-Variable allein
- Transcript-Felder sind `<textarea>` (editierbar nach Aufnahme) — `.value` verwenden, nicht `.textContent`
- „Add to Home Screen" funktioniert nur über HTTPS (GitHub Pages), nicht über `file://`

## VOCAB-Format

```js
{ w: "word", de: ["bedeutung1", "bedeutung2"], ex: "Example sentence.", key: "word", hint?: "aussprache-tipp" }
```

`key` ist die Whitelist für lokales Keyword-Matching (einzelne Tokens, die alle im Satz vorkommen müssen).
