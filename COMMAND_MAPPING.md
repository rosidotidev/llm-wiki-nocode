# LLM Wiki Command Mapping

## Command Invocation Rules

Quando l'utente scrive una richiesta, determino automaticamente quale comando eseguire:

### /wiki-ingest
**Esegui quando l'utente chiede di:**
- Ingestire un documento (`"ingestisci raw/file.md"`)
- Aggiungere sorgenti alla wiki
- Processare file approvati
- Resettare la wiki (`"--RESET-ALL"`)
- "Scansione" automatica di nuovi file

**Non fare domande.** Procedi con auto-scan (Step 0) oppure il file specificato (Steps 1-8).

### /wiki-query
**Esegui quando l'utente:**
- Fa una domanda sulla wiki (`"Come si dichiara un agent mfa?"`)
- Chiede informazioni presenti nella wiki
- Dice "rispondi da wiki" o "cosa dice la wiki su..."

**Non usare training data.** Leggi solo wiki pages. Min 3, max 8 pagine. Cita tutto con [[wikilinks]].

### /wiki-lint
**Esegui quando l'utente chiede di:**
- Controllare la salute della wiki (`"lint della wiki"`, `"controllo qualità"`)
- Trovare broken links, pagine orfane, contraddizioni
- Verificare la validità della wiki

**Phase 1 prima (deterministic)**, poi Phase 2 (semantic). Salva report in lint_pending/.

---

## Auto-Detection Logic

Se l'utente scrive qualcosa di ambiguo, usa il contesto per decidere:
- Verbo "ingestire", "aggiungere", "processare" → /wiki-ingest
- Domanda diretta, richiesta info → /wiki-query  
- Verbo "controllare", "verificare", "lint" → /wiki-lint

Non chiedere conferma. Esegui il comando, mostra i risultati.
