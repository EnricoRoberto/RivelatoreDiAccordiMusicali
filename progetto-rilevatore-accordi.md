# Rilevatore Accordi & Ritmo — Handoff progetto

File applicazione: `index.html` (singolo file standalone, HTML+CSS+JS, nessuna dipendenza esterna eccetto Google Fonts via CDN). Nome scelto deliberatamente `index.html` (non `rilevatore-accordi.html`) così su GitHub Pages l'URL è diretto: `https://<utente>.github.io/<repo>/` senza dover specificare il nome del file.

**Per continuare in una nuova chat**: allega questo `.md` **insieme al file `index.html` corrente**, poi descrivi cosa vuoi modificare. Questo documento spiega il progetto; l'HTML è il codice su cui lavorare.

## Obiettivo

Pagina web che, tramite microfono, rileva in tempo reale:
1. **L'accordo suonato** (notazione italiana: Do, Re, Mi, Fa, Sol, La, Si), su un'estensione di 3 ottave
2. **Il metro ritmico** (1/4, 2/4, 3/4, 4/4) e BPM stimato

Due modalità di cattura audio (per limitare la dimensione del frame analizzato):
- **Tieni premuto**: cattura finché il pulsante è premuto (cap di sicurezza 4s)
- **Durata fissa**: slider 1–6s, poi avvio con countdown

Due temi grafici alternabili con toggle in alto a destra (non a tendina): **Rock** (font Metal Mania/Rajdhani, palette rosso/nero/ambra) e **Classica** (font Cormorant/EB Garamond, palette avorio/oro/bordeaux, sfondo a righe da pentagramma).

## Architettura tecnica

### Rilevamento accordo (nota-per-nota, non chroma sommato)
- `AnalyserNode` con **FFT 32768** campioni — necessaria per risolvere i semitoni nelle ottave basse (a FFT più piccole, semitoni vicini a Mi2 finiscono nello stesso bin)
- Range di rilevamento: **3 ottave, Mi2–Re♯5** (MIDI 40–75, 36 semitoni), visualizzate come 12 colonne (una per nota) × 3 barrette colorate (una per ottava)
- Per ciascuna delle 36 note candidate, si calcola una **media geometrica pesata** dell'energia spettrale a fondamentale + 2ª/3ª/4ª armonica (vero *Harmonic Product Spectrum*, pesi `[1, 0.7, 0.5, 0.3]`). Scelta deliberata rispetto a una somma pesata: con la somma, una nota mai suonata riceveva comunque punteggio se la sua 2ª/3ª armonica coincideva con una nota vera suonata sopra (falso positivo "un'ottava sotto"). Con la media geometrica, se manca energia alla fondamentale il punteggio crolla anche se un'armonica è forte — testato con accordi sintetici a 3 voci, zero falsi positivi.
- Note "presenti" = massimi locali tra i 36 valori, sopra soglia relativa al picco più forte (`PEAK_REL_THRESHOLD = 0.25`)
- Le note rilevate vengono ripiegate in un vettore a 12 pitch-class e confrontate (cosine similarity) contro template di accordo: Maggiore `[0,4,7]`, Minore `[0,3,7]`, Diminuito `[0,3,6]`, Aumentato `[0,4,8]`, Quinta/power chord `[0,7]`, su tutte le 12 fondamentali
- Soglie di confidenza: `≥0.72` = sicuro, `0.50–0.72` = incerto, sotto = non identificabile
- Richiede almeno 2 pitch-class distinte tra le note rilevate per parlare di "accordo" (altrimenti segnala nota singola)

### Rigetto rumore/silenzio
- Gate su RMS del segnale (soglia calibrabile con pulsante dedicato, 1.2s di campionamento ambientale)
- Gate su **spectral flatness** (rumore bianco/ambientale ha flatness alta → scartato anche se sopra soglia RMS), calcolato su banda 55–5000Hz

### Rilevamento tempo/metro
- Onset detection su derivata dell'RMS (soglia adattiva media+0.8·dev.std, gap minimo 150ms tra onset)
- 0 onset → n/d, 1 onset → **1/4**, ≥2 onset → BPM da mediana degli inter-onset-interval, metro **2/4, 3/4 o 4/4** scelto massimizzando il contrasto d'accento a gruppi modulari
- È una stima euristica su finestre brevi (non un beat-tracker robusto); dichiarato in UI quando l'affidabilità è bassa

### Layout
- Ordine pannelli: header (titolo+toggle tema) → stato mic/calibrazione/diagnostica → controlli cattura → **risultato** (accordo + note rilevate + tempo/BPM) → visualizzazione live (waveform + 12 colonne × 3 ottave + legenda colori) → suggerimenti
- Il pannello risultato è stato **deliberatamente spostato sopra** la visualizzazione live (non è più `position:fixed` in fondo pagina) perché su mobile risultava tagliato/non visibile senza scroll
- Visualizzazione a 3 ottave **compattata** in un'unica riga di 12 colonne (non 3 righe separate) per stare in una schermata o poco più
- Result panel: mobile-first, stack verticale con testo a capo (niente più troncamento), `font-size: clamp(...)` responsivo; da 640px in su torna affiancato

## Deploy

Pensato per essere ospitato su **GitHub Pages** (il microfono richiede secure context: HTTPS o `localhost`; `file://` locale su Android spesso non funziona se aperto da un'app terza che lo serve con URI `content://`).

File di supporto già preparato: workflow GitHub Actions per deploy automatico su Pages ad ogni push su `main` (percorso richiesto nel repo: `.github/workflows/deploy-pages.yml`).

Push effettuato manualmente (upload da browser) o via **Claude Code** in locale (gira sul filesystem dell'utente, usa le credenziali git già configurate — nessun connector MCP GitHub disponibile per claude.ai al momento di questa scrittura).

## Possibili prossimi passi / limiti noti

- Il rilevamento del metro è una stima euristica semplice, non un vero beat-tracker (funziona meglio con ritmi netti e ripetuti, poco affidabile su singoli colpi isolati)
- Nessun rilevamento di settime/estensioni (solo triadi + power chord); estendibile aggiungendo qualità in `QUALITIES`
- FFT 32768 è pesante su mobile di fascia bassa: se emergono problemi di performance nel loop live, valutare di abbassarla leggermente o aumentare il throttle del refresh (attualmente 80ms)
- Nessun salvataggio/storico delle rilevazioni tra sessioni (nessuna persistenza richiesta finora)
