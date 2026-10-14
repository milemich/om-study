# OM Study — Stato del progetto

> Ultimo aggiornamento: 2026-10-14

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

Tutti i riassunti/quiz/flashcard/glossario sono stati scritti originalmente da Claude leggendo i PDF, **mai copiando testo verbatim** (vedi DECISIONI.md sulla policy copyright).

## Sezione Note (studenti)
Gli studenti possono prendere note personali mentre leggono un capitolo di un libro, e ritrovarle tutte in una sezione dedicata "Le mie note" con filtro/ricerca ed export (txt, Word, copia negli appunti, stampa/PDF). Le note sono private per utente (RLS su `user_id = auth.uid()`).

## Sezione Segnalazioni domande (studenti → admin)
Gli studenti possono segnalare una domanda del quiz la cui risposta corretta non è presente nel riassunto fornito, con un motivo (enum) + commento libero. L'admin vede le segnalazioni raggruppate per domanda in un pannello di gestione, con azioni "risolvi"/"ignora". Vedi DECISIONI.md per la policy su cosa va corretto.

## Non fatto per ora (scelta esplicita)
- **Assistente AI integrato in-app** (per fare domande di approfondimento sui contenuti): architettura già scoping (Edge Function + chiave Anthropic separata + cap di spesa/domande), ma rimandato — l'utente preferisce continuare a usare Claude direttamente per ora, senza costi aggiuntivi.

## Piccolo strascico tecnico
Esiste ancora una tabella di scratch `public._stage` in Supabase (usata per importare i libri in chunk), innocua e non raggiungibile dall'app/studenti. Può essere eliminata dal Table Editor di Supabase quando capita, oppure si può richiedere di ripulirla in una prossima sessione.
