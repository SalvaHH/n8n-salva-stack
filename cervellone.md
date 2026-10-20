[Knowledge base dalle email di un tour operator – documentazione tecnica.md](https://github.com/user-attachments/files/33727788/Knowledge.base.dalle.email.di.un.tour.operator.documentazione.tecnica.md)
# Knowledge base dalle email di un tour operator – documentazione tecnica

Oct 20, 2026 · @Salva

## Panoramica

Una knowledge base vettoriale costruita dalle risposte che un tour operator ha dato ai suoi clienti negli anni: circa 6.200 Q&A generiche, ognuna con embedding, su Supabase con pgvector e orchestrata in n8n. Servirà a un Responder per rispondere alle nuove email come risponderebbe il titolare.

&#91;embedded content: pipeline dalle email alla knowledge base · 4 fasi, 2 tabelle principali\]

Ogni fase lavora a lotti, salva il risultato prima di passare oltre e riparte da dove si era fermata: un crash non fa perdere lavoro.

## Struttura dati su Supabase

Tutto vive in un progetto Supabase dedicato, in due schemi: `emails_raw` per il layer grezzo e le valutazioni, `kb` per la knowledge base.

### Schema `emails_raw`

| Oggetto | Tipo | Contenuto |
| --- | --- | --- |
| `qa_pairs` | view | Coppie messaggio cliente + risposta dell'operatore (`question_email_id`, `answer_email_id`, `asked_at`, `answered_at`, `client_message`, `operator_response`) |
| `qa_evaluations` | tabella | Esito del filtro knowledge: `knowledge` (bool), `confidence`, `eval_source` (`regex` o versione del classificatore), `evaluated_at`. Chiave primaria (`question_email_id`, `answer_email_id`) |
| `qa_pairs_evaluated` | view | `qa_pairs` + valutazione, deduplicata con `distinct on (answer_email_id)` (tiene il messaggio cliente più vicino alla risposta) |

### Schema `kb`

| Oggetto | Tipo | Contenuto |
| --- | --- | --- |
| `kb_candidates` | tabella | Le Q&A estratte con embedding `vector(768)`: è la knowledge base. Colonne di matching: `dup_of`, `match_id`, `similarity`, `match_checked`. Colonne di verdetto: `verdict`, `verdict_source`, `verdict_confidence`, `times_seen`, `status` |
| `kb_processed` | tabella | Email processate che non hanno prodotto Q&A (evita di rielaborarle) |
| `kb_todo` | view | Email con `knowledge = true` non ancora estratte, in ordine cronologico |
| `kb_email_status` | view | Stato di ogni email knowledge: `todo`, `embedded`, `processed_no_items` |
| `kb_judge_todo` | view | Candidati senza verdetto, affiancati al loro match, per il classificatore |
| `match_next_batch(batch_size)` | funzione | Matching esatto a lotti, restituisce `{processed, remaining}` |
| `match_kb(vector, n)` | funzione | Ricerca per similarità (per il futuro Responder) |

### Permessi e sicurezza

- RLS attiva su tutte le tabelle, nessuna policy: la chiave pubblica `anon` non legge niente.
- n8n usa la chiave `service_role`, che ignora la RLS ma ha bisogno dei GRANT espliciti negli schemi custom: `usage` sullo schema, permessi su tabelle e view, `usage, select` sulle sequence, `execute` sulle funzioni.
- `alter default privileges` impostato su entrambi gli schemi, così le nuove tabelle e sequence ricevono i permessi in automatico.
- Le view usano `security_invoker = true`, per non aggirare la RLS.
- Entrambi gli schemi sono esposti nella Data API (necessario per il nodo Supabase di n8n). Dopo ogni nuova view o funzione: `notify pgrst, 'reload schema';`.

## Fase A – Filtro knowledge sulle coppie Q&A

Ogni coppia di `qa_pairs_evaluated` riceve un giudizio `knowledge` vero/falso: passano alla fase successiva solo le risposte con fatti riutilizzabili (luoghi, hotel, esperienze, distanze, prezzi, policy).

Workflow n8n:

1. **Lettura**: nodo Supabase su `emails_raw.qa_pairs_evaluated`, *Use Custom Schema* attivo, filtro stringa `knowledge=is.null`, Return All disattivato e limite 500.
2. **Filtro "?"**: IF su `client_message` contiene `?`. Senza punto interrogativo → scritto subito come `knowledge = false`.
3. **Filtro regex**: IF su `operator_response` (case-insensitive) con le formule di rifiuto tipiche (*already booked*, *fully booked*, *not available on*, *cannot accept*) → `knowledge = false`.
4. **Jev** (TypeSafe AI, un LLM classificatore, operazione Route) sul resto, con due route: `general_knowledge` e `booking_only`. Default `booking_only`, che vince in caso di dubbio.
5. **Scrittura** in `emails_raw.qa_evaluations`, una riga per coppia, sia per i `true` sia per i `false`.

| Esito | `knowledge` | `confidence` | `eval_source` |
| --- | --- | --- | --- |
| Scartato da "?" o regex | false | 1 | `regex` |
| Jev: `booking_only` | false | `route.confidence` | versione di Jev |
| Jev: `general_knowledge` | true | `route.confidence` | versione di Jev |

Il test decisivo nelle istruzioni di Jev: si può riscrivere una frase della risposta come fatto autonomo, senza nome e date del cliente, che insegni qualcosa sulla destinazione o sui servizi offerti? Se no, `booking_only`. Gli esempi negativi includono le formule di rifiuto ("fully booked") e le richieste di dettagli ("send me your ideas").

## Fase B – Estrazione Q&A ed embedding

Ogni email con `knowledge = true` viene scomposta da Gemini in Q&A atomiche e generiche; ogni Q&A riceve un embedding a 768 dimensioni e finisce in `kb.kb_candidates`. Risultato: circa 6.200 candidati.

Nodi, in ordine:

1. **Get many rows** (Supabase, schema `kb`, tabella `kb_todo`, limite 20).
2. **Basic LLM Chain** con **Gemini** (modello flash-lite): estrae solo conoscenza utile a clienti futuri, toglie nomi, date del viaggio, dettagli di prenotazione e saluti, una domanda per item. Restituisce `{"items": [{type, question, answer}]}`. Retry on fail: 3 tentativi.
3. **Edit Fields**: recupera `answer_email_id`, `question_email_id`, `answered_at` da `Get many rows`, perché l'output dell'LLM contiene solo `text`.
4. **Code in Python** (runner nativo, `_items`): toglie i recinti ```` ```json ````, fa il parse, crea un item per Q&A. Se non ci sono Q&A emette un item con `empty: true`.
5. **IF `empty`**: true → riga in `kb_processed`; false → passo successivo.
6. **HTTP Request** a `gemini-embedding-001:embedContent` con la credenziale *Google Gemini(PaLM) Api*, body `JSON.stringify({content:{parts:[{text: question + "\n" + answer}]}, output_dimensionality: 768})`.
7. **Merge** *Combine by Position* tra Q&A e risposta dell'embedding (l'HTTP perde i campi originali).
8. **Create a row** in `kb_candidates`, con `embedding = {{ $json.embedding.values }}` (l'API converte l'array in vettore).

Tipi di Q&A estratti: `info_request`, `service_request`, `pricing`, `policy`.

Ripresa dopo un crash: un'email è considerata fatta solo se ha righe in `kb_candidates` o in `kb_processed`, quindi se il workflow si interrompe a metà l'email torna in `kb_todo` al giro successivo.

## Fase C – Duplicati e matching

Ogni candidato viene collegato al candidato **precedente** (in ordine cronologico) più simile. Alla fine solo il primissimo candidato resta senza match.

**1. Duplicati esatti (SQL, una volta).** Candidati con domanda e risposta identiche (ignorando maiuscole e spazi) puntano alla prima occorrenza: `dup_of` = id della prima, `similarity = 1`, `verdict = 'REDUNDANT'`, `verdict_source = 'exact_text'`. Sono solo 22: Gemini riformula leggermente ogni estrazione, quindi i quasi-duplicati sono parafrasi, non testi identici.

**2. Matching esatto (funzione `kb.match_next_batch`).** Per ogni candidato non duplicato, confronta l'embedding con **tutti** i candidati precedenti non duplicati e salva il più simile in `match_id` e `similarity` (similarità coseno, `1 - distanza`). L'ordinamento è sul valore calcolato, non sull'operatore di distanza, così Postgres non usa l'indice approssimato.

Workflow n8n:

1. Manual Trigger.
2. **HTTP Request** `POST https://<project-ref>.supabase.co/rest/v1/rpc/match_next_batch`, credenziale *Supabase API*, header `Content-Profile: kb`, body `{"batch_size": 50}`.
3. **IF** `{{ $json.remaining }}` > 0 (validazione *loose*): true → torna all'HTTP Request.

La funzione restituisce `{"processed": n, "remaining": n}`. Il loop si ferma quando `remaining` è 0.

Prima versione scartata: cercava i 10 vicini più simili con l'indice HNSW e teneva il più simile tra quelli precedenti. Lasciava 497 candidati senza match, perché nei gruppi di parafrasi molto simili i 10 vicini potevano essere tutti successivi.

## Fase D – Verdetti del classificatore

Jev confronta ogni candidato con il suo match e assegna uno di quattro verdetti. Le soglie automatiche di similarità sono state scartate: con gli embedding Gemini quasi tutto supera 0,80 (la mediana è intorno a 0,89), e il classificatore costa poco. Circa 6.100 coppie in coda.

| Verdetto | Significato | Effetto nella KB finale |
| --- | --- | --- |
| `NEW` | Argomento o domanda diversi | Resta attivo |
| `REDUNDANT` | Stessa informazione, parole diverse | Escluso, incrementa `times_seen` del match |
| `ADDS_DETAIL` | Stesso tema, fatti in più | Resta attivo come voce separata |
| `CONTRADICTS` | Stesso tema, fatti in conflitto | Resta attivo con la sua data; lista di revisione per il titolare |

Workflow n8n:

1. **Manual Trigger** (test) e **Schedule Trigger** ogni 10 minuti.
2. **Get next pair**: Supabase su `kb.kb_judge_todo`, limite 1.
3. **Jev** (Route) con State = nuova coppia + voce esistente, quattro route. Retry on fail: 3.
4. **Save verdict**: update di `kb_candidates` dove `id = {{ $('Get next pair').first().json.id }}`, con `verdict = route.choice`, `verdict_source` = versione di Jev, `verdict_confidence = route.confidence`. Tutte e quattro le route entrano in questo nodo.
5. **Under 300?**: IF `{{ $runIndex }}` < 300 → torna a Get next pair; altrimenti l'esecuzione finisce.

Una coppia per giro rende affidabile `.first()` e il tetto di 300 tiene bassa la memoria di ogni esecuzione. Le esecuzioni riuscite non vengono salvate.

Primo campione (209 verdetti, email più vecchie): 105 `NEW`, 84 `ADDS_DETAIL`, 10 `CONTRADICTS`, 10 `REDUNDANT`. La lettura dei `CONTRADICTS` ha mostrato quasi solo falsi conflitti (stime di durata diverse, servizi diversi, consigli complementari): da qui la decisione di non disattivare mai automaticamente la voce più vecchia.

## Decisioni chiave

| Decisione | Motivo |
| --- | --- |
| Supabase con pgvector, niente database vettoriale separato | Vettori, email e valutazioni nello stesso database, uniti con SQL |
| Schema separato `kb` per la knowledge base | La KB non è dato grezzo; resta fuori da `emails_raw` |
| Scartare a monte i messaggi cliente senza "?" | Filtro gratuito e deterministico, riduce le chiamate LLM |
| Regex prima del classificatore per i rifiuti standard | Le formule di rifiuto sono fisse: non serve un LLM |
| Classificatore con default `booking_only` | Le prime versioni facevano passare risposte senza fatti; ora serve un fatto concreto |
| `eval_source` con la versione del classificatore | Permette di confrontare versioni diverse sulle stesse coppie |
| `confidence = 1` per la regex, su scala 0–1 | Una scala sola; l'origine della decisione sta in `eval_source` |
| Prima salvare tutti i candidati, poi deduplicare | Si può rifare la deduplicazione senza ripagare estrazione ed embedding |
| Embedding Gemini a 768 dimensioni | Multilingue (IT/EN), sotto il limite di 2.000 dimensioni dell'indice HNSW |
| Matching esatto invece che con l'indice | Con \~6.200 candidati è fattibile e non lascia buchi |
| Tutto al classificatore invece delle soglie | Distribuzione delle similarità schiacciata in alto, classificatore economico |
| `kb_candidates` è la KB finale | Niente copie: le voci attive sono quelle con `status = 'active'` |
| `CONTRADICTS` non disattiva la voce vecchia | Quasi tutti i casi letti erano falsi conflitti; la recenza si gestisce nel Responder |
| Ogni workflow lavora a lotti con ripresa | Un crash non perde lavoro e non accumula memoria |

## Problemi incontrati e soluzioni

| Sintomo | Causa | Soluzione |
| --- | --- | --- |
| `name '_input' is not defined` nel Code node | n8n usa il runner Python nativo | Usare `_items` (tutti gli item) o `_item` (un item); ogni item è un dict con chiave `json` |
| Crash di n8n e *Bad Gateway* | Uscita `done` del Loop ricollegata al Loop: loop infinito | L'uscita `done` non va collegata a niente; lavorare a lotti con limite |
| Messaggi senza "?" che passano l'IF | Condizione IF del tipo sbagliato | Boolean → *is true*, oppure String → *contains* sul campo giusto |
| `permission denied for table` / `for sequence` | Negli schemi custom Supabase non dà i GRANT in automatico | `grant usage` sullo schema, permessi su tabelle e sequence a `service_role`, `alter default privileges` |
| `Invalid schema: <nome>` | Schema non esposto nella Data API | Project Settings → Data API → Exposed schemas, oppure nodo Postgres diretto |
| `Could not find the table … in the schema cache` | View appena creata o mai creata | Crearla e lanciare `notify pgrst, 'reload schema';` |
| Il filtro del nodo Supabase non filtra | Sintassi SQL invece di PostgREST, e soprattutto *Use Custom Schema* spento (leggeva una view vecchia in `public`) | Filtro `knowledge=is.null`; *Use Custom Schema* attivo con lo schema giusto |
| `duplicate key value violates unique constraint` | Coppie già valutate rilette dal filtro sbagliato | Correggere schema e filtro; in caso di dubbio `truncate` e ripartire |
| Righe duplicate nella view delle coppie | Più messaggi cliente abbinati alla stessa risposta | `distinct on (answer_email_id)`, tenendo il messaggio più vicino |
| `null` negli id dopo l'LLM | L'output del nodo LLM contiene solo `text` | Edit Fields con `$('Get many rows').item.json…` |
| Ogni email con le stesse Q&A | Output del Basic LLM Chain fissato (*pinned*) | Togliere il pin nel pannello Output |
| `invalid input syntax for type vector` | Passato l'oggetto `embedding` intero | Passare `embedding.values` |
| *Upstream timeout* nel SQL Editor | Confronto di tutti con tutti in una query | Funzione a lotti chiamata in loop da n8n |
| Nodo HTTP con item vuoto | La funzione restituiva un numero nudo, illeggibile per n8n | Funzione che restituisce `jsonb` (`processed`, `remaining`); `drop function` prima di cambiare il tipo |
| 497 candidati senza match | Ricerca con l'indice limitata a 10 vicini, saturati dalle parafrasi successive | Matching esatto su tutti i precedenti |
| `column … does not exist` in un update | Colonna mai creata | `add column if not exists` all'inizio di ogni blocco |

Regola generale nata da questi errori: durante i test, mettere `onError` su *Stop workflow*. Con *Continue* gli errori passano come item normali e non si vedono.

## Prossimi passi e query di controllo

Le fasi A, B e C sono complete; la fase D è in corsa.

- [ ] Finire i verdetti del classificatore
- [ ] Policy cambiate nel tempo: trovare la data in cui una regola cambia nelle email e segnare come `superseded_policy` le voci precedenti
- [ ] Impostare `status` (`active`, `redundant`) e `times_seen`; `CONTRADICTS` resta attivo
- [ ] Esportare i `CONTRADICTS` come lista di revisione per il titolare
- [ ] Confrontare verdetti e similarità per trovare soglie automatiche da usare sulle email future
- [ ] Costruire il Responder: ricerca su `status = 'active'`, a parità di argomento la voce più recente
- [ ] Valutare una tabella di regole ufficiali per i cambi decisi prima che compaiano nelle email

### Query di controllo

Coda del classificatore:

```sql
select count(*) as da_fare from kb.kb_judge_todo;
```

Distribuzione dei verdetti:

```sql
select coalesce(verdict, 'da fare') as verdict,
       coalesce(verdict_source, '-') as source,
       count(*)
from kb.kb_candidates
group by 1, 2
order by 3 desc;
```

Soglie reali, a classificazione finita:

```sql
select round(similarity::numeric, 2) as sim,
       count(*) filter (where verdict = 'REDUNDANT')   as redundant,
       count(*) filter (where verdict = 'ADDS_DETAIL') as adds_detail,
       count(*) filter (where verdict = 'CONTRADICTS') as contradicts,
       count(*) filter (where verdict = 'NEW')         as new
from kb.kb_candidates
where verdict_source <> 'exact_text'
group by 1 order by 1 desc;
```

Stato delle email knowledge:

```sql
select status, count(*) from kb.kb_email_status group by 1;
```
