# Lorstudy — Definizione del progetto

> App Android per lo studio delle materie umanistiche (italiano, storia, filosofia, latino…).
> **Input:** il libro (PDF, EPUB, DOCX, foto delle pagine).
> **Output:** riassunti, mappe concettuali, linee del tempo, schede, flashcard, quiz e simulazioni di verifica.

Lorstudy mantiene la **logica di Lordoc Android**: Java nativo, nessun account, dati sul telefono,
formato testuale validato dal parser, modalità Studio/Simulazione, statistiche per elemento, storico, CI
che produce l'APK, AdMob + UMP. Cambia la **struttura**: Lordoc parte da una banca domande fissa negli
assets; Lorstudy parte da un **libro** e costruisce i contenuti con una **pipeline** di generazione.

---

## 1. Idea in una riga

*Carichi il libro, scegli il capitolo, e Lorstudy ti prepara tutto il necessario per capirlo,
memorizzarlo e verificarti.*

## 2. Il flusso

```
 LIBRO ──▶ 1. IMPORT ──▶ 2. STRUTTURA ──▶ 3. GENERAZIONE ──▶ 4. VALIDAZIONE ──▶ 5. STUDIO ──▶ 6. VERIFICA
 PDF/EPUB   estrazione    capitoli,        un generatore      parser per           lettura,      quiz, esiti,
 DOCX/foto  testo + OCR   paragrafi,       per tipo di        formato (come        flashcard     storico,
                          pagine           contenuto (IA)     QuestionParser)      con SRS       punti deboli
```

1. **Import** — PDF con testo (estrazione diretta), PDF scansionato o foto (OCR on‑device, ML Kit), EPUB, DOCX, testo incollato.
2. **Struttura** — il libro viene diviso in **Capitoli → Sezioni** (dall'indice del PDF se c'è, altrimenti con euristiche sui titoli + conferma dell'utente). Ogni sezione conserva i numeri di pagina, per citare la fonte.
3. **Generazione** — l'utente sceglie *cosa* generare e *su quale parte* (una sezione, un capitolo, più capitoli). Ogni tipo di contenuto ha un suo generatore con un prompt e un formato fissi.
4. **Validazione** — l'output dell'IA viene letto da un parser dedicato (come `QuestionParser` in Lordoc): blocchi malformati vengono scartati e segnalati, mai mostrati.
5. **Studio** — lettura di riassunti/mappe/schede, ripasso flashcard con ripetizione spaziata.
6. **Verifica** — quiz e simulazioni con la stessa logica d'esame di Lordoc; statistiche per argomento.

## 3. Contenuti generati

| Tipo | Formato di output | Schermata | Note |
|---|---|---|---|
| **Riassunto** (breve / medio / per punti) | Markdown | lettore (WebView) | Ogni paragrafo con rimando di pagina |
| **Mappa concettuale** | `[MAP]` nodi + archi | `ConceptMapView` (Canvas) | Relazioni: causa, conseguenza, parte di, autore di… |
| **Linea del tempo** | `[EVENT]` data + titolo + descrizione | `TimelineView` | Soprattutto storia |
| **Scheda autore / opera** | Markdown a sezioni fisse | lettore | Vita, poetica, opere, temi, contesto |
| **Analisi del testo** | Markdown a sezioni fisse | lettore | Parafrasi, figure retoriche, metrica, temi |
| **Glossario** | `[TERM]` termine + definizione | lista | Diventa anche flashcard |
| **Flashcard** | `[C]` fronte / retro | `FlashcardActivity` | Ripetizione spaziata (SM‑2) |
| **Quiz** | `[Q]…[/Q]` **formato Lordoc** | `ExamActivity` | SINGLE, MULTI, FILL, DRAG |
| **Domande aperte** | `[Q]` con `type: OPEN` | `ExamActivity` | Correzione IA con voto e punti mancanti |
| **Interrogazione simulata** | chat | `OralActivity` | v0.3 |

Esempi dei formati (tutti testuali, a blocchi, come Lordoc):

```text
[Q]
id: STO-C03-007
topic: C03
type: DRAG
text:
Abbina ogni evento al suo anno.
items:
Presa della Bastiglia => 1789
Congresso di Vienna => 1815
targets: 1789, 1815, 1848
explanation:
La Rivoluzione inizia nel 1789; il Congresso di Vienna (1814-15) apre la Restaurazione.
reference: p. 112-118
[/Q]

[C]
id: STO-C03-F012
front: Che cosa stabilì il principio di legittimità?
back: Il ritorno sui troni dei sovrani spodestati da Napoleone (Congresso di Vienna).
reference: p. 117
[/C]

[MAP]
title: Cause della Rivoluzione francese
node: n1 | Crisi finanziaria
node: n2 | Convocazione degli Stati Generali
edge: n1 -> n2 | causa
[/MAP]

[EVENT]
date: 1789-07-14
title: Presa della Bastiglia
text: Il popolo di Parigi assalta la fortezza, simbolo dell'assolutismo.
reference: p. 113
[/EVENT]
```

**Perché testo a blocchi e non JSON:** è la scelta di Lordoc — leggibile, modificabile a mano,
importabile da file, e un blocco rotto non invalida tutto il file.

## 4. Cosa resta uguale a Lordoc (la "logica")

| Logica Lordoc | In Lorstudy |
|---|---|
| Banca domande da file `.txt` + `index.txt`, parser con errori per riga | Ogni libro ha i suoi file generati; stesso parser esteso con `[C]`, `[MAP]`, `[EVENT]`, `[TERM]` |
| `Domain` con peso e `DomainScore` | **Capitolo/argomento** con accuratezza per capitolo; peso = lunghezza del capitolo |
| `ExamFactory`: esami predefiniti disgiunti, distribuzione pesata | **Verifiche per capitolo** e **verifica generale** sul libro, stessa distribuzione pesata |
| `QuestionFilter`: tutte / mai viste / sbagliate / marcate | Identico, + "in scadenza" per le flashcard |
| `ExamMode`: Studio / Simulazione a tempo, divieto di tornare indietro | Studio / **Verifica in classe** |
| Opzioni mescolate a ogni esame | Identico |
| `Scorer` | Voto in **decimi** (e trentesimi per università) invece di 300–1000 |
| `QuestionStat` (viste, corrette, ultima risposta, marcata) | Identico, + stato SRS per le flashcard |
| `UserDataStore`: TSV nella cartella privata | Identico, una cartella per libro |
| Storico + `ScoreChartView`, revisione errori | Identico |
| Importazione file `.txt` dal telefono | Import e **export** dei contenuti (condividere un set con i compagni) |
| `BaseActivity`, `Ui`, `Ads`, CI GitHub Actions, `release.properties` | Identici |

## 5. Cosa cambia (la "struttura")

**Lordoc:** contenuti statici negli assets → caricati all'avvio → esame.
**Lorstudy:** biblioteca dell'utente → pipeline asincrona → contenuti salvati → studio/verifica.

### Pacchetti

```
com.lorstudy
├── model/        Book, Chapter, Section, Question, Flashcard, ConceptMap, TimelineEvent, Term,
│                 StudySession, Response, Result, ContentType, SessionMode
├── ingest/       PdfExtractor, EpubExtractor, DocxExtractor, OcrExtractor,
│                 StructureDetector (indice/titoli → capitoli), TextCleaner, Chunker
├── generate/     GenerationJob, GenerationQueue, PromptLibrary (prompt per tipo),
│                 Generator (interfaccia) + SummaryGen, MapGen, QuizGen, FlashcardGen, TimelineGen…
├── ai/           AiProvider (interfaccia), GeminiProvider, GroqProvider, OpenRouterProvider,
│                 N8nProvider, ProviderChain (fallback, retry, limiti di frequenza)
├── format/       BlockParser (base comune) + QuestionParser, FlashcardParser, MapParser,
│                 EventParser, TermParser, ParseReport (errori per blocco)
├── engine/       SessionFactory, ContentFilter, Scorer, Srs (SM‑2), WeightedPicker
├── data/         Library (libri e capitoli), ContentStore (file generati), UserDataStore (TSV),
│                 ResourceOpener
├── util/         Text, Hashing, ecc.
└── android/      LibraryActivity, BookActivity, ImportActivity, GenerateActivity,
                  ReaderActivity, ConceptMapView, TimelineView, FlashcardActivity,
                  ExamActivity, ResultsActivity, ReviewActivity, HistoryActivity,
                  SettingsActivity, BaseActivity, Ui, Ads
```

### Schermate

1. **Biblioteca** — i tuoi libri (copertina, materia, % studiato). Pulsante *Aggiungi libro*.
2. **Libro** — indice dei capitoli; per ogni capitolo i contenuti già generati e i pulsanti per generarne altri; accuratezza per capitolo.
3. **Genera** — scelta del tipo, dell'intervallo (sezione/capitolo/pagine), delle opzioni (lunghezza, numero di domande, difficoltà); avanzamento della coda.
4. **Lettore** — riassunti, schede, analisi, con link alla pagina del libro.
5. **Mappa** / **Timeline** — zoom e trascinamento.
6. **Flashcard** — ripasso del giorno (tutti i libri o uno solo).
7. **Verifica** / **Risultati** / **Revisione** / **Storico** — come Lordoc.
8. **Impostazioni** — provider IA e chiave, lingua dei contenuti, livello scolastico, consenso annunci.

### Dati sul telefono

```
files/
├── library.tsv                     elenco libri
└── books/<bookId>/
    ├── source.pdf                  copia del libro (opzionale)
    ├── text/<sectionId>.txt        testo estratto per sezione, con marcatori di pagina
    ├── structure.tsv               capitoli, sezioni, pagine
    ├── content/<chapterId>/
    │   ├── summary_medium.md
    │   ├── map.txt  timeline.txt  terms.txt
    │   ├── quiz.txt  flashcards.txt
    │   └── author_<nome>.md
    ├── stats.tsv  srs.tsv  history.tsv
    └── jobs.tsv                    generazioni in corso / fallite
```

## 6. IA gratuita

| Provider | Uso | Perché |
|---|---|---|
| **Google Gemini Flash** (AI Studio) | principale | contesto lunghissimo: un capitolo intero in una chiamata; legge anche PDF e immagini |
| **Groq** (Llama/Qwen) | quiz, flashcard, chat | molto veloce |
| **OpenRouter** (modelli `:free`) | riserva | quando gli altri raggiungono i limiti |
| **Ollama** sul PC | facoltativo | zero limiti, privacy totale |

`ProviderChain` prova i provider in ordine; se l'output non supera il parser, ritenta una volta
chiedendo di correggere il formato, poi passa al successivo.

**Gateway:**
- **Fase 1 – chiave dell'utente**: l'utente incolla la sua chiave gratuita di Gemini nelle impostazioni (salvata cifrata con Android Keystore). Nessun server.
- **Fase 2 – n8n** (per il Play Store): l'app chiama `POST /webhook/generate { tipo, testo, opzioni }`; n8n sceglie il provider, applica il prompt e risponde nel formato a blocchi. Prompt aggiornabili senza nuova versione dell'app; nessuna chiave nell'APK.

**Testi lunghi:** se un capitolo supera il contesto del modello, `Chunker` lo divide, si genera per parti e
poi si fondono i risultati (map‑reduce per i riassunti, unione + deduplica per quiz e flashcard).

## 7. Requisiti

### Funzionali — MVP (v0.1)
1. Import PDF con testo e testo incollato.
2. Rilevamento capitoli dall'indice del PDF, modificabile a mano.
3. Generazione di riassunto, flashcard e quiz (SINGLE/MULTI) per capitolo.
4. Parser e segnalazione degli errori di formato.
5. Ripasso flashcard con SM‑2; verifiche per capitolo in modalità Studio e Verifica.
6. Risultati, revisione, storico, accuratezza per capitolo.
7. Impostazione della chiave Gemini.

### v0.2
8. OCR di PDF scansionati e foto; EPUB e DOCX.
9. Mappa concettuale, linea del tempo, glossario, scheda autore/opera, analisi del testo.
10. Domande FILL, DRAG e aperte con correzione IA.
11. Export/import dei contenuti e verifica generale sul libro.

### v0.3+
12. Interrogazione simulata (chat, poi voce).
13. Gateway n8n e pubblicazione su Play Store.
14. Piano di studio verso la data della verifica.

### Non funzionali
- Android 8.0+ (minSdk 26), target API 36, Java, nessuna libreria oltre AdMob/UMP, ML Kit (OCR) e un estrattore PDF.
- Studio e verifiche **offline**; rete solo per generare.
- Al provider IA si invia solo il testo da elaborare; il libro resta sul telefono.
- Nessun annuncio durante una verifica o un ripasso (come Lordoc).
- Contenuti in italiano; ogni elemento con riferimento di pagina.
- Test automatici: ogni parser ha test con blocchi validi e rotti (come `laBancaDomandeSiCaricaSenzaErrori`).

## 8. Rischi

| Rischio | Mitigazione |
|---|---|
| Errori dell'IA (date, attribuzioni) | Contenuti basati solo sul testo del libro, rimando di pagina, pulsante "segnala/modifica" |
| Limiti dei piani gratuiti | Generazione su richiesta, coda con ripresa, fallback tra provider |
| PDF senza testo o impaginati male | OCR, pulizia del testo, correzione manuale dei capitoli |
| Copyright del libro | Il libro non lascia il telefono salvo il testo inviato al provider; export solo dei contenuti generati |
| Utenti minorenni | Consenso UMP, nessun account, nessun dato raccolto |

## 9. Roadmap

| Fase | Risultato |
|---|---|
| 0 | Scheletro Android (copia di BaseActivity, Ui, CI) + `format/` e `engine/` portati da Lordoc con test |
| 1 | Import PDF + struttura capitoli |
| 2 | `ai/` + generatori di riassunto, flashcard, quiz → **MVP** |
| 3 | Mappe, timeline, schede, OCR |
| 4 | Domande aperte, interrogazione, n8n, Play Store |

## 10. Decisioni aperte

- Estrattore PDF: `PdfBox-Android` (completo, ~ alcuni MB) oppure Gemini che legge direttamente il PDF (zero librerie, ma invia il PDF al provider).
- Codice comune con Lordoc: copiare i pacchetti o estrarre una libreria condivisa.
- Pubblicazione con AdMob o uso personale.
