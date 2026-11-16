# Clasificación de la evolución de ventas en pequeños comercios argentinos

Trabajo práctico de Procesamiento de Lenguaje Natural (ITBA, 2Q 2026).

**Integrantes:** Julia Sexe, Isidro Perasso, Juan Martin Hussey, Stefania Ranucci

## ¿De qué se trata?

A partir de la explicación abierta que da una persona encuestada ("¿Por qué piensas que las ventas incrementaron/disminuyeron/permanecieron igual?"), el sistema clasifica la evolución de las ventas del comercio respecto del mes anterior en **Peor**, **Igual** o **Mejor**. Es una tarea supervisada multiclase en español, con **macro-F1** como métrica principal.

Datos: *LIFT Digitization and Performance Index - Argentina - Fall 2025* (ITBA y MIT LIFT Lab), encuesta presencial a comercios de AMBA y Tandil realizada entre el 29/09/2025 y el 28/10/2025.

## Contenido del repo

| Archivo | Descripción |
|---|---|
| `NotebookMIT_PLN_Entrega_Final.ipynb` | Notebook principal: carga y control de calidad, exploración, agrupación para evitar leakage, validación cruzada, comparación de modelos, evaluación en test, análisis de errores y exportación del modelo final. |
| `Notebook_demo_modelo.ipynb` | Demo interactiva (Gradio) del modelo final. |

También contiene una carpeta Entrega 1, donde realizamos la primera aproximación al trabajo. Ahí dentro está `NotebookMIT_PLN.ipynb` donde se realiza un análisis exploratorio inicial para conocer la base de datos y detectar las variaciones de limpieza de datos que podíamos probar en los experimentos.

El CSV original no está incluido: la notebook principal lo descarga desde un link de Dropbox en la celda de carga.

## Metodología

- **Corpus:** 923 registros, 703 con texto y etiqueta válidos, 697 tras el control de calidad y **689** tras eliminar respuestas formadas solo por puntuación. Textos muy cortos (mediana de 5 tokens).
- **Partición:** test agrupado de 104 observaciones (~15%) y 585 para desarrollo. Los textos duplicados y los registros del mismo comercio quedan siempre en la misma partición.
- **Validación:** `StratifiedGroupKFold` de 5 folds (seed = 42) sobre desarrollo. La selección del modelo se hace solo con el macro-F1 medio de validación cruzada.
- **Sistemas comparados:** baseline de clase mayoritaria, baseline de reglas (regex), BoW/TF-IDF de palabras (32 configuraciones), TF-IDF de caracteres, FastText (embeddings estáticos) y SentenceTransformer, con regresión logística y LinearSVC.
- **Incertidumbre:** bootstrap agrupado de 2.000 repeticiones sobre test.

## Resultados

| Modelo | Macro-F1 CV | Macro-F1 test | IC 95% (bootstrap) |
|---|---|---|---|
| SentenceTransformer + LinearSVC | 0,623 | **0,657** | 0,571 – 0,740 |
| TF-IDF con stemming + regresión logística | 0,587 | 0,580 | 0,488 – 0,673 |
| TF-IDF de caracteres | 0,591 | 0,570 | 0,469 – 0,669 |
| FastText | 0,604 | 0,556 | 0,460 – 0,642 |
| Baseline de reglas | 0,294 | 0,292 | 0,209 – 0,379 |
| Baseline mayoritaria | 0,191 | 0,185 | 0,151 – 0,215 |

El **modelo final** es el **Sentence Transformer** `paraphrase-multilingual-MiniLM-L12-v2` (usado como extractor de embeddings, sin fine-tuning) + `LinearSVC` (`C=0.1`, `class_weight="balanced"`), entrenado sobre el texto original sin limpieza.

Los intervalos del modelo final y de los restantes modelos aprendidos se solapan, por lo que su ventaja debe interpretarse con cautela. La clase **Igual** es la más difícil (recall de 0,444), y también rinden peor las respuestas de 1 o 2 palabras y las que contienen negación. El detalle completo está en el informe.

## Cómo reproducirlo

Las notebooks están pensadas para **Google Colab**, y contiene código Python.

1. Abrir `NotebookMIT_PLN_Entrega_Final.ipynb` en Colab y ejecutarla completa. Las últimas celdas generan y descargan `modelo_demo_ventas.zip` (clasificador entrenado y metadatos).
2. Abrir `Notebook_demo_modelo.ipynb`, ejecutarla y subir `modelo_demo_ventas.zip` cuando lo pida.

Versiones usadas: scikit-learn 1.6.1, sentence-transformers 5.7.0, semilla 42.

## Demo en vivo

Interfaz web hecha con Gradio: se escribe una explicación breve y el modelo devuelve Peor, Igual o Mejor, junto con los puntajes de decisión de LinearSVC (son márgenes, **no probabilidades**).

> Demo: https://d5fcc6cfeb8b79e501.gradio.live
>
> Es un enlace temporal de Gradio: deja de funcionar cuando se detiene la ejecución en Colab y vence como máximo a la semana. Si no abre, se puede volver a lanzar siguiendo los pasos de la sección anterior.

## Presentación del proyecto

Link a la presentación en Canva: https://canva.link/xyrhw5ulvjw25bh

## Limitaciones

- Corpus chico (689 respuestas) y test de solo 104 casos: las métricas tienen bastante incertidumbre.
- Muchas respuestas son de una o dos palabras y ambiguas: una misma palabra, como "Clima", aparece asociada a Mejor y a Igual.
- El modelo clasifica percepciones declaradas a partir del texto: **no establece causas reales** de la variación de las ventas.
- La muestra es de AMBA y Tandil y no es representativa de todos los pequeños comercios del país.
