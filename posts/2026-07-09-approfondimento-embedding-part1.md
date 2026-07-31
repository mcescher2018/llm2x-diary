---
layout: post
title: "Approfondimento sugli embedding, prima parte"
---

Questo post è il primo di una coppia che costituisce una digressione rispetto al filo logico principale.

Non è strettamente necessario per seguire LLMSQLPROD, ma è utile per mettere in relazione questa esperienza con approcci di machine learning più tradizionali.

La funzione `embed` precedentemente riportata è, nel contesto di LLMSQLPROD (e dei suoi precursori), sostanzialmente una black box.
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

La funzione `embed` mi dice quale modello sta usando, ma non rivela nulla in termini di sua architettura interna.

Ed è normale: un modello pre‑addestrato è una scatola chiusa che restituisce vettori, non spiegazioni.

Per capire cosa succede “dentro” un *embedder*, conviene costruirne uno minimale, partendo da zero e scegliendo Keras come framework per lo sviluppo.

Non sarà potente come all‑MiniLM, ma è sufficiente per vedere la struttura interna.

Un *embedder* è, intanto, una pipeline costruita attorno ad una rete neurale.

Le spiegazioni basiche sulle reti neurali che ricordo dai tempi dell'università parlavano di neuroni, layers, pesi, apprendimento supervisionato e via dicendo.

Come conciliare quella visione con la nostra funzione?

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

Che cos'è l'*encoder* che abbiamo costruito? È appunto una rete neurale che si aspetta di ricevere in input un vettore di int32 di lunghezza massima 32 e restituisce un vettore di numeri reali a 256 posizioni.

La parte principale della trasformazione del vettore di input la fanno i suoi hidden layers.

Il primo di questi è proprio di tipo `Embedding`: non è un insieme di neuroni nel senso classico (sommatori a soglia collegati da archi con pesi, le cosiddette sinapsi), ma solo una matrice di parametri, in questo caso `10000*128`. 

Di queste 10000 posizioni in pratica ne saranno usate solo tante quante sono le parole distinte del training set, ordinandole per frequenza. 

Verrà calcolato un vettore per ogni parola e, ad esempio, la frase “Dammi gli ordini” verrà convertita dal tokenizer in una sequenza di indici, ad esempio [15, 6, 121], che viene poi *padded* a lunghezza 32:

```text
[0, 0, ..., 0, 15, 6, 121]
``` 

L’Embedding layer trasformerà ciascun indice in un vettore di dimensione 128, producendo una matrice `32×128`.
Le ultime tre righe della matrice corrisponderanno ai vettori delle parole “dammi”, “gli” e “ordini”:

``` text
token_embedding[15]  = [0.12, -0.33, 0.88, ...]
token_embedding[6]   = [-0.44, 0.02, 0.11, ...]
token_embedding[121] = [0.09, 0.55, -0.21, ...]
```

Il `GlobalAveragePooling1D` chiamato nel secondo strato interno del modello calcolerà la media dei vettori parola lungo la sequenza.

Il suo risultato, quindi, non avrà più lunghezza MAX_LEN, ma EMB_DIM: sarà un singolo vettore che rappresenta l’intera frase.

Ad esempio, la frase “Dammi gli ordini” potrebbe produrre un embedding di questo tipo:

``` text
sentence_embedding = [0.031, -0.112, 0.087, 0.044, ...]   # 128 dimensioni
```

Questo vettore è la rappresentazione numerica della frase, che verrà poi trasformata dal layer Dense in un vettore finale a 256 dimensioni e normalizzata tramite l2_normalize.

Quindi, il `Dense` dell'ultimo layer interno, assocerà al vettore di sopra a 128 dimensioni un vettore a 256 dimensioni con una trasformazione lineare, anche qui con pesi che corrispondono a gradi di libertà. 

La `l2_normalize` utilizzata per l'output è una semplice funzione matematica che scala il vettore in modo che la sua norma sia 1.

Ciò renderà la *cosine similarity*, successivamente impiegata per valutare le distanze, equivalente a un prodotto scalare: una misura di allineamento tra vettori, non una distanza geometrica.

Questo è tutto sulla topologia dell'encoder. 

Il post successivo parlerà del suo addestramento.

[← Torna all’Index](../index.md) · [Post successivo →](2026-07-10-approfondimento-embedding-part2.md)