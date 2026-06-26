---
layout: post
title: "L'ombelico del mondo"
---

A questo punto, probabilmente l’avrete capito: il mio modo di vedere è DB‑centrico.

Spesso non afferro davvero il succo di un software dai diagrammi di flusso o dalle maschere, ma dalle tabelle.  
Il modello relazionale ha per me un’eleganza e una potenza ineguagliabili, qualunque cosa di diverso possa dire la narrazione del momento.

In ogni caso, qui si trattava di sviluppare un’applicazione che arrivasse a operare con un database. Quindi un DB bisognava costruirlo, anche se solo a scopo dimostrativo.

La scelta, per motivi di semplicità e prestazioni, è ricaduta su un file SQLite che rappresentasse un’area vendite: quella che forse, insieme a produzione e magazzino, padroneggio meglio.  
Poche tabelle, sì, con poche righe di contenuto, ma non troppo distanti — a livello logico — dai data mart reali.

Nel file ho inserito due tabelle di dimensioni esplicite (Products, Customers) e due tabelle rappresentative dei fatti elementari: OrderHeaders e OrderLines.  
Due, non una: una complicazione voluta, per simulare il caso reale di informazione del documento (ordine, fattura o quello che è) suddivisa fra più entità, collegate con relazione 1:n.

Legami e FK del mio `data.db` si possono vedere con lo slash command `/show_schema`, quindi non sto a dettagliarli in questo post.

Con una struttura di questo tipo erano possibili:

- Richieste di elencazioni sulle singole dimensioni (es. “dammi la lista dei clienti”)  
- Richieste di elaborazioni su una fact table logica (es. “dammi il fatturato totale”)  
- Utilizzo di misure standard (come fatturato e ordinato) e potenzialmente custom (es. marginalità, comunque definita)  
- Partizionamenti logici fino a un massimo di tre dimensioni (le due esplicite più il tempo, es. “dammi il fatturato per prodotto e per anno”)

Nota che alcuni degli elementi per realizzare quanto sopra non potevano essere contenuti nel DB fisico; li avrei messi nel semantic model e ne parleremo meglio più avanti.

Il DB, essendo il server text‑only, l’ho creato e navigato con strumenti a linea di comando.

Fra questi, una menzione speciale la merita **litecli**, soprattutto per i suoi meccanismi di proposta, gli elenchi a discesa di tabelle e campi e tutte le funzioni di velocizzazione delle operazioni comuni che offre. Un esempio di utilizzo:

```bash
$ litecli data.db
LiteCli: 1.17.1 (SQLite: 3.45.1)
GitHub: https://github.com/dbcli/litecli
data.db> SELECT * FROM OrderLines
+----+----------+------------+-----+------------+
| id | order_id | product_id | qty | unit_price |
+----+----------+------------+-----+------------+
| 1  | 1        | 1          | 2   | 10.0       |
| 2  | 1        | 2          | 1   | 20.0       |
| 3  | 2        | 1          | 3   | 10.0       |
| 4  | 3        | 3          | 5   | 5.0        |
+----+----------+------------+-----+------------+
```

Con questo piccolo data mart in mano, potevo finalmente iniziare la mia prima applicazione LLM2SQL.
Ed è lì che le cose sono diventate davvero interessanti.

---

[← Torna all’Index](../index.md) · [Post successivo →](2026-06-26-prima-iterazione.md)