---
layout: post
title: "Approfondimento sugli embedding, prima parte"
---

Questo post è il primo di una coppia che costituisce una digressione rispetto al filo logico principale.

Non è strettamente necessario per seguire LLMSQLPROD, ma per me è importante: mi permette di collegare questa esperienza con gli anni in cui lavoravo su modelli di machine learning più classici.

La funzione embed precedentemente riportata è, nel contesto di LLMSQLPROD (e dei suoi precursori), sostanzialmente una black box.
Questo è il suo codice:

```javascript
export default async function embed(text) {
    if (text == null || text === "") {
        // ritorna un embedding zero-vector
        return Array(384).fill(0);
    }

    if (!embedder) {
        embedder = await pipeline(
            "feature-extraction",
            "Xenova/all-MiniLM-L6-v2"
        );
    }

    const output = await embedder(text, {
        pooling: "mean",
        normalize: true
    });

    return Array.from(output.data);
}
```

La funzione embed mi dice quale modello sta usando, ma non mi dice nulla sulla sua architettura interna.

Ed è normale: un modello pre‑addestrato è una scatola chiusa che restituisce vettori, non spiegazioni.

Per capire cosa succede “dentro” un embedder, conviene costruirne uno minimale, partendo da zero e scegliendo Keras come framework per lo sviluppo.

Non sarà potente come all‑MiniLM, ma è sufficiente per vedere la struttura interna.

Un embedder è, intanto, una pipeline costruita attorno ad una rete neurale.

Le spiegazioni basiche delle reti neurali che ricordo dai tempi dell'università parlavano di cose tipo neuroni, layers, pesi, apprendimento supervisionato e via dicendo. Come conciliare quella visione con la nostra funzione?

Per rispondere, lasciamo da parte per il momento il Javascript e torniamo al Python, scrivendo questo codice con cui inizializziamo alcune variabili e inseriamo una funzione per istanziare un encoder che dell'embedder è il componente principale.

```python
import tensorflow as tf
from tensorflow.keras import layers, models

# Parametri di base
VOCAB_SIZE = 10000      # dimensione vocabolario
MAX_LEN    = 32         # lunghezza massima sequenza
EMB_DIM    = 128        # dimensione embedding
OUT_DIM    = 256        # dimensione vettore finale

def build_encoder():
    inputs = layers.Input(shape=(MAX_LEN,), dtype="int32")
    x = layers.Embedding(VOCAB_SIZE, EMB_DIM, mask_zero=True)(inputs)
    x = layers.GlobalAveragePooling1D()(x)
    x = layers.Dense(OUT_DIM, activation="linear")(x)
    outputs = tf.math.l2_normalize(x, axis=1)
    return models.Model(inputs, outputs)

encoder = build_encoder()
```

Che cos'è un encoder? E' appunto una rete neurale che si aspetta di ricevere in input un vettore di int32 di lunghezza massima 32 e restituisce un vetore di numeri reali a 256 posizioni.

La vera e propria trasformazione la fanno gli hidden layers il primo dei quali è proprio di tipo Embedding che non è un insieme di neuroni nel senso classico (sommatori a soglia collegati da archi con pesi, le cosiddette sinapsi) ma solo una matrice di parametri, in questo caso 10000*128. 

Di queste 10000 posizioni in pratica ne saranno usate solo tante quante sono le parole distinte del training set, ordinandole per frequenza. 

Viene calcolato un vettore per ogni parola, quindi, "Dammi gli ordini" diventa [15,6,121] e l'uscita x di tale layer è un elenco di vettori siffatti:

embedding[15] = [0.12, -0.33, 0.88, ...]
embedding[6] = [-0.44, 0.02, 0.11, ...]
embedding[121] = [0.09, 0.55, -0.21, ...]

Il GlobalAveragePooling1D calcola la media dei vettori parola lungo la sequenza.

Il risultato non ha più lunghezza MAX_LEN, ma EMB_DIM: è un singolo vettore che rappresenta l’intera frase.

Il Dense associa al vettore di sopra a 128 dimensioni un vettore a 256 dimensioni con una trasformazione lineare, anche qui con pesi che danno gradi di libertà. 

La l2_normalize scala il vettore in modo che la sua norma sia 1, il che rende la cosine similarity equivalente a un prodotto scalare: una misura di allineamento tra vettori, non una distanza geometrica.

Questo è tutto sulla topologia dell'encoder. Il post successivo parlerà del suo addestramento.

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*