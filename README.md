# Guida all'uso del compilatore automatico

### 1. Organizzazione dei file
Caricare i sorgenti LaTeX nella cartella `src/`.
* **Progetti Multi-file:** Devono avere un file principale e una cartella `contenuti/` (case-insensitive) per i file inclusi. Le modifiche ai file in `contenuti/` innescano automaticamente la ricompilazione del file padre.

### 2. File PDF Firmati
I file firmati vanno caricati **manualmente** in `docs/`.
Per evitarne la cancellazione automatica, il nome deve terminare obbligatoriamente con `_firmato.pdf` o `_signed.pdf` (es: `verbale_v1.0_firmato.pdf`).

### 3. Nomenclatura (Importante per il Sito Web)
Per consentire allo script Python di estrarre correttamente i metadati e costruire i link del sito, usare questa struttura (in ordine):

`nome_file [data] [versione] [firma]`

Regole formali:
* **Ordine campi:** nome -> data opzionale -> versione opzionale -> firma opzionale.
* **Separatori tra campi:** solo `_` oppure spazio (` `). Non sono ammessi altri separatori tra i campi.
* **Data (opzionale):** formato obbligatorio `YYYY-MM-DD`.
* **Versione (opzionale):** formato `vX`, `vX.Y` oppure `vX.Y.Z`.
* **Firma (opzionale):** solo token finale `firmato` oppure `signed`.

Esempi validi:
* `analisi_requisiti_2026-04-05_v2.1.pdf`
* `analisi requisiti 2026-04-05 v2.1 firmato.pdf`
* `verbale_2026-03-10.pdf`

Esempi non riconosciuti come metadati (restano parte del nome):
* `analisi-requisiti-v2.1.pdf` (usa `-` tra campi)
* `analisi.requisiti.v2.1.pdf` (usa `.` tra campi)
* `analisi_signed_v2.1.pdf` (firma non in posizione finale)

Comportamento sul sito:
* **Lista documenti visibile:** mostra tutti i PDF, tranne i duplicati con stesso identificativo `nome + data + versione` dove viene preferito il file `firmato/signed`.
* **Link stabile:** viene creato solo per documenti con versione, raggruppando per `cartella + nome + data` (la firma non incide). Il link stabile punta sempre alla versione piu alta disponibile.

Come calcolare il link stabile (quando vuoi scriverlo a mano):
1. Parti dal **nome documento** estratto, quindi senza versione e senza firma.
2. Applica la normalizzazione del sito: **Title Case** (ogni parola con iniziale maiuscola, quindi `dei` diventa `Dei`) e rimozione di eventuali caratteri non ASCII.
3. Sostituisci gli spazi con `_` e aggiungi `.pdf`.
4. Anteponi il percorso cartella: `./docs/<cartella>/`.

Esempio diretto:
* File: `Analisi_dei Requisiti v1.0 firmato.pdf`
* Link stabile: `./docs/scuola/Analisi_Dei_Requisiti.pdf`

Esempio pratico (stessa cartella):
* File presenti in `docs/scuola/`: `analisi_requisiti_2026-04-05_v2.0.pdf`, `analisi_requisiti_2026-04-05_v2.1_firmato.pdf`
* Link del file mostrato sul sito (click dalla lista): `./docs/scuola/analisi_requisiti_2026-04-05_v2.1_firmato.pdf`
* Link stabile da condividere (email, slide, documenti esterni): `./docs/scuola/Analisi_Requisiti_2026-04-05.pdf`

Nota: dopo un aggiornamento a `v2.2`, il link mostrato sul sito cambia, ma il link stabile resta uguale e punta automaticamente al PDF piu recente.

### 4. Attivazione Manuale
È possibile avviare la build manualmente dalla scheda **Actions** di GitHub selezionando il workflow e cliccando su "Run workflow".

### 5. Report
Consultare il file `report.md` nella root del repository per verificare i file compilati con successo (con link diretto) e quelli falliti.

### 6. Trigger e Rigenerazione
La build parte in automatico solo modificando file `.tex` in `src/`.
* **Aggiornamento risorse esterne:** Se modificate solo immagini o bibliografia, forzate la build con una modifica fittizia nel `.tex` o eliminando il vecchio PDF da `docs/`.
* **Rigenerazione Totale:** Per ricompilare tutto da zero, **eliminare l'intera cartella `docs/`** e fare il push.
    > ⚠️ **Attenzione:** Questo rimuoverà anche i file firmati manuali, assicuratevi di averne una copia.

# Obiettivi della build di compilazione automatica

L’obiettivo della build implementata con github action è di compilare automaticamente i progetti latex caricati nel repository, mantenendo **coerenza e consistenza** tra i file sorgenti latex e i rispettivi pdf.
Nello specifico la build garantisce che:
- `src/` e `docs/` siano perfettamente allineati;  
- ogni file pdf presente in `docs/` é generato esclusivamente dalla build di github action
- i documenti latex che falliscono la compilazione non avranno il rispettivo documento pdf (nemmeno la versione precedente alla build)
- non esistono PDF orfani o obsoleti;

Attenzione: tutto ció non vale per i file firmati, dovranno essere gestiti manualmente.

# Come funziona il file di compilazione automatica?

### Step 1 – Rilevazione modifiche
- Viene individuato l’ultimo commit creato dalla build automatica (`LAST_COMPILED`) e confrontato con `HEAD`.
- Dal diff si ottiene una lista unica di PDF da rigenerare, composta da:
  - tutti i `.tex` modificati/aggiunti/rinominati (se un file è in una cartella `contenuti/`, si considera il relativo file padre nella stessa directory);
  - tutti i `.pdf` in `docs/` modificati/aggiunti/rinominati manualmente (non orfani).

### Step 2 – Pulizia
- Si eliminano i PDF corrispondenti agli elementi identificati nello Step 1 (se esistono).
- Si eliminano i PDF orfani (`.pdf` presenti in `docs/` senza il rispettivo `.tex` in `src/`).
  - Eccezione: i file che terminano con `firmato.pdf` o `signed.pdf` non vengono eliminati se orfani.

### Step 3 – Creazione della lista di compilazione
- Scansionando `src/` si genera un lista di file `.tex` da compilare (detta `compile_list.txt`), composta da tutti i file `.tex` in `src/` a cui manca il rispettivo `.pdf` in `docs/`. Nella scansione si escludono i file nelle cartelle `contenuti/`.

### Step 4 – Compilazione e report
- I file in `compile_list.txt` vengono compilati con `latexmk` dentro l’immagine Docker `ghcr.io/xu-cheng/texlive-full:latest`.
- Viene generato/aggiornato il `report.md` con:
  - ✅ elenco dei `.pdf` compilati correttamente (con link ai file);
  - ❌ elenco dei documenti che hanno fallito (con link alla build).
- Se la lista è vuota, il report indicherá che non è stata necessaria alcuna compilazione.

### Step 5 – Commit finale
- Se sono stati generati o aggiornati PDF, il builder crea un commit automatico: `Automated LaTeX build (base: <SHA>)` dove `<SHA>` è il commit della precedente build automatica ritenuta coerente.  
- Questo commit diventa il nuovo punto di riferimento per la prossima build (in pratica, ogni commit “Automated LaTeX build”rappresenta uno snapshot coerente tra `src/` e `docs/`

# Informazioni di compilazione
- Compilatore: `latexmk`  
- Ambiente: Docker `ghcr.io/xu-cheng/texlive-full`