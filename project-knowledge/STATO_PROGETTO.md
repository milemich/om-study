# OM Study — Stato del progetto

> Ultimo aggiornamento: 2026-10-19 (tasto indietro portato in produzione, allineata alla beta)

## Cos'è
Web app gamificata (stile Duolingo/flashcard) per aiutare gli studenti di Biotransenergetica (BTE) a studiare i temi del corso: XP, livelli, streak giornaliere, badge, flashcard a ripetizione dilazionata (SM-2), quiz a correzione automatica, e una sezione "Libri" per studiare testi esterni tramite riassunti originali, quiz, flashcard e glossario.

## Dove vive
- **Repo GitHub**: `milemich/om-study` (branch `main`)
- **Hosting**: GitHub Pages, servito direttamente da `index.html` (single-file app: HTML + CSS + JS inline, nessun build step)
- **Backend**: Supabase
  - Organizzazione: `OM_Counseling`
  - Progetto: `OM_Study` (project id: `auuatxzqvwjbwfketgjh`)
  - Separato dall'organizzazione `TMS` usata per il portale trasporti (altro progetto)
- **Librerie usate** (via jsDelivr UMD, nessun npm/bundler): Supabase JS client v2.45.4, Lucide icons

## Ambiente produzione vs beta (admin)
Per poter testare nuove funzionalità senza rischiare di rompere l'esperienza degli studenti già attivi, l'app esiste in due copie nello stesso repo/branch `main`:
- **Produzione** (`index.html`, root del sito) — quello che usano tutti gli studenti. URL: quello pubblicato in GitHub Pages.
- **Beta** (`beta/index.html`) — stessa app, stesso Supabase/dati, ma con `const BETA_MODE = true` nello script. URL: lo stesso dominio + `/beta/`.
  - Mostra un nastro "VERSIONE BETA" in alto.
  - Al login, chi non ha ruolo `admin` viene disconnesso automaticamente e rimandato al sito principale (quindi è sicura anche se qualcuno trova l'URL per sbaglio).
- Il database Supabase è **condiviso** tra le due versioni (nessun ambiente separato): scelta fatta per semplicità. I dettagli e il workflow di rilascio (beta → produzione) sono in `DECISIONI.md`.
- Attualmente **produzione e beta hanno lo stesso codice** (solo flag `BETA_MODE` e titolo pagina diversi): il 2026-10-19 è stata portata in produzione la gestione del tasto indietro con ripristino dello scorrimento (vedi sotto), testata prima in beta.

## Tasto indietro / navigazione (in produzione dal 2026-10-19)
La navigazione interna (tab, libri, capitoli, quiz, music...) usa un piccolo router basato su `history.pushState`/`popstate`: ogni volta che si entra in una nuova schermata si registra una voce in cronologia, così il tasto indietro del telefono torna alla schermata precedente dell'app invece di uscire, e ritrova anche lo stesso punto di scorrimento. I semplici aggiornamenti della stessa schermata (dopo un salvataggio, un toast, ecc.) non toccano la cronologia. Funzioni chiave: `nav(renderFn)` (push + render) e il listener `popstate` (richiama la vista salvata e ripristina lo scroll). I pulsanti "Indietro" dell'interfaccia usano `history.back()` anziché richiamare direttamente la vista precedente.

## Architettura dati (Supabase / Postgres)

### Tabelle core
- `profiles` — utenti, con `email`, ruolo admin (via `is_admin()`)
- `modules` — moduli di corso; `kind` distingue corso normale vs libro (`kind='book'`), più `author`, `cover_color`
- `lessons` — lezioni/capitoli dentro un modulo; `key_points` per i capitoli-libro
- `questions` + `question_keys` — domande quiz e risposte corrette (tabelle separate per non esporre le risposte agli studenti via RLS)
- `flashcards` — carte per ripetizione dilazionata
- `glossary_terms` — termini di glossario per i libri
- `chapter_reads` — tracciamento capitoli letti (per XP e badge "libro completato")
- `notes` — note personali degli studenti (vedi sotto)
- `question_reports` — segnalazioni di domande con risposta sbagliata/mancante (vedi sotto)
- `songs` — canti della sezione Music: `title`, `lyrics` (testo con sintassi colori e tempi), `audio_path` (file nel bucket Storage `music`), `is_published`, `order_index`. Non c'è un campo artista: l'artista va messo nel testo, tra parentesi in cima.

### Tracking progressi/gamification
- `lesson_attempts`, `answer_log`, `lesson_progress`, `flashcard_reviews`, `daily_activity`, `xp_events`
- `badges` + `user_badges` — 14 badge iniziali + 4 aggiunti per la sezione libri (read_1, read_25, book_1, book_3)

### Funzioni / RPC principali (SECURITY DEFINER, accesso controllato)
- `level_for_xp(xp)` — formula livello: `floor(sqrt(xp/100))+1`
- `_today()` — data di oggi in timezone Europe/Rome, usata per gli streak
- `_record_activity`, `_check_badges` — helper interni (eseguibili solo dal sistema, REVOKE/GRANT bloccano l'accesso diretto degli studenti)
- `start_lesson`, `submit_answer`, `finish_lesson` — flusso quiz; XP: 10 per risposta corretta + 20 bonus lezione perfetta (dimezzato se ripetuta)
- `review_flashcard`, `get_due_flashcards` — spaced repetition stile SM-2 (ease factor, intervallo, data di scadenza)
- `get_my_stats` — statistiche utente (XP, livello, streak, badge)
- `mark_chapter_read(uuid)` — 15 XP per capitolo letto, una sola volta
- `import_book(data jsonb, p_publish boolean default true)` — importazione bulk di un libro intero (modulo + lezioni + domande + flashcard + glossario) da un payload JSON strutturato; riusabile per ogni nuovo libro

### Edge Function
- `invite-user` (Deno) — invito nuovi studenti / reset password, solo admin, usa la service role key lato server

## Contenuti libri caricati finora
1. **"Terapia verbale"** (primo contenuto caricato, materiale di base del corso)
2. **"Introduzione alla realtà"** — Edoardo Camurri
3. **"Il corpo non mente"** — Luciano Marchino & Monique Mizrahil
4. **"La scienza e il piano akashico"** — Ervin Laszlo (15 capitoli, 75 domande, 75 flashcard, 42 termini di glossario)
5. **"Oltre i confini"** — Ken Wilber (da *No Boundary*, edizione inglese; 10 capitoli, 50 domande, 50 flashcard, 47 termini di glossario)
6. **"Il calice e la spada"** — Riane Eisler (15 lezioni: introduzione, 13 capitoli e postfazione/bilancio critico; 75 domande, 75 flashcard, 45 termini di glossario; module id `e5a8458a-ec3f-4b79-81be-06c69890130a`, order_index 5, copertina `#7a3b52`)

Tutti i riassunti/quiz/flashcard/glossario sono stati scritti originalmente da Claude leggendo i PDF, **mai copiando testo verbatim** (vedi DECISIONI.md sulla policy copyright).

Nota sul libro di Laszlo: contiene tesi non accettate dalla scienza ufficiale (campo akashico, telepatia, effetti della preghiera, esperienze di quasi-morte e comunicazione dopo la morte, critiche al darwinismo). I riassunti le attribuiscono sempre all'autore e riportano "Nota critica" nei capitoli interessati; i quiz chiedono ciò che dice il libro, non che sia vero. Inoltre sono segnalati i dati invecchiati (es. bosone di Higgs osservato nel 2012).

Nota sul libro di Wilber: il PDF è in inglese e **non è stata prodotta una traduzione integrale** (non consentita dalla policy copyright, neppure tradotta): il titolo italiano «Oltre i confini» e tutti i contenuti sono una rielaborazione originale in italiano. Ogni capitolo ha una "Nota critica" dove il testo mescola scienza e misticismo (parallelo fisica quantistica–Oriente, filosofia perenne, ESP, inconscio collettivo, Sé unico/reincarnazione, teoria dei blocchi muscolari di Reich/Lowen, proiezione generalizzata) o propone esercizi (cautela per chi ha traumi, ansia o dissociazione). Il libro è del 1979 (prefazione 2001): le letture consigliate sono datate e l'autore ha rivisto in seguito alcune posizioni.

Nota sul libro di Eisler: PDF in italiano (edizione Z-Library), contenuti rielaborati originalmente, nessun testo copiato. Le tesi (società preistoriche mutuali dell'«Antica Europa», culto della Dea, invasioni kurgan, «capovolgimento della realtà» nella Bibbia e nella Grecia, gilania/androcrazia, scenari futuri) sono sempre attribuite all'autrice. Ogni lezione ha una "Nota critica": la ricostruzione archeologica (Çatal Höyük, Creta, Kurgan) è molto contestata, le cifre sulle cacce alle streghe sono discusse, i dati sono del 1987 e vanno aggiornati. L'ultima lezione (postfazione 2011) include un bilancio critico complessivo: il libro va usato come stimolo di riflessione, non come manuale di archeologia. Le domande di tipo "nota critica" verificano che lo studente distingua tesi e fatti.

## Sezione Note (studenti)
Gli studenti possono prendere note personali mentre leggono un capitolo di un libro, e ritrovarle tutte in una sezione dedicata "Le mie note" con filtro/ricerca ed export (txt, Word, copia negli appunti, stampa/PDF). Le note sono private per utente (RLS su `user_id = auth.uid()`).

## Sezione Segnalazioni domande (studenti → admin)
Gli studenti possono segnalare una domanda del quiz la cui risposta corretta non è presente nel riassunto fornito, con un motivo (enum) + commento libero. L'admin vede le segnalazioni raggruppate per domanda in un pannello di gestione, con azioni "risolvi"/"ignora". Vedi DECISIONI.md per la policy su cosa va corretto.

## Sezione Music
Tab "Music" nella barra in basso: elenco canzoni e pagina canzone a schermo intero in stile **karaoke notturno** (sfondo scuro, verso attivo grande e luminoso, gli altri sfumati). Il lettore audio è custom (play/pausa, barra, loop). Se i versi hanno i tempi, l'evidenziazione segue la registrazione e toccando un verso si salta lì; senza tempi si può evidenziare i versi a mano toccandoli. Proposte grafiche valutate (canvas "Music – testi moderni"): scelta la versione B, karaoke.
Audio nel bucket Storage privato `music` (URL firmati da 1 ora, limite file 50 MB). Lettura per tutti gli utenti loggati, scrittura solo admin (`is_admin()`). L'admin crea/modifica/elimina canzoni dall'app, carica l'audio dal form e imposta i tempi con lo strumento "Tempi del karaoke" (si fa partire l'audio e si tocca il pulsante all'inizio di ogni verso; dopo l'ultimo si riparte dal primo per le ripetizioni).
**Limite operativo**: Claude non può scrivere nello Storage. Inserisce titolo e testo via SQL; il file audio lo carica Michele dal form di modifica nell'app (e poi imposta i tempi). Il rilevamento automatico dei tempi dalle pause dell'audio è stato provato e non è affidabile: si usa lo strumento manuale.
Sintassi testi: una riga = un verso; `[rosso]` cambia colore da lì in poi (anche sulle righe seguenti); colori: rosso, arancio, oro, giallo, verde, azzurro, blu, viola, rosa, marrone, grigio, nero, arcobaleno, oppure `[#1a2b3c]`; il testo tra `(parentesi)` è più piccolo; una riga che inizia con `(` è la nota del verso sopra; `@1:35.2` (o `@0:12,1:05` per più ripetizioni) all'inizio di riga = tempo/i di inizio del verso; riga vuota = spazio. I colori sono schiariti automaticamente per lo sfondo scuro.
Convenzione colori per gli Orixás (usata nelle due canzoni): Ogun rosso, Oxum oro, Iansã giallo, Oxossi verde, Iemanjá blu, Xangò arancio, Nanà marrone, Oxumarè arcobaleno, Omolù grigio.
Canzoni inserite:
1. "Oi vamos Saravà" (order_index 1) — registrazione (m4a, ~2:33) e tempi da inserire dall'app.
2. "Saravà" — Gloria Zuy Zucchelli (order_index 2, inserita il 2026-10-18, song id `99511642-258c-4ff3-8f5b-b4d2711ba3ae`) — testo presente; audio mp3 (~4:15) e tempi da inserire dall'app. Testo inserito senza tempi (`@`) e senza righe di nota.

## Non fatto per ora (scelta esplicita)
- **Assistente AI integrato in-app** (per fare domande di approfondimento sui contenuti): architettura già scoping (Edge Function + chiave Anthropic separata + cap di spesa/domande), ma rimandato — l'utente preferisce continuare a usare Claude direttamente per ora, senza costi aggiuntivi.

## Piccolo strascico tecnico
Esiste ancora una tabella di scratch `public._stage` in Supabase (usata per importare i libri in chunk; ora contiene anche le parti 300-315 del libro di Laszlo, 400-410 di Wilber e 500-515 di Eisler), innocua e non raggiungibile dall'app/studenti. Può essere eliminata dal Table Editor di Supabase quando capita (i tentativi automatici di `DROP TABLE` vengono bloccati dallo strumento con stato "cancelled", serve farlo manualmente), oppure si può richiedere di ripulirla in una prossima sessione.
