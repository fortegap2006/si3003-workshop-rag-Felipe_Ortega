# Respuestas del workshop RAG
Escribe aquí tus respuestas con tus propias palabras. Pega tablas, resultados y enlaces a las capturas de `evidencias/`.

---

# Bloque 0 · Teoría

## T1 · Cuántos nodos viven en cada capa

_Evidencia:_ 
**Codigo python**
<img width="700" height="227" alt="image" src="https://github.com/user-attachments/assets/a1de0e4a-f460-4440-9bd4-2fca89d55d89" />

T1. ¿CUÁNTOS NODOS VIVEN EN CADA CAPA?

CÁLCULO A MANO

Datos:

M = 16
N = 1.000.000 vectores
Probabilidad de llegar al menos a la capa k: P(L >= k) = M^(-k)

La fórmula para calcular el número esperado de nodos es:

E_k = N / M^k

Capa >= 1:

E_1 = 1.000.000 / 16 = 62.500 nodos

Capa >= 2:

E_2 = 1.000.000 / 16^2 E_2 = 1.000.000 / 256 = 3.906,25 nodos

Capa >= 3:

E_3 = 1.000.000 / 16^3 E_3 = 1.000.000 / 4.096 = 244,14 nodos

SIMULACIÓN A MANO

Cada vector tiene una probabilidad de 1/16 de subir a la siguiente capa.

Capa >= 1: de 1.000.000 de vectores, esperamos que 1.000.000 / 16 = 62.500 lleguen a esta capa.
Capa >= 2: de los 62.500 vectores anteriores, esperamos que 62.500 / 16 = 3.906,25 lleguen a esta capa.
Capa >= 3: de los 3.906,25 vectores anteriores, esperamos que 3.906,25 / 16 = 244,14 lleguen a esta capa.

Resultados teóricos:

Capa >= 1: 62.500 nodos Capa >= 2: 3.906,25 nodos Capa >= 3: 244,14 nodos

Estos valores son estimaciones matemáticas. Una simulación real en Python puede producir resultados ligeramente diferentes debido al azar.


_Mi respuesta / análisis: ¿POR QUÉ MEJORA LA VELOCIDAD DE BÚSQUEDA?

Tener pocos nodos en las capas altas permite realizar saltos largos y recorrer rápidamente grandes partes del índice. Después, las capas inferiores permiten encontrar el resultado con mayor precisión, evitando revisar todos los nodos.


## T2 · ef, k y el tamaño de la cola

PREGUNTAS:
Si pides k = 10 vecinos y configuras ef = 5, ¿qué falla y por qué?
R/: ef debe ser siempre ≥ k. Si pides k=10 resultados con ef=5, solo tienes 5 candidatos para elegir — imposible retornar 10.

Según la presentación, ¿con qué valor conviene empezar y cuándo se sube?
R/: con ef = 2k. Si el recall es insuficiente, súbelo (Recall < 95%)

¿Cuál de los dos parámetros, ef o ef_construction, puedes cambiar en producción sin reconstruir el índice? Justifica.
R/: ef ya que subirlo mejora el recall pero aumenta la latencia


## T3 · Inserción frente a búsqueda

_Evidencia:_ 



| Característica | Inserción | Búsqueda |
|---|---|---|
| ¿Se sortea la capa? | Sí, se asigna una altura aleatoria al nuevo nodo para mantener la estructura jerárquica del índice. | No, se recorren las capas existentes porque los nodos ya tienen una altura asignada. |
| Valor de ef en las capas altas | Se utiliza `ef_construction` para controlar los candidatos considerados al conectar el nuevo nodo. | Se utiliza un `ef` pequeño para avanzar rápidamente por las capas superiores hacia la zona del resultado. |
| Parámetro de la fase principal | `ef_construction`, que controla la búsqueda de candidatos durante la construcción del índice. | `ef`, que controla cuántos candidatos se consideran durante la búsqueda. |
| ¿Modifica el índice? | Sí, añade el nodo y crea o actualiza sus conexiones. | No, solo recorre las conexiones existentes para encontrar los vecinos más cercanos. |



_Mi respuesta / análisis:_
R/: La inserción incorpora nuevos documentos al índice, asigna niveles a los nodos y crea conexiones para que puedan encontrarse posteriormente. En cambio, la búsqueda utiliza las conexiones ya construidas para localizar los documentos más relevantes sin modificar el índice.


## T4 · El invariante de FAISSVectorStore


## 1. Tabla de posiciones antes y después de borrar B

Antes de eliminar B, la lista de documentos y el índice FAISS están sincronizados. Si se elimina B únicamente de `self.documents`, las posiciones dejan de corresponder.

| Posición | Documento antes de borrar | Documento después de borrar | Vector FAISS |
|---|---|---|---|
| 0 | A | A | A |
| 1 | B | C | B |
| 2 | C | D | C |
| 3 | D | No existe | D |

Después de borrar B, la lista contiene `[A, C, D]`, pero FAISS conserva los vectores `[A, B, C, D]`.

Si una búsqueda devuelve los índices `[1, 2, 3]`, sucede lo siguiente:

- Índice 1: devuelve C, aunque el vector corresponde a B.
- Índice 2: devuelve D, aunque el vector corresponde a C.
- Índice 3: provoca un error porque la lista solo tiene tres elementos y las posiciones válidas son 0, 1 y 2.

## 2. ¿Qué resultado sería un error silencioso y cuál lanzaría una excepción?

Los índices 1 y 2 producen errores silenciosos, porque devuelven documentos equivocados sin avisar. El índice 3 lanza una excepción `IndexError`, porque intenta acceder a una posición que ya no existe en la lista.

## 3. Relación con las bases vectoriales como Qdrant

Las bases de datos vectoriales especializadas, como Qdrant, utilizan identificadores para relacionar los vectores con sus documentos, en lugar de depender únicamente de las posiciones de una lista de Python. Esto permite gestionar inserciones y eliminaciones de forma más segura y evita que una búsqueda devuelva documentos incorrectos por una desincronización.



## T5 · El costo de la fuerza bruta

Con ~768 millones de multiplicaciones por consulta y 100 consultas por segundo, ¿cuántas multiplicaciones por segundo necesitas?
R/: Tenemos 768 millones de vectores de 768 dimensiones y 100 consultas por segundo.

El cálculo es:

768.000.000 × 768 × 100 = 58.982.400.000.000

Escribe la cifra en notación científica y explica por qué «ser exacto» deja de ser viable.
R/: **5,9 × 10¹³ multiplicaciones por segundo.**
La búsqueda por fuerza bruta debe comparar cada consulta con todos los vectores almacenados. Esto requiere una gran cantidad de operaciones y recursos computacionales. Con millones de vectores y 100 consultas por segundo, el costo de procesamiento, memoria y energía es muy elevado, lo que puede aumentar la latencia y reducir el rendimiento del sistema.


Una frase: ¿qué se sacrifica en ANN (Approximate Nearest Neighbor) y qué se gana?
R/: En ANN (Approximate Nearest Neighbor) se sacrifica la garantía de encontrar siempre el vecino más cercano exacto, pero se gana velocidad y eficiencia computacional, permitiendo procesar más consultas en menos tiempo y con menos recursos.

---

# Bloque A · App RAG empresarial

## A1 · Ejecuta la app RAG de punta a punta

_Evidencia:_ …

_Mi respuesta / análisis:_


## A2 · Tu propio corpus de documentos

_Evidencia:_ …

_Mi respuesta / análisis:_


## A3 · Ingeniería del prompt (antes y después)

_Evidencia:_ …

_Mi respuesta / análisis:_


## A4 · Muestra el score de similitud en las fuentes

_Evidencia:_ …

_Mi respuesta / análisis:_


---

# Bloque B · RAG sobre papers

## B1 · Ejecuta el RAG de papers y evalúa con el ground truth

_Evidencia:_ …

_Mi respuesta / análisis:_


## B2 · Experimentos con chunk_size, k y score_threshold

_Evidencia:_ …

_Mi respuesta / análisis:_


## B3 · Preguntas trampa: ¿el RAG admite que no sabe?

_Evidencia:_ …

_Mi respuesta / análisis:_


## B4 · Preguntas en español sobre papers en inglés

_Evidencia:_ …

_Mi respuesta / análisis:_


## B5 · Escribe tu propio quiz con ground truth

_Evidencia:_ …

_Mi respuesta / análisis:_


---

# Reflexión

## R1 · Reflexión final con tus datos

_Evidencia:_ …

_Mi respuesta / análisis:_


---

# Retos

## C1 · HNSW contra fuerza bruta con FAISS

_Evidencia:_ …

_Mi respuesta / análisis:_


## C2 · Un tercer paper: el RAG que explica HNSW

_Evidencia:_ …

_Mi respuesta / análisis:_


## C3 · App Gradio para los papers con filtro por paper

_Evidencia:_ …

_Mi respuesta / análisis:_


## C4 · Evaluación automática de la recuperación

_Evidencia:_ …

_Mi respuesta / análisis:_


## C5 · MMR frente a similitud pura

_Evidencia:_ …

_Mi respuesta / análisis:_


## C6 · Persistencia del índice en disco

_Evidencia:_ …

_Mi respuesta / análisis:_

