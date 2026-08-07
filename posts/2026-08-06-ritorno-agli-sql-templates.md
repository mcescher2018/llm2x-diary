---
layout: post
title: "Ritorno agli SQL Templates"
---

La prima lezione appresa è stata che l'abbandono degli SQL generici era quello che ci voleva per dare nuova spinta al progetto. 

E anche la maggiore leggerezza del *semantic model* ha aiutato tantissimo.

Nel caso dell'intent `list_entity` (associato a richieste come "dammi la lista dei clienti") il codice del corrispondente file di SQL template era ritornato molto semplice, essendo questa la sua parte più importante:

```javascript
const table = SEMANTIC_MODEL.entities[params.entity].table;
const sql = `
    SELECT *
    FROM ${table}
    LIMIT 100;
`;
```

Dal *semantic model* si estraeva il nome della tabella fisica corrispondente all'entità semantica presente nel messaggio (es. clienti -> Customers), operativamente ricavata dal *params extractor*. 

Ma la query era adesso una `SELECT * FROM`, nient'altro.

Il caso più complicato era probabilmente il template di `sales_by_dimension` in cui il core era questo:

```javascript
const ctx = prepareBuilderContext(params);
if (ctx.earlyReturn) return ctx.sql;

const { 
    factTable,
    joinClauses,
    dimExpr,
    timeExpr,
    selectClause,
    whereClause
} = ctx;

const groupBy = [dimExpr, timeExpr].filter(Boolean).join(", ");
const limit = params.limit ? `LIMIT ${params.limit}` : "";

const sql = `
    SELECT
        ${selectClause}
    FROM ${factTable}
    ${joinClauses.join("\n")}
    ${whereClause}
    GROUP BY ${groupBy}
    ORDER BY value DESC
    ${limit};
`;
```

In questo caso, l'estrattore dei parametri a monte recuperava dal prompt principalmente la misura (es. fatturato) e le dimensioni esplicite (es. prodotto) o implicite (il tempo e i suoi valori limite). 

La `prepareBuilderContext` aveva una complessità non troppo distante da quella del vecchio *AST builder* ma c'era un vantaggio enorme: questa struttura doveva essere inserita in un comando SQL specifico, quindi c'erano molti meno problemi di ridurre a una gestione comune scenari diversi.

Ad esempio, in questo caso non occorreva "nascondere", per certi input, le parti `WHERE`, `GROUP BY`, `ORDER BY` perchè c'erano sempre, data l'aggregazione dimensionale.

Questo proprio perché gli intent ai quali corrispondevano comandi molto diversi, semplicemente prendevano altre strade.

Il codice non era sempre limpido come quello di LLMREPORT.

Ad esempio, il *join builder* o i *parser* relativi ai filtri temporali erano molto complessi.

Ma, adesso, il caso particolare problematico poteva avere una sua funzione dedicata che interveniva solo per quello.

La scelta del ritorno agli SQL templates è ciò che ha permesso di passare da una applicazione che rispondeva in modo soddisfacente solo nel 40-50% dei casi ad una in cui il tasso di risposte accettabili è diventato dell'80-90%.

Perché, è ovvio, proprio questa percentuale era il principale parametro di qualità del sistema.

Quello che, fra l’altro, ha spostato la mia decisione dal tenere tutto in locale al parlarne in questo blog.

[← Torna all’Index](../index.md) · [Post successivo →](2026-08-07-gestire-la-conversazione.md)