# Guida all'uso del compilatore automatico

### 1. Organizzazione dei file e Cartelle
I progetti LaTeX vanno caricati nella cartella `src/`. Il sistema supporta due modalità:
* **Progetti Mono-file:** Un singolo file `.tex` nella root o in una sottocartella.
* **Progetti Multi-file:** È necessario avere un file `.tex` principale (il *main*) e una sottocartella chiamata **`contenuti/`** (o varianti come `Contenuti/`) dove posizionare i capitoli o le sezioni incluse.
    > **Nota tecnica:** Il sistema riconosce automaticamente le modifiche ai file dentro le cartelle "contenuti" e innesca la ricompilazione di tutti i file `.tex` presenti nella cartella superiore (padre) per garantire che il documento principale venga aggiornato.

### 2. Gestione dei PDF Firmati
I file PDF firmati **devono essere caricati manualmente** nella cartella `docs/`.
Per evitare che il sistema di pulizia automatica li elimini (poiché non hanno un corrispettivo `.tex` in `src/`), è obbligatorio che il nome del file termini con:
* `_firmato.pdf`
* `_signed.pdf`

*Esempio corretto:* `verbale_v1.0_firmato.pdf`

### 3. Convenzioni di Nomenclatura (Versionamento)
Per garantire la corretta indicizzazione da parte del sito web e degli script di parsing:
* **Versione:** Inserire la versione alla fine del nome del file (es: `nome_progetto_v0.1.5.tex`).
* **Spazi:** Utilizzare l'underscore `_` o lo spazio standard.
* **Date:** Utilizzare il trattino `-` per le date (es: `2023-10-12`).
* **File Firmati:** Come indicato sopra, aggiungere il suffisso `_firmato` o `_signed` dopo la versione (es: `nome_progetto_v0.1.5_firmato.pdf`).

### 4. Attivazione Manuale
Oltre all'attivazione automatica su `push`, è possibile avviare la build manualmente dalla scheda **Actions** di GitHub selezionando il workflow e cliccando su "Run workflow".

### 5. Report e Debug
Al termine di ogni esecuzione, viene generato il file **`report.md`** nella root del repository. Consultatelo per verificare:
* ✅ Quali file sono stati compilati con successo (con link diretto).
* ❌ Quali file hanno fallito la compilazione (con link ai log di errore).

### 6. Trigger, Aggiornamenti e Rigenerazione Totale
Il sistema parte in automatico quando vengono aggiunti o modificati file `.tex` in `src/`.
Tuttavia, esistono casi particolari per forzare la compilazione:
* **Aggiornamento risorse esterne:** Se modificate solo immagini o bibliografia senza toccare i `.tex`, la build non parte. Per forzare l'aggiornamento, fate una modifica fittizia (es. uno spazio) nel file `.tex` principale oppure eliminate il singolo PDF da `docs/`.
* **Rigenerazione Totale (Full Rebuild):** Per forzare la ricompilazione di **tutti** i documenti presenti nel repository, è sufficiente **eliminare completamente la cartella `docs/`** e fare il push. Il sistema rileverà l'assenza di tutti i PDF e li rigenererà da zero.
    > ⚠️ **Attenzione:** Eliminando l'intera cartella `docs/` verranno persi anche i file firmati manualmente. Assicuratevi di averne una copia di backup da ricaricare successivamente.

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
