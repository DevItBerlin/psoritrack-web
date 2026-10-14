> **ARCHIVIERT (2026-10-14).** Dieses Repo ist nach `DevItBerlin/psoritrack` übernommen:
> Mockups samt Historie unter `docs/mockup/`, Website unter `website/` (gehostet bei netcup).
> Hier wird nichts mehr geändert.

# psoritrack-web – Claude Code Project Guide

Zwei Dinge liegen in diesem Repo:

1. **Die öffentliche Website** (GitHub Pages, `DevItBerlin/psoritrack-web`) –
   `index.html`, `datenschutz.html`, `impressum.html`, `style.css`, `assets/`.
2. **Die UI-Mockups** für die PsoriTrack-App – `mockups/nav-redesign-v<N>.html`.
   Hier passiert der Großteil der Arbeit.

Die **App selbst** liegt in einem anderen Repo: `..\psoritrack` (React Native + Expo,
FastAPI, Supabase) mit eigener `CLAUDE.md`. Änderungen hier fließen dort **nicht**
automatisch ein – die Mockups sind Entwurf, keine Implementierung.

> **Nicht verwechseln:** Der Ordner `..\Symptom Tracker\` enthält eine längst
> aufgegebene Vorstufe (`psoriasis-app-prototype.html`, `psoriasis-app-v2.html`) und
> eine veraltete `CLAUDE.md`. Deren Anweisungen gelten hier nicht.

---

## Mockups

- **Eine Datei pro Version**, self-contained: ein `<style>`-Block, ein `<script>`-Block,
  keine Build-Schritte, keine externen Abhängigkeiten. Öffnen = Doppelklick im Browser.
- **Aktuell ist die höchste Versionsnummer** (`ls mockups/nav-redesign-v*.html`).
  Der `<title>` im File wird nicht zuverlässig mitgezogen – Dateiname und `git log`
  sind die Wahrheit, nicht der Titel.
- **Varianten:** `v<N>a` / `v<N>b` sind parallele Entwürfe derselben Version (meist
  einer pro Entwickler), `v<N>merge` bzw. ein Commit „Merge aus vX und vY" führt sie
  wieder zusammen. Die Linie ist also nicht streng linear.
- **Neue Version = neue Datei**: vorherige Version kopieren, dann ändern. Bestehende
  Versionen nicht nachträglich umschreiben, außer der Nutzer sagt es ausdrücklich –
  sie sind der Diskussionsstand mit Florian.
- **UI-Sprache ist Deutsch** (Umlaute im HTML normal, nur in Commit-Messages
  transliteriert).
- **Demo-Daten** stehen als `const` in GROSSBUCHSTABEN oben im Script-Block
  (`SERIES`, `FACTORS`, `AMP_LEVEL`, `FOODS` …). Das sind Platzhalter, keine Logik –
  siehe Ticket unten.

## Tickets

`tickets/` sammelt Backend-Arbeit, die aus den Mockups folgt, aber noch **nicht**
umgesetzt wird. Offen: `tickets/ampel-scoring-engine.md` – die regelbasierte
Ampel-Engine, die die Platzhalter `SERIES`/`AMP_LEVEL`/`ampStep()` ersetzen soll.
Nicht implementieren, bevor die Mockups final abgenommen sind.

## Website

`index.html` verlinkt die Mockups **nicht** – sie sind zwar im Repo und damit über
GitHub Pages erreichbar, aber nicht Teil der öffentlichen Navigation. Beim Bearbeiten
der Mockups die Seiten-Dateien nicht anfassen (und umgekehrt).
`style.css` wird per Query-String versioniert (`style.css?v=YYYYMMDD-N`) – bei
CSS-Änderungen den Wert hochzählen, sonst greift der Browser-Cache.

## Git

- **Alles geht direkt auf `main`** – keine Feature-Branches, `main` ist die komplette
  Historie. Kein PR-Prozess.
- **Vorher `git fetch`**: Florian pusht von einem anderen Rechner, das Remote ist oft
  voraus. Bei Divergenz **rebasen, nie force-pushen**.
- **Commit-Stil:** `mockups: v<N> - <deutsche Zusammenfassung, Umlaute als ue/ae/oe>`.
- **Nur die Dateien committen, die zur Aufgabe gehören.** Es liegt dauerhaft eine
  untracked `mockups/Design PowerPoint Mockup.pptx` herum – die bleibt untracked.
- Fertige Arbeit wird committet und gepusht, ohne jedes Mal zu fragen. Nicht ohne
  Rückfrage: force push, History-Rewrite, Remote-Refs löschen.
