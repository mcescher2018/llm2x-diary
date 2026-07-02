---
layout: post
title: "A pugni con Tyson"
---

Che cos'è concretamente un layer semantico? 

Niente di trascendentale di per sé, consistendo nel mio caso, in questo semplice insieme di file:

- entities.js
- fk_map.js
- grouping_map.js
- kpi_map.js
- synonyms.js

A titolo di esempio, riporto una parte del codice del primo:

```javascript
export const ENTITY_MAP = {
    ordini: {
        name: "orders",
        type: "composite",
        tables: ["OrderLines", "OrderHeaders"],
        dateField: "OrderHeaders.order_date",
        join: [
            { from: "OrderLines.order_id", to: "OrderHeaders.id" }
        ]
    },

    clienti: {
        name: "customers",
        type: "single",
        tables: ["Customers"],
        dateField: null,
        join: []
    },

    prodotti: {
        name: "products",
        type: "single",
        tables: ["Products"],
        dateField: null,
        join: []
    }
};
```

Da qui emerge, fra le varie cose, la natura composita dell'entità logica orders, perchè è fisicamente una join fra due tabelle, una di testata e una di righe.

Come funzionava l'elaborazione dell'AST? 

Beh, dal messaggio in linguaggio naturale - eventualmente sfruttando i sinonimi - si estraeva il nome dell'entità e successivamente si costruiva il suo albero di proprietà più o meno annidate. 

Ecco il pezzo di codice in cui l'entità veniva riconosciuta:

```javascript
export function detectEntity(text) {
    console.log("[Entity] Detecting entity in:", text);
    const words = text.split(/\s+/);

    for (const key of Object.keys(ENTITY_MAP)) {
        if (words.includes(key)) {
            return ENTITY_MAP[key];
        }
    }

    for (const [syn, target] of Object.entries(ENTITY_SYNONYMS)) {
        if (words.includes(syn)) {
            return ENTITY_MAP[target];
        }
    }

    return null;
}
```

In sostanza, a partire dal messaggio, si estraeva un semantic intent che consisteva nella label dell'intento utente più la rappresentazione delle entità coinvolte e dei loro legami:

```javascript
const entity = detectEntity(t);
if (entity) {
    console.log("[Parser] Entity detected:", entity.name);
    ast.entity = entity;
    ast.tables = entity.tables.slice();
    ast.join = (entity.join || []).slice();
}
```

Sì, lo so, sarebbe stato meglio avere un middleware in più, perchè qui si tentava di colmare lo scalino linguaggio naturale -> linguaggio artificiale in un solo colpo. 

Ma l'impressione è che non sarebbe bastato, e il successivo codice dell'SQL Builder, lo illustra molto bene.

*Nota: il codice seguente è solo un estratto, riportato per mostrare la complessità della logica, non per analizzarne ogni dettaglio.*

```javascript
/* ------------------------------------------------------------
* JOIN (fact internal joins + dimension joins)
* ------------------------------------------------------------ */
let joinParts = [];

// 1. Join interne definite dall'entità o dal KPI
if (Array.isArray(ast.join) && ast.join.length > 0) {
    console.log("[Builder] Processing internal joins...");
    for (const j of ast.join) {
        const toTable = j.to.split(".")[0];
        const joinSQL = `JOIN ${toTable} ON ${j.from} = ${j.to}`;
        console.log("  [Internal Join] →", joinSQL);
        joinParts.push(joinSQL);
    }
}

// 2. Join fact → dimension (FK_MAP)
console.log("[Builder] Processing FK_MAP joins...");
for (const fk of FK_MAP) {
    const factHasFrom = ast.tables.includes(fk.fromTable);
    const factHasTo = ast.tables.includes(fk.toTable);

    console.log(`  [FK Check] ${fk.fromTable}.${fk.fromField} → ${fk.toTable}.${fk.toField}`);
    console.log("    factHasFrom:", factHasFrom, "factHasTo:", factHasTo);

    if (factHasFrom && factHasTo) {
        const joinSQL = `JOIN ${fk.toTable} ON ${fk.fromTable}.${fk.fromField} = ${fk.toTable}.${fk.toField}`;
        if (!joinParts.includes(joinSQL)) {
            console.log("  [FK Join Added] →", joinSQL);
            joinParts.push(joinSQL);
        } else {
            console.log("  [FK Join Skipped - duplicate]");
        }
    }
}

const joinClause = joinParts.join("\n");

/* ------------------------------------------------------------
* WHERE
* ------------------------------------------------------------ */
let whereClause = "";
if (Array.isArray(ast.filters) && ast.filters.length > 0) {
    console.log("[Builder] Processing filters...");
    const parts = ast.filters.map(f => {
        const isRawSQLValue =
            typeof f.value === "string" &&
            f.value.match(/^\s*(DATE|datetime|strftime)\s*\(/i);

        const value = isRawSQLValue ? f.value : `'${f.value}'`;
        const filterSQL = `${f.field} ${f.op} ${value}`;
        console.log("  [Filter] →", filterSQL);
        return filterSQL;
    });

    whereClause = "WHERE " + parts.join(" AND ");
}

/* ------------------------------------------------------------
* GROUP BY (supporto MULTIPLO)
* ------------------------------------------------------------ */
let groupByClause = "";
if (ast.groupBy && ast.groupBy.field) {

    // Normalizza in array
    const fields = Array.isArray(ast.groupBy.field)
        ? ast.groupBy.field
        : [ast.groupBy.field];

    // Rimuovi eventuali duplicati
    const uniqueFields = [...new Set(fields)];

    if (uniqueFields.length > 0) {
        groupByClause = `GROUP BY ${uniqueFields.join(", ")}`;
    }
}
```

Le istruzioni, prese singolarmente, erano semplici ma il complesso diventò, almeno per me, quasi ingovernabile. 

Bastava, ad esempio, cambiare una riga con l'obiettivo fixare o migliorare la gestione di un caso, per introdurre problemi in tantissimi altri.

Senza considerare anche che, con tutta questa complessità, si andava a realizzare solo modello specifico di query: quello con SELECT, FROM, WHERE, GROUP BY. 

Niente LEFT, CASE, UNION, subquery, CTE, tabelle temporanee e altre strutture di cui in certi casi è molto difficile fare a meno.

Lo sforzo di parametrizzazione e raffinamento del codice era immane e il risultato, non solo non poteva raggiungere per definizione la generalità cui aspirava, ma era anche estremamente fragile, come mostrarono i test fatti.

Il genio mi aveva portato a confrontarmi con un obiettivo semplicemente troppo ambizioso.

Come sfidare Tyson in una scazzottata e pretendere di vincere, appunto.

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*