---
layout: post
title: "La posta si alza"
---

Tutti abbiamo visto i grafici dell'effetto Dunning-Kruger, e, forse, ci hanno anche strappato un mezzo sorriso. 

Ben pochi, però, hanno imparato a non caderci: soprattutto quando si sono spinti oltre la loro comfort zone. 

Per non essere da meno anch'io mi sono sentito bravo troppo presto, dopo aver scritto LLMREPORT.

O meglio, ho immaginato che, come era stato facile arrivare fin lì, altrettanto facile sarebbe stato andare oltre. 

LLMREPORT funzionava come atteso e con un'architettura pulita ma presentava dei difetti congeniti che ne rendevano impossibile l'utilizzo pratico. Eccone alcuni:

- Non aveva interfaccia grafica. Questo forse era il problema minore e la risposta era appunto un frontend molto semplice, in React ad esempio, così era anche l'occasione per provarlo.

- Aveva intent molto stretti. Quindi rispondeva a volte in modo "ottuso" perché il classificatore prompt based aveva poche alternative.
In altri termini, gli chiedevi "Dammi il fatturato totale" e lui rispondeva con il fatturato per cliente.

- Era una soluzione estremamente chiusa: di fatto era impossibile aggiungere tabelle o tipi di report senza stravolgerne la struttura e inoltre, dato che affidava la classificazione ad una semplice chiamata LLM, era a rischio enorme di allucinazioni.

Ultimo aspetto, più sociale che tecnico, era che molti mi avevano parlato delle meraviglie di questi server MCP, e io non sapevo neanche cos'erano. La visione di qualche filmato YouTube mi aveva più confuso che aiutato.

L'applicazione aveva un semplice `sqlExecutor` siffatto:

```javascript
import Database from 'better-sqlite3';

const dbPath = process.env.DB_PATH || '/home/master/llmreport/data.db';
const db = new Database(dbPath, { readonly: false });

export function execute(sql) {
  const stmt = db.prepare(sql);
  return stmt.all();
}
```

Impacchettare questa function in un modulo MCP dedicato che in più avesse anche qualche funzione gestita da un adeguato client grafico tramite opportuni slash commands era un metodo naturale di introdurlo nel progetto.

Del resto, per i nuovi sviluppi, avevo al mio fianco una lampada da strofinare e un genio quasi onnipotente, cosa poteva andar storto?

Il problema è che i geni, a volte, hanno anche un lato oscuro. 

[← Torna all’Index](../index.md) · [Post successivo →](2026-07-02-grandi-speranze.md)