# Lorstudy — Definizione del progetto

> Assistente di studio con IA per materie umanistiche (italiano, storia, filosofia, latino…).
> Trasforma i materiali dello studente in riassunti, mappe concettuali, quiz, flashcard e linee del tempo.

---

## 1. Idea e obiettivo

Lo studente carica un materiale (PDF del libro, foto di appunti, testo incollato, link, audio della lezione) e Lorstudy lo trasforma in **strumenti di studio attivo**:

| Strumento | A cosa serve |
|---|---|
| **Riassunto** (breve / medio / esteso / per punti) | Capire il contenuto in fretta |
| **Mappa concettuale** | Vedere relazioni causa‑effetto, autori‑opere, eventi‑conseguenze |
| **Linea del tempo** (storia) | Collocare eventi e periodi |
| **Schede autore / opera** (italiano) | Vita, poetica, opere, temi, contesto |
| **Analisi del testo guidata** (poesia/prosa) | Parafrasi, figure retoriche, metrica, temi |
| **Flashcard** con ripetizione spaziata | Memorizzare date, definizioni, concetti |
| **Quiz** (risposta multipla, V/F, completamento, aperte con correzione IA) | Verificarsi |
| **Simulazione interrogazione** (chat/voce) | Allenarsi all'orale |
| **Glossario** | Termini chiave con definizioni |

**Principio guida:** l'IA lavora *solo sul materiale dello studente* (approccio RAG), così riduce gli errori e resta aderente al programma del docente.

## 2. Target

- Studenti di scuola secondaria (medie e superiori) e università (lettere, storia, filosofia).
- Uso principale: preparazione di verifiche, interrogazioni, esami, maturità.

## 3. Requisiti funzionali

### MVP (v0.1)
1. **Import materiali**: testo incollato, PDF, DOCX, immagini (OCR).
2. **Organizzazione**: Materie → Argomenti → Materiali.
3. **Generazione**: riassunto, mappa concettuale, flashcard, quiz a risposta multipla.
4. **Modifica**: ogni output è modificabile dallo studente.
5. **Studio flashcard** con algoritmo di ripetizione spaziata (SM‑2 o FSRS).
6. **Quiz** con punteggio e spiegazione delle risposte sbagliate.
7. **Esportazione**: PDF / Markdown / Anki (.apkg o CSV).

### v0.2
8. Linea del tempo interattiva, schede autore/opera, analisi del testo.
9. Domande aperte corrette dall'IA con feedback.
10. Simulazione d'interrogazione (chat).
11. Statistiche: argomenti deboli, streak, tempo di studio.

### v0.3+
12. Audio lezione → trascrizione → riassunto.
13. Modalità voce per l'interrogazione.
14. Condivisione set di flashcard/quiz tra compagni, classi del docente.
15. Piano di studio verso una data di verifica.

## 4. Requisiti non funzionali

- **Costo zero o quasi**: solo modelli e servizi con piano gratuito.
- **Affidabilità**: ogni risposta cita il passo del materiale da cui deriva; avviso se l'informazione non è presente nella fonte.
- **Privacy**: materiali dello studente privati; nessun uso per addestramento; conformità GDPR (utenti minorenni → attenzione al consenso).
- **Mobile‑first**, funzionante anche offline per il ripasso (PWA).
- **Lingua**: interfaccia e prompt in italiano.
- **Accessibilità**: font leggibili, modalità dislessia (OpenDyslexic, sintesi vocale).

## 5. Architettura proposta

```
┌──────────────┐     HTTPS     ┌──────────────────┐     ┌──────────────────────┐
│  Frontend    │ ────────────▶ │  Orchestratore IA│ ──▶ │  Provider LLM gratuiti│
│  PWA (Next.js│               │  n8n (webhook)   │     │  Gemini / Groq /      │
│  o React)    │ ◀──────────── │  oppure API      │     │  OpenRouter free /    │
└──────┬───────┘   JSON        │  serverless      │     │  Ollama locale        │
       │                       └────────┬─────────┘     └──────────────────────┘
       ▼                                ▼
┌──────────────────────────────────────────────┐
│ Supabase (free): Auth, Postgres, Storage,    │
│ pgvector per la ricerca nei materiali (RAG)  │
└──────────────────────────────────────────────┘
```

### Stack consigliato
| Livello | Scelta | Perché |
|---|---|---|
| Frontend | **Next.js + Tailwind + shadcn/ui**, PWA | Gratuito su Vercel, mobile‑friendly |
| Mappe | **React Flow** o **Mermaid / Markmap** | L'IA genera JSON/Markdown → render automatico |
| Timeline | **vis-timeline** | Linee del tempo interattive |
| Backend/DB | **Supabase** (piano free) | Auth, DB, storage file, pgvector |
| Orchestrazione IA | **n8n** (self‑host gratuito, es. su Railway/Render/VPS) | Workflow visuali, facile da modificare |
| OCR | **Tesseract** (locale) o Gemini vision | Foto di appunti/libri |
| Trascrizione | **Groq Whisper** (free tier) | Audio delle lezioni |

### IA gratuite consigliate
| Provider | Modello | Uso ideale |
|---|---|---|
| **Google AI Studio** | Gemini Flash | Contesto molto lungo (capitoli interi), PDF e immagini → **scelta principale** |
| **Groq** | Llama / Qwen, Whisper | Velocissimo: quiz, flashcard, chat interrogazione, trascrizione |
| **OpenRouter** (modelli `:free`) | vari | Fallback se gli altri raggiungono i limiti |
| **Ollama** (locale) | Qwen / Mistral / Gemma | Zero costi e privacy totale, se hai un PC adatto |
| **Embeddings** | Gemini embeddings o `nomic-embed` via Ollama | RAG sui materiali |

> I limiti dei piani gratuiti cambiano spesso: prevedere **fallback automatico** tra provider nel workflow.

### n8n vs alternativa
- **n8n** è un'ottima scelta per iniziare: un webhook per ogni strumento, facile da iterare sui prompt.
- Alternativa senza server da gestire: **API Routes di Next.js + Vercel AI SDK**, che supporta Gemini/Groq/OpenRouter con lo stesso codice. Consiglio: partire con n8n per prototipare i prompt, poi spostare in codice i flussi stabili.

## 6. Workflow IA (n8n)

**Workflow "Ingest"** — `POST /webhook/ingest`
1. Ricevi file → estrai testo (PDF parser / OCR).
2. Pulisci e dividi in blocchi (~1000 token, sovrapposizione 150).
3. Calcola embeddings → salva in Supabase (`chunks`).
4. Genera automaticamente titolo, materia rilevata, parole chiave.

**Workflow "Genera"** — `POST /webhook/generate { materialId, tipo, opzioni }`
1. Recupera i blocchi rilevanti (o tutto il materiale se entra nel contesto).
2. Scegli il prompt in base a `tipo` (riassunto, mappa, quiz, flashcard, timeline…).
3. Chiama l'LLM chiedendo **output JSON con schema fisso**.
4. Valida il JSON (se invalido → riprova / fallback su altro provider).
5. Salva e restituisce al frontend.

**Workflow "Correggi"** — risposte aperte e interrogazione: confronta la risposta dello studente con la fonte, restituisce voto, punti mancanti, suggerimenti.

### Esempi di schema di output
```json
// flashcard
{ "cards": [ { "fronte": "Anno della pace di Westfalia?", "retro": "1648", "fonte": "p. 42", "tag": ["storia","'600"] } ] }

// quiz
{ "domande": [ { "testo": "...", "opzioni": ["A","B","C","D"], "corretta": 2, "spiegazione": "...", "difficolta": "media" } ] }

// mappa concettuale
{ "nodi": [ { "id": "1", "label": "Rivoluzione francese" } ],
  "archi": [ { "da": "1", "a": "2", "relazione": "causa" } ] }

// timeline
{ "eventi": [ { "data": "1789-07-14", "titolo": "Presa della Bastiglia", "descrizione": "..." } ] }
```

## 7. Modello dati (essenziale)

- `users` — profilo, livello scolastico
- `subjects` — materia (Italiano, Storia…)
- `topics` — argomento (es. "Leopardi", "Prima guerra mondiale")
- `materials` — file/testo sorgente, testo estratto
- `chunks` — blocchi + embedding
- `outputs` — tipo, contenuto JSON, versione
- `flashcards` — fronte, retro, stato ripetizione spaziata (intervallo, scadenza, facilità)
- `quiz_attempts` — risposte, punteggio, data

## 8. Caratteristiche distintive (rispetto a ChatGPT generico)

1. **Specializzazione umanistica**: schede autore, analisi del testo, timeline, rapporti causa‑effetto.
2. **Fedeltà alla fonte** con citazione della pagina.
3. **Ciclo completo**: capire (riassunto/mappa) → memorizzare (flashcard) → verificare (quiz/interrogazione).
4. **Gratis**, basata su provider con piani gratuiti.
5. **Pensata per la scuola italiana** (lessico, programmi, maturità).

## 9. Roadmap

| Fase | Durata indicativa | Risultato |
|---|---|---|
| 0. Prompt lab | 1 sett. | Workflow n8n con prompt testati su materiali reali |
| 1. MVP | 3–4 sett. | Import, riassunto, mappa, flashcard, quiz, ripasso |
| 2. Umanistica+ | 3 sett. | Timeline, schede autore, analisi testo, domande aperte |
| 3. Orale | 2 sett. | Simulazione interrogazione, audio |
| 4. Social/docenti | — | Condivisione, classi |

## 10. Rischi e mitigazioni

| Rischio | Mitigazione |
|---|---|
| Allucinazioni (date, autori sbagliati) | RAG + citazioni + segnalazione errori da parte dell'utente |
| Limiti dei piani gratuiti | Fallback multi‑provider, cache degli output, generazione su richiesta |
| Copyright dei libri caricati | Materiali privati per utente, nessuna condivisione pubblica dei testi |
| Utenti minorenni | Consenso genitori, nessun dato inviato a provider che lo usano per addestramento dove possibile |

## 11. Domande aperte

- Il repository **Lordoc** e la documentazione citata non sono presenti in questa sessione: da integrare per allineare stack, stile e convenzioni.
- Web app, app mobile nativa o entrambe?
- n8n self‑hosted (dove?) o n8n Cloud?
- Solo uso personale o pubblico (impatta auth, privacy, costi)?
