# Network Log Acquisition and Correlation Framework  
## Zeek + ntopng → SQLite Integration

---

## Abstract

Questo repository contiene gli script sviluppati nell’ambito della tesi per l’acquisizione, normalizzazione e correlazione di dati di traffico di rete provenienti da **Zeek** e **ntopng**.  
L’obiettivo della tesi è il confronto tra i due strumenti in termini di:

- capacità di scoperta di nuovi nodi di rete
- quantità e qualità delle informazioni aggiuntive raccolte
- utilità dei dati prodotti per l’integrazione con la piattaforma **NotLine**

Il framework implementa una pipeline di ingestione automatica dei log Zeek in SQLite e una procedura di merge e correlazione multi–sorgente con i dati ntopng, producendo un database integrato pronto per successive elaborazioni e per l’esportazione verso sistemi esterni.

---

## Obiettivo della Tesi

Il lavoro sperimentale ha lo scopo di:

- confrontare Zeek e ntopng nella scoperta di nuovi nodi di rete
- valutare la ricchezza informativa generata dai due sistemi
- misurare il contributo informativo aggiuntivo fornito da ciascun tool
- costruire una pipeline dati riutilizzabile
- preparare i dati per la successiva integrazione nella piattaforma **NotLine**

Gli script qui presenti rappresentano il layer di raccolta, normalizzazione e correlazione dati della pipeline.

---

## Architettura Logica

```
Zeek JSON Logs
      │
      ▼
Directory Monitor (watchdog)
      │
      ▼
Parser + Normalizer
      │
      ▼
SQLite zeek.db (tabelle dinamiche)
      │
      ├───────────────► Merge Engine ◄────────────── ntopng.db
      │
      ▼
merged.db (dataset correlato)
      │
      ▼
Export verso NotLine
```

---

## Componenti del Repository

### Zeek Log Ingestion Engine

Script di monitoraggio e ingestione automatica dei log Zeek in formato JSON.

### Funzionalità

- monitoraggio directory tramite watchdog
- lettura incrementale file `.log`
- parsing JSON line-by-line
- creazione automatica tabelle SQLite
- aggiunta dinamica colonne
- conversione tipi dati
- gestione timestamp
- tabella di stato `file_offsets` per resume sicuro
- prevenzione duplicazioni

### Strategie Implementative

- normalizzazione nomi campo (`.` e `-` → `_`)
- conversione datetime → ISO format
- gestione valori nulli e placeholder
- schema adattivo con ALTER TABLE dinamico

---

### Database Merge and Correlation Engine

Script di integrazione e correlazione tra database ntopng e Zeek.

### Pipeline di Merge

**Fase 1 — Base ntopng**

- lettura tabella `hosts`
- creazione tabella `interesse`
- chiave primaria su IP
- indici su IP e MAC

**Fase 2 — Arricchimento ntopng**

- merge dati dalle altre tabelle ntopng
- aggiornamento record per IP

**Fase 3 — Correlazione Zeek**

Matching basato su:

- IP sorgente
- MAC address layer 2
- UID di connessione Zeek

Per ogni host vengono memorizzati:

- lista UID associati
- JSON completo dei record Zeek
- informazioni di sessione correlate

**Fase 4 — Record non correlati**

- salvataggio in tabella `no_inser_zeek`
- conservazione completa del record originale

---

## Schema Database Resultante

### Tabella `interesse`

Contiene:

- attributi host ntopng
- campi arricchiti ntopng
- UID Zeek associati
- JSON Zeek aggregati

Campi aggiuntivi:

```
zeek_uid_list
zeek_json
```

---

### Tabella `no_inser_zeek`

Contiene record Zeek non correlati:

- uid
- ip
- mac
- payload completo

---

## Ottimizzazioni SQLite

Configurazioni applicate:

```
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
PRAGMA temp_store=MEMORY;
PRAGMA cache_size=-200000;
```

Obiettivi:

- miglior throughput inserimenti
- riduzione lock I/O
- gestione dataset di grandi dimensioni

---

## Requisiti Software

- Python ≥ 3.9
- SQLite ≥ 3
- Zeek con log JSON abilitati
- ntopng con database SQLite

### Dipendenze Python

```
pip install watchdog python-dateutil
```

---

## Configurazione

Aggiornare i path nei file script.

### Zeek Ingestion

```
ZEEK_LOG_DIR = "/path/to/zeek/logs/"
DB_PATH = "zeek.db"
```

### Merge Engine

```
DB_NTOP = "/path/ntopng.db"
DB_ZEEK = "/path/zeek.db"
DB_MERGED = "/path/merged.db"
```

---

## Esecuzione

### Avvio ingestione Zeek

```
python Database_zeek.py
```

Monitoraggio continuo dei log.

---

### Esecuzione merge database

```
python Databae_unificato.py
```

Esecuzione batch del processo di correlazione.

---

## Integrazione con NotLine

Il database finale `merged.db` è progettato come sorgente dati per successive procedure di:

- esportazione strutturata
- trasformazione dati
- import verso piattaforma NotLine
- arricchimento asset e nodi di rete

La struttura JSON per UID Zeek consente una facile serializzazione verso API o pipeline ETL.

---

## Limitazioni

- schema Zeek dinamico → tabelle potenzialmente molto larghe
- matching Zeek–ntopng dipendente da coerenza IP/MAC
- merge basato su euristiche deterministiche
- non progettato per ambienti distribuiti
- codice orientato a sperimentazione accademica

---

## Contesto Accademico

Codice sviluppato come componente sperimentale di tesi in:

- network monitoring
- traffic analysis
- log correlation
- data integration multi–sorgente
- asset discovery

---

## Licenza

Uso accademico e di ricerca.
