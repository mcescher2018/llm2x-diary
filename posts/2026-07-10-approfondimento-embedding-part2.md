---
layout: post
title: "Approfondimento sugli embedding, seconda parte"
---

Riprendiamo l’`encoder` definito nella prima parte e vediamo come addestrarlo tramite un modello Siamese.

I pesi di cui parlavamo sono semplicemente parametri del modello, e di norma vengono inizializzati a valori casuali.

Con un procedimento iterativo, però, possono essere anche “modellati” in modo che frasi di senso simile (es. “dammi gli ordini” e “dammi il fatturato”) abbiano una distanza euclidea bassa, mentre frasi diverse risultino lontane.

Ecco un esempio di training set in pandas, con i valori della label/distanza obiettivo pari ad 1 per le frasi da giudicare simili e 0 per quelle da ritenere diverse.

```python
data = {
    "text_a": [
        "numero ordini per anno",
        "fatturato totale per cliente",
        "numero ordini per anno",
        "vendite mensili",
        "fatturato totale per cliente"
    ],
    "text_b": [
        "conteggio ordini per anno",
        "ricavi totali per cliente",
        "vendite mensili",
        "numero ordini per anno",
        "numero ordini per anno"
    ],
    "label": [1, 1, 0, 0, 0]
}
```

Per eseguire l'addestramento, il primo passo è convertire gli array di stringhe del training set in array di vettori a 32 posizioni.

Questo si fa con un Tokenizer come il seguente (il codice è da considerarsi insieme al precedente visto che utilizza le costanti MAX_LEN e VOCAB_SIZE)

```python
from tensorflow.keras.preprocessing.text import Tokenizer
from tensorflow.keras.preprocessing.sequence import pad_sequences

tokenizer = Tokenizer(num_words=VOCAB_SIZE, oov_token="<UNK>")
tokenizer.fit_on_texts(df["text_a"].tolist() + df["text_b"].tolist())

seq_a = pad_sequences(tokenizer.texts_to_sequences(df["text_a"]), maxlen=MAX_LEN)
seq_b = pad_sequences(tokenizer.texts_to_sequences(df["text_b"]), maxlen=MAX_LEN)

labels = df["label"].values
```

Questi input passano da due modelli gemelli utilizzati per calcolare la distanza fra rispettivi i output:

```python
input_a = layers.Input(shape=(MAX_LEN,))
input_b = layers.Input(shape=(MAX_LEN,))

emb_a = encoder(input_a)
emb_b = encoder(input_b)

# distanza euclidea
distance = tf.norm(emb_a - emb_b, axis=1, keepdims=True)

siamese = models.Model([input_a, input_b], distance)
```

Il passo finale è dato dalla definizione di una contrastive_loss e dalla sua minimizzazione iterativa:

```python
def contrastive_loss(y_true, d, margin=1.0):
    y_true = tf.cast(y_true, d.dtype)
    pos = y_true * tf.square(d)
    neg = (1 - y_true) * tf.square(tf.maximum(margin - d, 0))
    return tf.reduce_mean(pos + neg)

siamese.compile(
    optimizer=tf.keras.optimizers.Adam(1e-3),
    loss=contrastive_loss
)

siamese.fit(
    [seq_a, seq_b],   
    labels,          
    batch_size=32,
    epochs=10
)
```

La contrastive loss prende due embedding e una label (1 → le frasi sono simili, 0 → le frasi sono diverse) e restituisce la perdita (loss) per quella coppia, cioè quanto il modello sta sbagliando.

Minimizzare tale loss quindi vuol dire allenare il sistema a produrre distanze piccole per frasi simili e distanze grandi per frasi diverse

Il codice per il training sopra riportato produce un log di questo tipo:

```text
Epoch 1/10
1/1 [==============================] - 1s 822ms/step - loss: 0.2393
Epoch 2/10
1/1 [==============================] - 0s 13ms/step - loss: 0.1705
Epoch 3/10
1/1 [==============================] - 0s 6ms/step - loss: 0.1183
Epoch 4/10
1/1 [==============================] - 0s 6ms/step - loss: 0.0839
Epoch 5/10
1/1 [==============================] - 0s 4ms/step - loss: 0.0638
Epoch 6/10
1/1 [==============================] - 0s 7ms/step - loss: 0.0487
Epoch 7/10
1/1 [==============================] - 0s 5ms/step - loss: 0.0367
Epoch 8/10
1/1 [==============================] - 0s 6ms/step - loss: 0.0275
Epoch 9/10
1/1 [==============================] - 0s 8ms/step - loss: 0.0208
Epoch 10/10
1/1 [==============================] - 0s 7ms/step - loss: 0.0160
```

Alla fine del processo il sistema è addestrato ed è dunque pronto per trovare la distanza fra due nuove frasi di input, cosa che si potrà ottenere con un codice tipo il seguente:

```python
text_a = "fatturato totale per cliente"
text_b = "numero ordini per cliente"

seq_a = pad_sequences(tokenizer.texts_to_sequences([text_a]), maxlen=MAX_LEN)
seq_b = pad_sequences(tokenizer.texts_to_sequences([text_b]), maxlen=MAX_LEN)

d = siamese.predict([seq_a, seq_b])
print("Distanza:", d[0][0])
```

Nell'esempio considerato l'output è pari a 0.72, perché le due frasi, pur diverse, sono semanticamente correlate (entrambe parlano di vendite verso clienti):

```text
1/1 [==============================] - 0s 18ms/step
Distanza: 0.72428966
```

Notare che il training del modello Siamese ha aggiornato i pesi dell’`encoder`, che è poi il componente che ci interessava davvero ottenere: un *embedder* minimale, costruito e addestrato da zero.

Questo significa che, terminato l’addestramento, possiamo usare direttamente l’`encoder` per ottenere il vettore rappresentativo di una singola frase, come in questo esempio:

```python
text = "fatturato totale per cliente"

seq = pad_sequences(tokenizer.texts_to_sequences([text]), maxlen=MAX_LEN)

emb = encoder.predict(seq)
print("Embedding:", emb[0][:10], "...")   # stampo solo le prime 10 dimensioni
```

Qui l’`encoder` restituirà un embedding normalizzato di 256 dimensioni (pari all'OUT_DIM della prima parte) come dimostra l'output:

``` text
1/1 [==============================] - 0s 67ms/step
Embedding: [ 0.04486587 -0.06921046  0.00728898 -0.04408914 -0.02778636 -0.02405077 -0.05727383 -0.00229595 -0.13183357  0.15121849] ...
```

È, infatti, proprio questo vettore a 256 posizioni che il modello Siamese ha imparato a organizzare nello spazio in modo che frasi simili finissero vicine, frasi diverse lontane.

Era meglio chiamare una `embed()` senza porsi particolari domande, vero?

Beh, forse si. 

E quanto spiegato sopra è un percorso a sé, perché un *embedding* serio usa meccanismi molto più complessi, come ad esempio il *multi‑head self‑attention* per modellare le relazioni tra parole anche quando sono distanti nella sequenza.

Digressione finita ... pronti per ripartire con il cuore di LLMSQLPROD?

[← Torna all’Index](../index.md) · [Post successivo →](2026-07-31-deterministico-anche-troppo.md)