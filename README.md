# Análisis de la Polarización Ideológica en Redes Sociales a partir de la Propagación de Contenidos Desinformativos

Asignatura: Tratamiento de Datos — Máster de Ingeniería de Telecomunicación
Dataset: PHEME — Rumour Detection and Veracity Classification

---

## Descripción del problema

La expansión de la desinformación en redes sociales ha transformado la manera en que los individuos acceden a la información y configuran sus opiniones. Este proyecto estudia la relación entre la difusión de desinformación y la polarización ideológica en Twitter, utilizando el dataset PHEME, que contiene hilos de conversación sobre eventos noticiosos reales junto con anotaciones de veracidad.

Partimos de la hipótesis de que ciertos patrones de desinformación actúan como catalizadores de polarización, intensificando la separación entre grupos sociales mediante el uso de lenguaje alarmista, emotivo y confrontativo.

---

## Nota sobre el contenido del repositorio

Para mantener el repositorio en un tamaño razonable, no se han incluido los ficheros de gran tamaño. En concreto quedan fuera:

- El dataset original PHEME (`data/pheme-rnr-dataset/`), que puede descargarse desde el enlace de Figshare indicado más abajo.
- Los datos intermedios generados durante el procesamiento (`data/processed/` y `data/vectors/`), que se regeneran automáticamente al ejecutar los notebooks 1 y 2.
- El modelo RoBERTa ajustado mediante fine-tuning (`models/`), que ocupa varios cientos de MB y excede el límite práctico de GitHub. Se regenera ejecutando el notebook 3.

Sí se incluyen los cuatro notebooks, todas las figuras generadas (`results/`), esta memoria y el fichero de dependencias. Con descargar el dataset y ejecutar los notebooks en orden se reproduce el trabajo completo.

---

## Descripción del dataset

El dataset PHEME (Zubiaga et al., 2016) contiene tweets organizados en hilos de conversación sobre 9 eventos noticiosos reales, anotados con etiquetas de veracidad.

Fuente: [Figshare — PHEME Dataset](https://figshare.com/articles/dataset/PHEME_dataset_for_Rumour_Detection_and_Veracity_Classification/6392078)

### Estructura

```
pheme-rnr-dataset/
  <evento>-all-rnr-threads/
    rumours/ | non-rumours/
      <thread_id>/
        annotation.json          veracidad del hilo
        source-tweets/<id>.json  tweet original
        reactions/<id>.json      respuestas al tweet
```

### Eventos cubiertos

| Evento | Tweets |
|--------|--------|
| charliehebdo | 38,268 |
| ferguson | 24,175 |
| sydneysiege | 23,996 |
| ottawashooting | 12,284 |
| germanwings-crash | 4,489 |
| prince-toronto | 902 |
| putinmissing | 835 |
| ebola-essien | 226 |
| gurlitt | 179 |

### Estadísticas del dataset

| Métrica | Valor |
|---------|-------|
| Total tweets cargados | 105,354 |
| Total tweets (tras limpieza) | 96,936 |
| Hilos únicos | 6,416 |
| Longitud media del texto | 14.8 palabras |
| Vocabulario total | 50,315 términos únicos |

### Distribución de clases

| Clase | Tweets | % |
|-------|--------|---|
| non-rumour | 66,522 | 68.6% |
| true | 14,042 | 14.5% |
| unverified | 10,438 | 10.8% |
| false | 5,934 | 6.1% |

El dataset presenta un desbalance notable: la clase non-rumour supone el 68.6% de los datos, lo que nos llevó a usar F1-macro como métrica principal en lugar de accuracy.

### Split de datos

```
Train:  67,855 tweets (70%)
Val:    14,540 tweets (15%)
Test:   14,541 tweets (15%)
```

El split es estratificado para mantener la distribución de clases en cada partición.

---

## Metodología

### Notebook 1 — Análisis exploratorio (EDA)

- Descripción general del dataset y análisis de valores nulos y duplicados
- Distribución de clases de veracidad por evento
- Análisis de longitud de textos por clase
- Frecuencias léxicas y nubes de palabras por clase
- Análisis de engagement (retweets, favoritos, seguidores) sobre source tweets
- Formulación de hipótesis de trabajo

### Notebook 2 — Representación vectorial

Comparamos tres estrategias de vectorización.

TF-IDF:
- 20,000 términos, unigramas y bigramas, sublinear TF
- Aplicado sobre todos los tweets (67,855 en train)

Word2Vec (promedio de tokens):
- Implementado sin gensim, que no es compatible con Python 3.14
- Se usa el promedio de tokens RoBERTa excluyendo el token [CLS], equivalente conceptualmente a Word2Vec promediado
- Aplicado solo sobre source tweets (4,483 en train) por limitaciones de CPU

RoBERTa CLS:
- Modelo roberta-base de HuggingFace
- Vector del token [CLS] como representación global del documento
- Aplicado solo sobre source tweets (4,483 en train)

### Notebook 3 — Modelado y evaluación

Se entrenan y evalúan los siguientes modelos:

- SVM (LinearSVC) con TF-IDF y RoBERTa, pesos de clase balanceados
- Random Forest (200 árboles) con TF-IDF y RoBERTa, pesos de clase balanceados
- Red neuronal (PyTorch): arquitectura feedforward con BatchNorm, ReLU y Dropout, entrenada sobre TF-IDF y RoBERTa
- Fine-tuning de roberta-base (HuggingFace) ajustado con 4 epochs, lr=2e-5 y warmup lineal

### Notebook 4 — Extensión: análisis correlacional de polarización

Análisis de la carga emocional y los indicadores lingüísticos de polarización por clase de veracidad:

- VADER para análisis de sentimiento específico de tweets
- TextBlob para polaridad y subjetividad
- Indicadores léxicos: palabras negativas, palabras de alarma, mayúsculas, score de polarización
- Tests estadísticos Kruskal-Wallis y Mann-Whitney
- Correlaciones de Spearman entre desinformación y carga emocional
- Análisis por evento noticioso y perfil emocional por clase

---

## Resultados experimentales

### Comparación de representaciones vectoriales (KNN baseline, val set)

| Representación | Accuracy | F1-macro |
|----------------|----------|----------|
| TF-IDF (todos los tweets) | 0.696 | 0.393 |
| Word2Vec — promedio tokens (source) | 0.753 | 0.639 |
| RoBERTa CLS (source) | 0.746 | 0.634 |

### Evaluación comparativa de modelos (test set)

| Modelo | Accuracy | F1-macro | ROC-AUC |
|--------|----------|----------|---------|
| Fine-tuning RoBERTa | 0.820 | 0.765 | 0.942 |
| Red Neuronal + RoBERTa | 0.797 | 0.724 | 0.924 |
| SVM + RoBERTa | 0.758 | 0.670 | — |
| Random Forest + RoBERTa | 0.710 | 0.474 | 0.901 |
| SVM + TF-IDF | 0.664 | 0.484 | — |
| Red Neuronal + TF-IDF | 0.649 | 0.473 | 0.742 |
| Random Forest + TF-IDF | 0.721 | 0.471 | 0.749 |

La métrica principal es F1-macro, dado el desbalance de clases.

### Detalle del mejor modelo (Fine-tuning RoBERTa)

| Clase | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| false | 0.74 | 0.85 | 0.79 | 96 |
| non-rumour | 0.94 | 0.84 | 0.88 | 601 |
| true | 0.66 | 0.82 | 0.74 | 160 |
| unverified | 0.62 | 0.68 | 0.65 | 104 |
| macro avg | 0.74 | 0.80 | 0.77 | 961 |

### Resultados del análisis de polarización (Notebook 4)

Sentimiento VADER por clase:

| Clase | Negatividad | Positividad | Compound |
|-------|-------------|-------------|----------|
| false | 0.117 | 0.094 | -0.068 |
| unverified | 0.124 | 0.080 | -0.124 |
| true | 0.132 | 0.097 | -0.100 |
| non-rumour | 0.123 | 0.109 | -0.059 |

Score de polarización léxica:

| Clase | Ratio palabras negativas | Ratio alarma | Score polarización |
|-------|--------------------------|--------------|-------------------|
| false | 0.0083 | 0.0074 | 0.1211 |
| unverified | 0.0086 | 0.0055 | 0.1022 |
| true | 0.0100 | 0.0072 | 0.1268 |
| non-rumour | 0.0071 | 0.0036 | 0.0877 |

Correlaciones de Spearman (rumour frente a métricas emocionales):

| Variable | Spearman r | Significación |
|----------|------------|---------------|
| polarization_score | +0.078 | *** |
| ratio_alarm | +0.076 | *** |
| vader_pos | -0.071 | *** |
| vader_compound | -0.039 | *** |
| ratio_negative | +0.032 | *** |
| vader_neg | +0.003 | ns |

---

## Discusión

### Verificación de hipótesis

Hipótesis 1: las distintas clases de veracidad presentan diferencias léxicas detectables. Confirmada. El vocabulario varía entre clases de forma estadísticamente significativa; TF-IDF lo captura lo suficiente para que incluso SVM obtenga F1=0.484, aunque las clases minoritarias siguen siendo un reto.

Hipótesis 2: las representaciones contextuales (RoBERTa) superan a las clásicas (TF-IDF, Word2Vec) en la tarea de clasificación de veracidad. Confirmada. Los modelos sobre RoBERTa superan consistentemente a TF-IDF (F1-macro medio 0.658 frente a 0.476). El fine-tuning alcanza F1=0.765 frente al 0.484 del mejor modelo TF-IDF, una diferencia que resulta bastante clara.

Hipótesis 3: el engagement (retweets, favoritos) actúa como señal de desinformación. Parcialmente confirmada. Los source tweets de rumores tienen mayor mediana de retweets que los non-rumours, pero la variación entre eventos es grande: lo que aplica a charliehebdo no es necesariamente extrapolable a gurlitt.

Hipótesis 4: los contenidos desinformativos presentan mayor carga emocional negativa y signos lingüísticos de polarización que los verdaderos. Parcialmente confirmada. Los contenidos desinformativos usan significativamente más palabras de alarma y obtienen un mayor score de polarización léxica que los non-rumours (p<0.001). Lo que sorprende es que los tweets true son, de media, más negativos que los false en VADER, lo que complica la interpretación: la desinformación parece caracterizarse más por el lenguaje alarmista que por la negatividad emocional en sí.

### Observaciones y limitaciones

La clase unverified es consistentemente la más difícil de clasificar, con F1 entre 0.32 y 0.65 según el modelo. Tiene sentido: sin un veredicto definitivo, los tweets de esta clase son semánticamente ambiguos incluso para un anotador humano.

Random Forest + RoBERTa muestra precisión muy alta en las clases mayoritarias (0.86 a 0.97) pero recall bajo en las minoritarias, lo que sugiere que el modelo aprende a predecir non-rumour con frecuencia a pesar del balanceo de pesos. Con más datos o un ajuste del threshold probablemente se corregiría.

El evento germanwings-crash destaca por la mayor negatividad media (compound -0.119) con un 56% de contenido rumorológico, lo que ilustra cómo el contexto del evento (una tragedia aérea) sesga el tono del lenguaje independientemente de la veracidad.

La Red Neuronal + TF-IDF presenta overfitting claro: la val loss aumenta mientras la train loss cae, lo que sugiere que con 20k features y 67k ejemplos la arquitectura feedforward sin regularización adicional no generaliza bien.

Conviene señalar dos limitaciones de peso. Primero, los embeddings RoBERTa se calcularon solo sobre source tweets (unos 6,400) por restricciones de CPU; con acceso a GPU podría reentrenarse sobre el corpus completo. Segundo, el desbalance de clases (68.6% non-rumour) afecta especialmente a false (6.1%), que es la clase más interesante para detección de desinformación y a la vez la que menos datos tiene.

---

## Conclusiones

El fine-tuning de roberta-base resulta ser el mejor enfoque de los probados, alcanzando F1-macro=0.765 y ROC-AUC=0.942 en test. La diferencia respecto a los modelos clásicos es suficientemente grande como para afirmar que el contexto semántico sí importa en esta tarea.

Dicho esto, hay que tener cuidado con generalizar. Los resultados se obtienen sobre source tweets (unos 6,400 ejemplos) por limitaciones de CPU, no sobre el corpus completo, y el dataset PHEME cubre nueve eventos noticiosos de 2014 y 2015 que no necesariamente representan el ecosistema actual de desinformación.

El análisis de polarización aporta una perspectiva complementaria: los rumores no se distinguen tanto por ser más negativos (los tweets true también lo son) sino por usar más lenguaje de alarma y urgencia. Es un matiz que la clasificación por veracidad no captura directamente y que podría servir como señal adicional en un sistema real.

---

## Estructura del repositorio

```
TratamientoDatosMayo/
├── data/
│   ├── pheme-rnr-dataset/     dataset original (no incluido, descargable de Figshare)
│   ├── processed/             datos procesados (no incluido, generado por NB1)
│   └── vectors/               vectores precomputados (no incluido, generado por NB2)
├── models/                    modelo fine-tuned (no incluido por tamaño, generado por NB3)
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_vectorization.ipynb
│   ├── 03_modeling.ipynb
│   └── 04_extension.ipynb
├── results/                   figuras generadas
├── requirements.txt
└── README.md
```

---

## Instrucciones de reproducción

### 1. Clonar el repositorio y descargar el dataset

```bash
git clone <url-del-repositorio>
cd TratamientoDatosMayo
```

Descargar el dataset PHEME desde [Figshare](https://figshare.com/articles/dataset/PHEME_dataset_for_Rumour_Detection_and_Veracity_Classification/6392078), descomprimir y copiar las 9 carpetas de eventos en `data/pheme-rnr-dataset/`.

### 2. Instalar dependencias

```bash
py -m pip install -r requirements.txt
```

### 3. Ejecutar los notebooks en orden

```
01_eda.ipynb           genera data/processed/ y results/
02_vectorization.ipynb genera data/vectors/ (unos 30-60 min en CPU)
03_modeling.ipynb      entrena modelos y genera results/ (unas 2h en CPU)
04_extension.ipynb     análisis de polarización y resultados finales
```

Los notebooks 2 y 3 son intensivos en CPU. Se recomienda GPU para el fine-tuning de RoBERTa.

---

## Reconocimiento de autoría

- Dataset PHEME: Zubiaga, A., Liakata, M., Procter, R., Hoi, G. W. S., & Tolmie, P. (2016). *Analysing How People Orient to and Spread Rumours in Social Media by Looking at Conversational Threads.*
- Modelo RoBERTa: Liu et al. (2019). *RoBERTa: A Robustly Optimized BERT Pretraining Approach.* Meta AI.
- Análisis de sentimiento VADER: Hutto, C.J. & Gilbert, E.E. (2014). *VADER: A Parsimonious Rule-based Model for Sentiment Analysis of Social Media Text.*
