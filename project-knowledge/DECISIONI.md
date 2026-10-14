# OM Study — Decisioni e policy

> Queste sono le regole stabili del progetto: vanno rispettate in ogni nuova chat/sessione, anche future.

## Policy sui contenuti dei libri (IMPORTANTE)
- **Non si copia mai testo verbatim** dai PDF dei libri (fonte spesso Z-Library) in riassunti, quiz, flashcard o glossario. Tutto va scritto in parole originali da Claude, dopo aver letto il libro per intero.
- **Le lacune informative intenzionali nei quiz vanno bene, anzi sono volute**: può capitare che una domanda richieda un'informazione non presente nel riassunto fornito nell'app. Questo è un meccanismo voluto per spingere lo studente ad attivarsi e leggere il libro vero, non un bug.
- **Le informazioni fattualmente sbagliate NON sono mai accettabili**: se una domanda/risposta è semplicemente sbagliata (non "mancante", ma *errata*), va sempre corretta.
- Il canale per emergere questi casi è la sezione "Segnalazioni domande": gli studenti segnalano, l'admin (Michele) raccoglie le segnalazioni e chiede a Claude di correggere solo gli errori fattuali veri, lasciando stare le lacune volute.
- Si ricorda sempre agli studenti (disclosure) la provenienza Z-Library e il copyright dei libri, raccomandando l'acquisto dell'edizione originale.

## Accesso e inviti
- Niente registrazione libera: gli account si creano solo su invito dell'admin (via Edge Function `invite-user`, che usa la Admin API di Supabase lato server).
- I contenuti (domande, flashcard, riassunti) sono generati da Claude a partire dal materiale del corso fornito da Michele, non scritti a mano da zero.

## Gamification — regole fissate
- XP: 10 per risposta corretta, +20 bonus per lezione perfetta (tutte corrette), dimezzato se la lezione viene ripetuta.
- Livello: `floor(sqrt(xp/100)) + 1`.
- Streak: calcolato su giorno civile in timezone Europe/Rome.
- Flashcard: ripetizione dilazionata stile SM-2 (ease factor, intervallo, data di scadenza) — non un algoritmo semplice a intervalli fissi.
- Badge: criteri multipli (lezioni completate, lezioni perfette, streak, XP totali, carte ripassate, moduli completati, capitoli letti, libri completati).

## Sicurezza / RLS — principi da mantenere
- Gli studenti non scrivono mai direttamente XP o progressi: tutto passa da funzioni RPC `SECURITY DEFINER` (mai scrittura diretta sulle tabelle di gamification).
- Gli studenti non possono leggere le risposte corrette direttamente dalla tabella `questions`: le risposte vivono in `question_keys`, selezionabile solo dagli admin; lo studente la riceve solo come esito di `submit_answer`.
- Le note personali (`notes`) sono strettamente private per utente (RLS `user_id = auth.uid()` su select/insert/update/delete).
- Le segnalazioni (`question_reports`): lo studente può inserire solo le proprie (status forzato a `open`), vederle solo le proprie; solo l'admin può aggiornarle/eliminarle (`is_admin()`).

## Assistente AI in-app — deciso di NON farlo per ora
Michele ha valutato l'idea di integrare Claude come assistente di approfondimento dentro l'app (per rispondere a domande degli studenti su qualsiasi contenuto caricato), ma ha deciso di **non procedere per il momento**: preferisce vedere come funziona l'app senza, dato che può comunque usare Claude direttamente lui stesso senza costi aggiuntivi. Se si deciderà di riconsiderarlo in futuro, l'architettura di base è già definita:
- Edge Function Supabase con chiave Anthropic propria (`ANTHROPIC_API_KEY` nei secrets, mai lato client)
- Risposte basate solo sui riassunti/key_points/glossario già presenti in app (non sui PDF grezzi, per restare nel limite del copyright)
- Cap giornaliero di domande per studente (proposto: 20/giorno come punto di partenza)
- Cap di spesa mensile a livello di organizzazione, impostato nella console Anthropic

## Workflow tecnico per import libri (da riusare per libri futuri)
1. PDF → `pdftotext` (layout mode) → reflow/chunking manuale in Python
2. Claude legge il libro per intero (a blocchi) e scrive riassunti/quiz/flashcard/glossario originali in JSON
3. Il JSON viene caricato in una tabella di staging (`public._stage`, chiave `part` intero) a blocchi, per via dei limiti di dimensione delle singole chiamate
4. Si chiama `import_book(jsonb, boolean)` che crea in un colpo solo modulo + lezioni + domande + question_keys + flashcard + glossario
