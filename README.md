# Guida all'uso del compilatore automatico

### 1. Organizzazione dei file
Caricare i sorgenti LaTeX nella cartella `src/`.
* **Progetti Multi-file:** Devono avere un file principale e una cartella `contenuti/` (case-insensitive) per i file inclusi. Le modifiche ai file in `contenuti/` innescano automaticamente la ricompilazione del file padre.

### 2. File PDF Firmati
I file firmati vanno caricati **manualmente** in `docs/`.
Per evitarne la cancellazione automatica, il nome deve terminare obbligatoriamente con `_firmato.pdf` o `_signed.pdf` (es: `verbale_v1.0_firmato.pdf`).

### 3. Nomenclatura (Importante per il Sito Web)
Per consentire allo script Python di generare correttamente l'albero dei file per il sito GitHub Pages, seguire queste regole:
* **Versione:** Va posta alla fine del nome (es: `nome_progetto_v0.1.5.tex`).
* **Separatori:** Usare `_` o spazi standard. Per le date usare il trattino `-` (es: `2023-10-12`).
* **Firma:** Il suffisso `_firmato` deve seguire la versione (es: `nome_progetto_v0.1.5_firmato.pdf`).

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
