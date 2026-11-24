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
- **Privacy** (come Lordoc): nessun account, dati sul telefono; al provider IA si invia solo il testo da elaborare; consenso GDPR con UMP; attenzione agli utenti minorenni.
- **App Android nativa** (min Android 8.0, target API 36); ripasso di flashcard e quiz **offline**, IA solo per generare.
- **Lingua**: interfaccia e prompt in italiano.
- **Accessibilità**: font leggibili, modalità dislessia (OpenDyslexic, sintesi vocale).

## 5. Architettura (allineata a Lordoc Android)

Lorstudy riprende l'impostazione di **Lordoc Android** (simulatore CCNA): app **Android nativa in Java**,
API standard, nessun account, dati sul telefono, CI su GitHub Actions che produce l'APK, AdMob + UMP
per il consenso GDPR. Cambia il contenuto: invece di una banca domande fissa, i contenuti vengono
**generati dall'IA** a partire dai materiali dello studente.

```
┌──────────────────────────┐   HTTPS/JSON   ┌───────────────────────┐   ┌───────────────────────┐
│ App Android (Java)       │ ─────────────▶ │ Gateway IA            │──▶│ LLM gratuiti          │
│ - import PDF/foto/testo  │                │ A) n8n (webhook)      │   │ Gemini Flash (princ.) │
│ - engine quiz (da Lordoc)│ ◀───────────── │ B) chiamata diretta   │   │ Groq (veloce/Whisper) │
│ - flashcard SRS, mappe   │  formato .txt  │    con chiave utente  │   │ OpenRouter :free      │
│ - storico, statistiche   │  tipo Lordoc   └───────────────────────┘   └───────────────────────┘
└──────────────────────────┘
   dati locali: cartella privata app (come UserDataStore di Lordoc)
```

### Cosa si riusa da Lordoc Android
| Componente Lordoc | Uso in Lorstudy |
|---|---|
| `model/` (`Question`, `QuestionType`, `ExamSession`, `Response`) | Quiz: SINGLE, MULTI, FILL, DRAG (es. abbinare autore→opera, evento→secolo) |
| `data/QuestionParser` + formato `[Q]…[/Q]` | **Formato di scambio con l'IA**: l'LLM genera domande in questo formato, il parser le valida (già gestisce gli errori) |
| `engine/` (`ExamFactory`, `QuestionFilter`, `Scorer`) | Verifiche: filtro mai viste / sbagliate / marcate, punteggio in decimi invece di 300–1000 |
| `Domain` / `DomainScore` | Diventano **Materia → Argomento**, con accuratezza per argomento |
| Modalità **Studio** / **Simulazione** | Studio = spiegazione subito; Simulazione = "verifica in classe" a tempo |
| `ExamActivity`, `ResultsActivity`, `ReviewActivity`, `HistoryActivity`, `ScoreChartView` | Stesse schermate, testi adattati |
| `ReferenceActivity` (cheat sheet HTML) | Mostra **riassunti, schede autore e mappe** generati (HTML/Markdown in WebView) |
| `Ads`, `BaseActivity`, `Ui`, CI `build.yml`, `release.properties` | Identici |

### Nuovi componenti
- `ImportActivity`: testo, PDF (`PdfRenderer` + estrazione testo), foto (ML Kit OCR on‑device, gratuito).
- `AiClient`: chiama il gateway, riceve testo/JSON, retry e fallback tra provider.
- `FlashcardActivity` + `engine/Srs` (algoritmo SM‑2, semplice e senza librerie).
- `MapView`: mappa concettuale disegnata a mano su `Canvas` (come `ScoreChartView`) oppure Markmap in WebView.
- `TimelineView`: linea del tempo per storia.

### Gateway IA: due opzioni
| | A) n8n | B) chiamata diretta dall'app |
|---|---|---|
| Pro | Prompt modificabili senza aggiornare l'app; fallback tra provider; chiavi non nell'APK | Nessun server, zero costi, funziona subito |
| Contro | Serve hosting (VPS/Render/Railway, o n8n sul tuo PC) | Ogni utente inserisce la sua chiave Gemini gratuita (o la chiave finisce nell'APK: **da evitare**) |
| Quando | Pubblicazione su Play Store | Prototipo / uso personale |

**Consiglio:** partire con **B** (chiave Gemini gratuita dell'utente nelle impostazioni) per avere l'app
funzionante subito, e tenere l'`AiClient` astratto per passare ad **A (n8n)** al momento della pubblicazione.

## 6. Workflow IA (n8n)

**Workflow "Ingest"** — `POST /webhook/ingest`
1. Ricevi file → estrai testo (PDF parser / OCR).
2. Pulisci e dividi in blocchi (~1000 token, sovrapposizione 150).
3. Restituisce il testo pulito all’app, che lo salva in `fonte.txt` (embeddings solo se il materiale supera il contesto del modello).
4. Genera automaticamente titolo, materia rilevata, parole chiave.

**Workflow "Genera"** — `POST /webhook/generate { materialId, tipo, opzioni }`
1. Riceve il testo del materiale dall’app (Gemini Flash gestisce capitoli interi).
2. Scegli il prompt in base a `tipo` (riassunto, mappa, quiz, flashcard, timeline…).
3. Chiama l’LLM chiedendo output in formato fisso (JSON, oppure `[Q]…[/Q]` per i quiz).
4. Valida l’output (nell’app con `QuestionParser`) (se invalido → riprova / fallback su altro provider).
5. Restituisce all’app, che salva in locale.

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

## 7. Dati (locali, come in Lordoc)

Nessun account: tutto nella cartella privata dell'app, con file di testo come fa `UserDataStore`.

```
files/
├── materie/<materia>/<argomento>/
│   ├── fonte.txt          testo estratto dal materiale
│   ├── riassunto.md
│   ├── mappa.json         { nodi, archi }
│   ├── timeline.json      { eventi }
│   ├── quiz.txt           formato [Q]...[/Q] di Lordoc
│   └── flashcard.txt      blocchi [C] fronte / retro [/C]
├── srs.txt                stato flashcard (intervallo, scadenza, facilità)
└── storico.txt            verifiche svolte (come Lordoc)
```

Esempio di domanda generata (stesso formato di Lordoc, `domain` = argomento):
```text
[Q]
id: ST-RIVFR-001
domain: 3
type: SINGLE
text:
In che anno avvenne la presa della Bastiglia?
options:
A) 1776
B) 1789
C) 1799
D) 1815
answer: B
explanation:
Il 14 luglio 1789 il popolo di Parigi assaltò la Bastiglia, simbolo dell'assolutismo.
reference: Libro p. 112
[/Q]
```

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

- Copiare i pacchetti `model/data/engine/util` da Lordoc o estrarli in una libreria comune?
- Gateway: chiave utente (B) subito, n8n (A) dopo? Dove ospitare n8n?
- Pubblicazione su Play Store (con AdMob come Lordoc) o solo uso personale?
