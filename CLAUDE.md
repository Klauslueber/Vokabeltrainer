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
| 3 Sprechen | `card3` | 2 Sätze: Whisper API (wenn OpenAI-Key) oder Web `SpeechRecognition`; editierbare Textareas |
| 4 Feedback | `card4` | Claude API oder lokales Keyword-Matching → `showFeedback()`; bei Claude-Auswertung zusätzlich Synonyme |

Navigation: `goPhase(n)` blendet Karten und Dot-Indikatoren um, scrollt nach oben.

Die Wortkarte (`.word-card`, oberhalb aller vier Phasen-Cards) ist phasenübergreifend sichtbar und enthält den 🗑️-Button (`deleteCurrentWord()`) zum Löschen der aktuellen Vokabel.

### State & Persistenz

`localStorage`:
- `vt_key` — Claude API-Key (Feedback-Phase)
- `vt_openai_key` — OpenAI API-Key (Whisper-Spracherkennung); unabhängig von `vt_progress`
- `vt_progress` — `{ queue, qIdx, useAPI, apiKey }` — wird bei jedem Wort gespeichert (`saveProgress()`)

Globale Variablen: `VOCAB[]`, `queue[]` (shuffled Indices), `qIdx`, `voc` (aktuelle Vokabel), `sentences[2]`, `apiKey`/`useAPI` (Claude), `openaiKey`, `mediaRecorder`/`audioChunks[]`/`recognition` (Aufnahme-Backends), `isRecording`/`activeN` (welche Aufnahme läuft gerade), `speechSupported`

**Aufnahme-State:** `isRecording` + `activeN` bilden eine simple State-Maschine. Solange `isRecording`, stoppt ein erneuter `startRecording()`-Aufruf die laufende Aufnahme statt eine neue zu starten. Beide Backends (Whisper, Web Speech) müssen `isRecording` im Abschluss-Handler zurücksetzen, sonst bleibt die App „in Aufnahme" hängen.

**Vokabel löschen:** `deleteCurrentWord()` löscht per DELETE-Request in Supabase (`voc.id`) und muss `VOCAB` und `queue` konsistent halten: `queue` enthält Indices in `VOCAB`, daher werden nach `VOCAB.splice()` alle `queue`-Einträge > dem gelöschten Index um 1 dekrementiert (der gelöschte Index selbst wird herausgefiltert). `qIdx` bleibt unverändert — durch das Herausfiltern des aktuellen Eintrags zeigt `queue[qIdx]` danach automatisch auf das nächste Wort, ein `loadWord()` reicht.

**Wichtig:** Bei vorhandenem `vt_progress` überspringt `init()` den Setup-Screen komplett und springt direkt ins Training. Schlüssel können dann nur noch über den ⚙️-Button (`openSettings()`) gesetzt werden — der Setup-Screen ist nicht mehr erreichbar, bis das Training abgeschlossen ist oder `vt_progress` gelöscht wird.

### Supabase

- **URL:** `https://duzfmlkjrfvkklvshptf.supabase.co`
- **Tabelle:** `vocab` — `id, word, translations TEXT[], example, key, hint, created_at`
- **RLS:** Public read/insert/update/delete (Anon-Key im HTML, kein Secret)
- Daten werden bei `init()` via PostgREST API geladen; `VOCAB` ist danach befüllt
- **Mapping** (`loadVocabFromSupabase()`): `id→id`, `word→w`, `translations→de`, `example→ex`, `hint→hint`; fehlendes `key` fällt auf das erste Wort von `word` zurück (`word.toLowerCase().split(' ')[0]`). `id` wird für `deleteCurrentWord()` benötigt.
- **Neue Vokabel:** `saveNewVocab()` (Formular `newWord`/`newTranslations`/…) POSTet direkt in die Tabelle und ruft danach `loadVocabFromSupabase()` erneut auf; `translations` wird per Komma gesplittet
- **Vokabel löschen:** `deleteCurrentWord()` sendet `DELETE /vocab?id=eq.{voc.id}`, nach Bestätigung durch `confirm()`

### Claude API (Feedback-Phase)

- Modell: `claude-haiku-4-5`
- Direkter Browser-Call mit Header `anthropic-dangerous-direct-browser-access: true`
- Fallback auf `evaluateLocally()` wenn kein API-Key oder API-Fehler
- Lokales Matching: alle Tokens aus `voc.key` müssen im Satz vorkommen
- Der Prompt in `evaluateWithClaude()` fordert neben der Bewertung auch 3-5 Synonyme zu `voc.w`, markiert mit einer Zeile `SYNONYMS: ...`; die Antwort wird per Regex in `feedbackText`/`synonyms` gesplittet. Reihenfolge in `showFeedback()`: Bewertungstext → 📖 Beispielsatz (`voc.ex`) → 🔁 Synonyme. Nur im Claude-Modus verfügbar, `evaluateLocally()` liefert keine Synonyme.

### OpenAI Whisper (Sprechen-Phase)

- Modell: `whisper-1`, Endpoint `https://api.openai.com/v1/audio/transcriptions`, `language=en`
- `startRecording(n)` verzweigt: mit `openaiKey` → `startWhisperRecording()` (MediaRecorder + Whisper API), sonst → `startWebSpeechRecording()` (Web Speech API)
- MediaRecorder bevorzugt `audio/webm;codecs=opus`, fällt auf `audio/webm` bzw. `audio/mp4` zurück (Safari/iOS)
- Während des API-Calls zeigt der Button „⏳ Transkribiere …" und ist disabled
- **Invariante:** Der Whisper-`onstop`-Handler setzt `btn.disabled = true` während der Transkription. `resetRecBtn(n)` (im `finally`) muss `disabled` wieder auf `false` setzen — sonst bleibt der Aufnahme-Button beim nächsten Wort ausgegraut, weil `loadWord()` den Zustand nicht erneut aufhebt. (`recBtn2` ist absichtlich disabled, bis Satz 1 vorliegt.)

## Wichtige JS-Funktionen

- `init()` — Einstiegspunkt: Speech-Support, Supabase laden, gespeicherten Fortschritt wiederherstellen
- `loadWord()` — UI zurücksetzen, `goPhase(1)` aufrufen
- `goPhase(n)` — Karte n (1–4) einblenden, Dots aktualisieren, nach oben scrollen
- `checkTranslation()` — Fuzzy-Match via `normalize()` (Umlaute → ASCII, Sonderzeichen entfernen)
- `startRecording(n)` — Dispatcher; delegiert je nach `openaiKey` an `startWhisperRecording()` oder `startWebSpeechRecording()`; bei laufender Aufnahme: stoppen
- `openSettings()` — Prompt-Dialoge für Claude- und OpenAI-Key; einziger Weg, Keys zu setzen wenn `init()` direkt ins Training springt
- `confirmFromTranscripts()` — liest Sätze aus `#transcript1/2` Textarea-`.value` (iOS-sicher)
- `onTranscriptInput(n)` — live `has-text`-Klasse aktualisieren; Satz-2-Button freischalten
- `resetRecBtn(n)` — Aufnahme-Button optisch + funktional zurücksetzen (Text je nach `sentences[n-1]`, `disabled = false`)
- `pronounce()` — bevorzugt Stimmen Samantha/Alex/Daniel/Karen; bereinigt `[...]`, `sb.`, `sth.`
- `deleteCurrentWord()` — löscht die aktuelle Vokabel nach `confirm()` per Supabase-DELETE, hält `VOCAB`/`queue` konsistent (siehe Invariante oben) und lädt direkt das nächste Wort

## iOS-Besonderheiten

- `SpeechRecognition.onend` liefert auf iOS keine zuverlässigen Ergebnisse → Sätze immer via `.value` aus Textarea-DOM-Elementen lesen, nie aus `finalText`-Variable allein
- Transcript-Felder sind `<textarea>` (editierbar nach Aufnahme) — `.value` verwenden, nicht `.textContent`
- „Add to Home Screen" funktioniert nur über HTTPS (GitHub Pages), nicht über `file://`

## VOCAB-Format

```js
{ id: 1, w: "word", de: ["bedeutung1", "bedeutung2"], ex: "Example sentence.", key: "word", hint?: "aussprache-tipp" }
```

`key` ist die Whitelist für lokales Keyword-Matching (einzelne Tokens, die alle im Satz vorkommen müssen). `id` ist die Supabase-Row-ID, benötigt für `deleteCurrentWord()`.
