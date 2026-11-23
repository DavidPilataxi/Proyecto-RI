# Sistema de Recuperación de Información sobre FiQA

Proyecto de primer bimestre de Recuperación de Información  
**Integrantes:** David Pilataxi y Mijael Molina

[Aplicación pública en Streamlit Community Cloud](https://proyecto-ri-cwfjvw5gyplb9lufzxabup.streamlit.app/) · [Código fuente en GitHub](https://github.com/DavidPilataxi/Proyecto-RI)

## Qué hace la aplicación

Busca en 57.600 documentos de [FiQA](https://huggingface.co/datasets/BeIR/fiqa), un corpus en inglés de preguntas y respuestas financieras. La interfaz permite introducir una consulta libre, elegir un modelo, ajustar el top‑n entre 3 y 20, comparar los cuatro rankings y consultar las métricas precalculadas. Para probar la relevancia, selecciona una consulta del conjunto de prueba. Los resultados etiquetados como relevantes en los *qrels* se marcan con ✅.

| Modelo | Método |
|---|---|
| Jaccard | Intersección sobre unión de los términos distintos de la consulta y el documento. |
| TF‑IDF | Coseno entre vectores con pesos `tf × ln(N/df)`. |
| BM25 | Ponderación por rareza, saturación de frecuencia y normalización por longitud. Parámetros `k1 = 1.5` y `b = 0.75`. |
| Embeddings | Vectores de 384 dimensiones con `all-MiniLM-L6-v2` y búsqueda por distancia coseno en ChromaDB. |

Los modelos clásicos comparten un índice invertido y el mismo preprocesamiento de consultas y documentos. El modelo semántico genera un vector por consulta y reutiliza la colección vectorial ya construida.

## Ejecutar en local

Usa Python 3.12. Desde PowerShell:

```powershell
git clone https://github.com/DavidPilataxi/Proyecto-RI.git
Set-Location Proyecto-RI
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m streamlit run app.py
```

En Linux o macOS, crea el entorno con `python3.12 -m venv .venv` y usa `.venv/bin/python` en los dos últimos comandos. Abre la URL local que imprima Streamlit. La primera ejecución puede requerir Internet para descargar las *stopwords* de NLTK y el modelo preentrenado.

## Datos y artefactos

| Ruta | Contenido |
|---|---|
| `app.py` | Interfaz y carga de recursos. |
| `src/` | Preprocesamiento, índice, modelos y evaluación. |
| `scripts/` | Preparación del corpus, construcción de índices y cálculo de métricas. |
| `data/corpus.jsonl` | Corpus de 57.600 textos filtrados. |
| `data/queries.json`, `data/qrels.json` | Consultas de prueba y juicios de relevancia. |
| `artifacts/index.pkl` | Índice invertido para los modelos clásicos. |
| `artifacts/chroma_db/` | Base persistente con la colección `fiqa`. Se requiere la carpeta completa. |
| `artifacts/metrics.json` | Métricas agregadas y por consulta. |

Para reconstruir los artefactos, activa el entorno virtual y ejecuta desde la raíz, en este orden:

```text
python scripts/prepare_data.py
python scripts/build_all.py
python scripts/build_embeddings.py
```

La reconstrucción vuelve a generar los índices y las métricas. La creación de embeddings para el corpus completo tardó unos 55 minutos en CPU según el registro del proyecto. Para la ejecución normal se usan los artefactos versionados.

## Despliegue

La aplicación se publicó en Streamlit Community Cloud con `app.py` como entrada. La rama desplegada debe contener `requirements.txt`, `src/`, `data/` y `artifacts/`. Si aparece un `FileNotFoundError` al arrancar, comprueba que `artifacts/index.pkl` esté versionado en esa rama. Si la portada abre, prueba una consulta en **Comparar modelos** para verificar la carga de ChromaDB y del modelo semántico. Tras un periodo de inactividad, la aplicación puede pedir que se reactive desde la misma URL.

## Evaluación

Se usaron 648 consultas de la partición de prueba de [FiQA qrels](https://huggingface.co/datasets/BeIR/fiqa-qrels), con 1.705 pares consulta–documento relevantes. Precision@10 y Recall@10 se calculan por consulta y luego se promedian. MAP@100 promedia la precisión promedio calculada hasta el rango 100, con el total de documentos relevantes de cada consulta como denominador.

| Modelo | MAP@100 | Precision@10 | Recall@10 |
|---|---:|---:|---:|
| Jaccard | 0,0355 | 0,0148 | 0,0600 |
| TF‑IDF | 0,1354 | 0,0508 | 0,2159 |
| BM25 | 0,2075 | 0,0704 | 0,3132 |
| Embeddings | 0,3072 | 0,1042 | 0,4383 |

Embeddings obtuvo el mayor promedio en las tres métricas. Los resultados por consulta muestran excepciones. Con la consulta **2076**, sobre deducciones fiscales por gastos veterinarios en Canadá, recuperó ambos relevantes en los dos primeros puestos (AP@100 = 1,0000), mientras BM25 obtuvo 0,0000. Con la consulta **5155**, sobre dos alternativas de pago de un seguro, BM25 ubicó el único relevante primero (AP@100 = 1,0000), mientras Embeddings obtuvo 0,0000.

## Prueba rápida para la defensa

1. Abre la URL y selecciona la consulta **2076** en el selector con qrels.
2. Con top‑n = 10, abre **Comparar modelos**, revisa los cuatro rankings y expande un resultado.
3. Repite con la consulta **5155** y revisa **Métricas** para comparar ambos casos.
