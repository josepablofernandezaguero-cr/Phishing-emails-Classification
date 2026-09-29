[README.md](https://github.com/user-attachments/files/32832119/README.md)
# Detección de correos de phishing con Deep Learning

[![Abrir en Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/TU-USUARIO/deteccion-phishing-emails-nlp/blob/main/notebooks/Proyecto_Phishing_Emails.ipynb)

Clasificación de correos electrónicos en **phishing** o **legítimos** a partir de su texto, con **TensorFlow/Keras**. El proyecto compara tres arquitecturas de procesamiento de lenguaje natural (LSTM apilada, CNN 1D y Transformer), maneja el desbalance de clases con ponderación y explica las predicciones con **LIME**.

**Resultado principal:** el Transformer alcanza **83.6 % de accuracy** en el conjunto de prueba y **detecta el 90.6 % de los correos de phishing**, con una precisión del 73.7 % sobre esa clase.

<img width="840" height="700" alt="matriz_confusion_transformer" src="https://github.com/user-attachments/assets/0bbdaaf3-67dc-4481-9b0d-6b79498c3a74" />

## Problema

El phishing sigue siendo una de las principales vías de entrada de fraudes y ataques. Un filtro automático debe equilibrar dos errores con costos distintos: **no detectar** un correo malicioso (falso negativo) y **marcar** un correo legítimo como sospechoso (falso positivo). Por eso el proyecto no se queda con el accuracy y analiza el desempeño sobre la clase de phishing por separado.

## Datos

| Aspecto | Detalle |
|---|---|
| Fuente | [Phishing Email Dataset](https://www.kaggle.com/datasets/subhajournal/phishingemails) (Kaggle), archivo `Phishing_Email.csv` |
| Tamaño | 18,650 correos: 11,322 legítimos (60.7 %) y 7,328 de phishing (39.3 %) |
| Variables | `Email Text` (texto del correo) y `Email Type` (clase) |
| Partición | 70 / 15 / 15 estratificada por clase, con semilla fija: **13,055 entrenamiento, 2,797 validación, 2,798 prueba** |

<img width="589" height="543" alt="distribucion_clases" src="https://github.com/user-attachments/assets/0dfc3c24-a549-4c2a-9346-8ff62da25ebc" />

## Metodología

**1. Preprocesamiento**
- Limpieza: minúsculas, eliminación de caracteres especiales y de *stopwords* en inglés y español.
- Vocabulario de 30,000 palabras con token para palabras desconocidas. Sobre el corpus completo hay 166,603 palabras distintas, y las palabras fuera del vocabulario representan alrededor del 4.5 % de los tokens.
- Longitud de secuencia de **729 tokens**, el percentil 90 de la longitud de los correos (mediana de 159 palabras), con relleno al final.

**2. Modelos**

| Modelo | Arquitectura |
|---|---|
| LSTM | Embedding de 64 dimensiones, tres capas LSTM de 64 unidades, capa densa de 64 y salida softmax |
| CNN 1D | Embedding de 64 dimensiones, tres capas convolucionales (128, 128 y 64 filtros) con normalización por lotes y max pooling, capa densa de 64 y salida softmax |
| Transformer | Embedding de palabras con codificación de posición (32 dimensiones), un bloque Transformer con 2 cabezas de atención, capa de compuerta, pooling global, dos capas de dropout (0.3) y **ponderación de clases balanceada** |

**3. Entrenamiento:** optimizador Adam, entropía cruzada categórica, lotes de 32, hasta 10 épocas con parada temprana (paciencia de 3 sobre la pérdida de validación) y restauración de los mejores pesos. Para cada arquitectura se probaron tres tasas de aprendizaje.

**4. Interpretabilidad:** LIME sobre seis correos de prueba (tres de phishing y tres legítimos) clasificados correctamente por el Transformer con confianza de al menos 0.70, para ver qué palabras empujan cada decisión.

## Resultados (conjunto de prueba, 2,798 correos)

**Accuracy por modelo y tasa de aprendizaje**

| Modelo | Tasa de aprendizaje | Accuracy |
|---|---|---|
| LSTM | 1e-3 | 0.621 |
| LSTM | **1e-4** | **0.685** |
| LSTM | 1e-5 | 0.607 |
| CNN 1D | 1e-6 | 0.653 |
| CNN 1D | 2e-6 | 0.680 |
| CNN 1D | **3e-6** | **0.746** |
| Transformer | 1e-7 | 0.424 |
| Transformer | 1e-6 | 0.488 |
| Transformer | **9e-6** | **0.836** |

**Mejor modelo de cada familia, por clase**

| Modelo | Accuracy | Phishing: precisión | Phishing: recall | Legítimo: precisión | Legítimo: recall |
|---|---|---|---|---|---|
| LSTM | 0.685 | 0.88 | 0.23 | 0.66 | 0.98 |
| CNN 1D | 0.746 | 0.68 | 0.66 | 0.78 | 0.80 |
| **Transformer** | **0.836** | 0.74 | **0.91** | 0.93 | 0.79 |

<img width="576" height="455" alt="accuracy_transformer" src="https://github.com/user-attachments/assets/aa2e3b2f-1e3e-4770-91c2-e0d982601a6d" />

## Hallazgos

- **El Transformer es el mejor modelo:** supera a la CNN por unos 9 puntos de accuracy y a la LSTM por unos 15, y es el único que combina un recall alto sobre el phishing (0.91) con un accuracy razonable. De los 1,099 correos de phishing de la prueba detecta 996 y deja pasar 103.
- **Costo en falsos positivos:** 356 de los 1,699 correos legítimos (21 %) se marcan como phishing. Es el precio de priorizar la detección, y el umbral de decisión se puede ajustar según el costo que asuma cada organización.
- **La LSTM casi no detecta phishing:** con un recall de 0.23 tiende a etiquetar todo como legítimo, así que su accuracy engaña. El resultado ilustra por qué, con clases desbalanceadas, hay que mirar las métricas por clase.
- **Los modelos siguen aprendiendo:** las curvas de entrenamiento y validación siguen subiendo en la última época, y para la CNN y el Transformer la mejor tasa de aprendizaje es la más alta de las probadas. Hay margen para mejorar con más épocas y tasas mayores.

## Limitaciones

- **La comparación no es del todo justa:** solo el Transformer se entrenó con ponderación de clases, así que parte de su ventaja sobre la LSTM y la CNN puede deberse a ella y no solo a la arquitectura.
- **Búsqueda de hiperparámetros acotada:** solo se varió la tasa de aprendizaje, con óptimos en el borde de la rejilla, y con un tope de 10 épocas.
- **El mejor modelo se eligió con el mismo conjunto de prueba** con el que se reportan los resultados, lo que tiende a ser optimista.
- No se verificó si hay correos duplicados o casi duplicados entre las particiones, lo que también inflaría los resultados.
- Los resultados corresponden a una sola partición y a un solo entrenamiento por configuración, sin semilla fija para TensorFlow.
- El corpus es mayoritariamente en inglés, así que no hay garantía de que funcione con correos en español.

## Próximos pasos

- Añadir una línea base clásica (TF-IDF con regresión logística o SVM), que sirve de referencia para medir cuánto aporta el aprendizaje profundo.
- Entrenar todos los modelos con la misma ponderación de clases y explorar tasas de aprendizaje mayores y más épocas.
- Elegir el umbral de decisión con curvas precisión-recall y reportar el AUC.
- Ajustar un modelo preentrenado (DistilBERT o similar).
- Validación cruzada y revisión de duplicados entre particiones.
- Publicar una demo interactiva (Gradio o Streamlit) donde se pegue un correo y se obtenga la predicción con su explicación.

## Cómo reproducirlo

Recomendado en Google Colab con GPU (el proyecto se ejecutó con una T4) usando el botón de arriba.

Para correrlo localmente:

```bash
pip install tensorflow scikit-learn pandas numpy matplotlib seaborn nltk kagglehub lime jupyter
jupyter notebook notebooks/Proyecto_Phishing_Emails.ipynb
```

El dataset se descarga automáticamente con `kagglehub`. Las *stopwords* se descargan con `nltk`.

## Créditos

Dataset [Phishing Email Dataset](https://www.kaggle.com/datasets/subhajournal/phishingemails), publicado en Kaggle.

## Autor

José Pablo Fernández Agüero, Estadístico · [LinkedIn](https://www.linkedin.com/in/TU-PERFIL)
