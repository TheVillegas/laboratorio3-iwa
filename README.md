# Laboratorio 3: aplicación inteligente adaptativa

## Identificación

- Integrantes: Matías Villegas
- Curso: Ingeniería Web Avanzada, OII436-1
- URL del repositorio: https://github.com/TheVillegas/laboratorio3-iwa
- Fecha: noviembre de 2026

## Objetivo y alcance

Recomendador de recursos de aprendizaje basado en contenido. Los recursos y los
intereses de cada usuario se representan con TF-IDF sobre la columna `etiquetas`,
se comparan con similitud del coseno y el perfil se actualiza automáticamente
cuando el usuario indica **Me interesa** sobre un recurso. Se usa la versión de
consola; no se implementó la extensión Angular.

Se completaron los cuatro bloques de `motor.py`:

| Función | Implementación |
|---|---|
| `crear_perfil` | `vectorizador.transform([intereses]).toarray()[0]`: reutiliza el vocabulario del catálogo y devuelve un vector 1D propio de cada usuario. |
| `calcular_puntuaciones` | `cosine_similarity([perfil], matriz)[0]`: una puntuación por recurso, en el orden del CSV. Un perfil nulo produce ceros. |
| `recomendar` | Filtra puntuaciones `> 0`, ordena por `(-puntuación, id)` y devuelve como máximo 3 resultados construidos con `construir_resultado`. |
| `actualizar_perfil` | `mezcla = 0.4 * perfil + 0.6 * recurso`, normalizada a norma L2 con `normalize([mezcla])[0]`. |

## Preparación y ejecución

Versión de Python utilizada: **3.14.8** (Linux). Sirve cualquier versión 3.10 o posterior.

```bash
python -m venv .venv
# Windows: py -m venv .venv
```

Activación del entorno. Comando utilizado en este trabajo: `source .venv/bin/activate`.

```bash
source .venv/bin/activate          # macOS o Linux
.venv\Scripts\Activate.ps1         # Windows PowerShell
.venv\Scripts\activate.bat         # Windows CMD
```

Instalación de dependencias y comprobación:

```bash
python -m pip install -r requirements.txt
python -c "import sys; print(sys.executable)"   # debe apuntar a .venv
python app.py --comprobar
```

Resultado observado de `--comprobar`: 12 recursos, 4 usuarios y una matriz TF-IDF
de 12 filas x 17 términos (angular, api, componentes, consultas, datos, formularios,
funciones, graficos, http, interfaces, pandas, python, sql, tablas, validacion,
variables, web).

Ejecución:

```bash
python app.py --demo   # ejecuta T1-T4 y regenera evidencias/
python app.py          # modo interactivo: r recomendar, c catálogo, m me interesa, u usuario, s salir
```

`requirements.txt` se generó con `python -m pip freeze` dentro del entorno virtual.
`.gitignore` excluye `.venv/`, `__pycache__/`, `*.py[cod]`, `.DS_Store` y `node_modules/`.
No se necesitan instrucciones adicionales para reproducir el trabajo desde una copia nueva.

## Catálogo y perfiles (Paso 1)

- `datos/recursos.csv` tiene las columnas `id`, `titulo`, `descripcion` y `etiquetas`.
  Hay 12 recursos, de R01 a R12. **Solo `etiquetas` interviene en el cálculo**;
  `titulo` y `descripcion` sirven únicamente para presentar el recurso.
- Valores vacíos: `cargar_catalogo` lee el CSV con `keep_default_na=False`, quita
  espacios y lanza `ValueError` si alguna de las cuatro columnas queda vacía.
- Identificadores repetidos: `recursos["id"].duplicated()` produce un `ValueError`.
  La aplicación muestra el error y no continúa.
- `datos/perfiles.json`: u1 "python datos pandas", u2 "angular componentes interfaces",
  u3 "" (intereses vacíos válidos) y u4 "zzxyz123" (término fuera del vocabulario).
  u3 y u4 producen un vector nulo, así que reciben una lista vacía y un mensaje informativo.
- Cada usuario tiene su propio vector en `estado[usuario_id]` y su propio historial
  en `historial[usuario_id]`. El estado vive en memoria durante la ejecución; al
  reiniciar se recargan los perfiles del JSON.

## Pruebas T1–T4

Evidencia: `evidencias/reporte_demo.json` y `evidencias/puntuaciones_antes_despues.csv`,
generados con `python app.py --demo` (código de salida 0).

| Caso | Entrada o interacción | Resultado esperado | Resultado observado | Estado |
|---|---|---|---|---|
| T1 | Recomendar para u1 y u2 sin interacciones | Hasta 3 resultados positivos, orden descendente, desempate por ID; R02 primero para u1 y R07 primero para u2 | u1: R02 (1.0000), R12 (0.4965), R03 (0.4326). u2: R07 (1.0000), R08 (0.5677), R09 (0.3087). Ambas listas son válidas. | Cumple |
| T2 | u1 indica Me interesa sobre R04 y se recalcula en la misma ejecución | Cambia el vector de u1; sube R05; R04 pasa al primer lugar; se mantienen los 12 candidatos; se registra el evento | Nuevo ranking: R04 (0.8752), R02 (0.6880), R09 (0.4958). R05 pasa de 0.0000 a 0.4378. Hay 12 puntuaciones antes y 12 después. Historial: `[{numero: 1, accion: "Me interesa", recurso: "R04"}]` | Cumple |
| T3 | Consultar u2 después de la interacción de u1 | El perfil, el historial y las puntuaciones de u2 no cambian | `perfil_sin_cambios`, `puntuaciones_sin_cambios`, `recomendaciones_sin_cambios` e `historial_sin_cambios` dan `true`. Perfil de u2: componentes 0.6575, interfaces 0.5647, angular 0.4988 | Cumple |
| T4 | Recomendar para u3 ("") y u4 ("zzxyz123") | Lista vacía y mensaje informativo, sin error | Ambos obtienen `[]`, todas sus puntuaciones son 0 y aparece el mensaje "No hay coincidencias positivas. Declare intereses presentes en el catálogo." | Cumple |

Prueba adicional en el modo interactivo: con u1, Me interesa sobre `R99` muestra
"No existe el recurso R99." y el perfil de u1 queda igual.

**Interpretación.** T1 muestra que el mismo catálogo da rankings distintos según
el perfil. T2 muestra que la interacción cambia el vector y, con él, las puntuaciones.
T3 muestra que los estados están aislados por usuario. T4 muestra que un vector nulo
no recibe recursos de puntuación cero como si fueran relevantes.

## Comparación de la adaptación (u1 y R04)

Pesos no nulos del perfil de u1:

| Término | Antes | Después |
|---|---|---|
| pandas | 0.753247 | 0.376477 |
| datos | 0.465091 | 0.232455 |
| python | 0.465091 | 0.637116 |
| api | — | 0.446269 |
| web | — | 0.446269 |

Recomendaciones: antes R02, R12 y R03; después R04, R02 y R09.

Puntuaciones de los mismos 12 candidatos (`evidencias/puntuaciones_antes_despues.csv`):

| id | Antes | Después | Diferencia |
|---|---|---|---|
| R01 | 0.1861 | 0.2549 | +0.0688 |
| R02 | 1.0000 | 0.6880 | −0.3120 |
| R03 | 0.4326 | 0.4044 | −0.0282 |
| R04 *(valorado)* | 0.2510 | 0.8752 | +0.6241 |
| **R05** | **0.0000** | **0.4378** | **+0.4378** |
| R06 | 0.0000 | 0.4378 | +0.4378 |
| R07 | 0.0000 | 0.0000 | 0.0000 |
| R08 | 0.0000 | 0.0000 | 0.0000 |
| **R09** | **0.0000** | **0.4958** | **+0.4958** |
| R10 | 0.2053 | 0.1026 | −0.1027 |
| R11 | 0.2053 | 0.1026 | −0.1027 |
| R12 | 0.4965 | 0.4642 | −0.0324 |

**Cambio en recursos no valorados (R05 y R09).** R05 ("web api http") y R09
("angular web api") no recibieron ninguna valoración y tenían puntuación 0, porque
no compartían términos con "python datos pandas". La valoración de R04
("python web api") incorporó al perfil, mediante `0.4*p + 0.6*x`, los términos
**api** y **web** con peso 0.4463. Esos términos aparecen en R05 y R09, así que su
coseno sube a 0.4378 y 0.4958, y R09 entra al top 3. Como el perfil se normaliza,
los términos antiguos (pandas, datos) pierden peso relativo. Por eso bajan R02,
R10, R11 y R12. R07 y R08 siguen en 0 porque no comparten ningún término con el
perfil. El cambio viene del vector actualizado y no de una lista fija: los 12
candidatos son los mismos y se recalculan todos con `calcular_puntuaciones`.

**Problemas detectados.** No hubo pruebas fallidas. El material venía comprimido
con una carpeta anidada y metadatos de macOS (`__MACOSX`); se aplanó la estructura
y se eliminaron esos metadatos. No se modificaron los datos ni `app.py`.

## Preguntas

**1. Contenido y perfil.** El contenido de los recursos está en `datos/recursos.csv`,
en la columna `etiquetas` (por ejemplo, "python web api"), que `construir_indice`
convierte en la matriz TF-IDF. El perfil del usuario está en `datos/perfiles.json`,
en el campo `intereses`, y se convierte en el vector inicial con `crear_perfil`.
Durante la ejecución, el perfil es ese vector actual (`estado[usuario]`) más su
historial de eventos (`historial[usuario]`) en `app.py`.

**2. fit_transform y transform.** `fit_transform` aprende una sola vez, a partir del
catálogo, el vocabulario (17 términos) y los pesos IDF, y vectoriza los recursos.
`transform` proyecta los intereses en ese mismo espacio, con las mismas columnas y
los mismos IDF, lo que permite calcular el coseno. Si se ajustara un vectorizador
para cada perfil, cada uno tendría su propio vocabulario, con otro orden, otra
dimensión y otros IDF. Los vectores serían incompatibles con la matriz: la
dimensión no coincidiría o la columna *i* significaría términos distintos, y las
similitudes no tendrían sentido.

**3. Recomendaciones distintas y 0,80.** La puntuación depende del vector del perfil.
u1 tiene peso en python, datos y pandas; u2 lo tiene en angular, componentes e
interfaces. Por eso el coseno favorece recursos distintos: R02 para u1 y R07 para
u2. Una similitud de 0,80 **no** significa una probabilidad del 80 % de que el
recurso guste. Es el coseno del ángulo entre dos vectores TF-IDF, una medida
geométrica de solapamiento de etiquetas ponderadas. No está calibrada con datos
de preferencias reales ni capta significado: sinónimos con palabras distintas
dan 0.

**4. Edición manual frente a adaptación.** Editar a mano `intereses` en el JSON
cambia el punto de partida, exige reiniciar y no deja traza de por qué cambió.
En la adaptación implementada, el **evento** es "Me interesa" sobre un recurso
(R04). El **estado modificado** es solo el vector actual de ese usuario
(`estado["u1"]`) y se añade el evento a `historial["u1"]`; los intereses iniciales
y los demás usuarios no cambian. El **efecto observable** es el nuevo perfil
(aparecen api y web) y las puntuaciones recalculadas en la misma ejecución
(R05 pasa de 0 a 0.4378 y R04 llega al primer lugar).

**5. MAPE-K.**
- *Monitorizar*: la opción `m` del modo interactivo o `registrar_interes` capturan
  el evento Me interesa y validan el ID con `obtener_indice`.
- *Analizar*: `calcular_puntuaciones` mide la similitud (coseno) entre el perfil y
  cada recurso; se obtiene el vector del recurso valorado, `matriz[indice]`.
- *Planificar*: `actualizar_perfil` decide el nuevo perfil con la regla
  `0.4*p + 0.6*x` y la normalización L2.
- *Ejecutar*: `estado[usuario_id] = nuevo` aplica el cambio y `recomendar` entrega
  el nuevo ranking.
- *Conocimiento*: el vectorizador y la matriz TF-IDF del catálogo,
  `estado` (perfiles actuales) e `historial` (eventos por usuario).

**6. Limitación y mejora.** Solo existe retroalimentación positiva, y con
alpha = 0.6 una sola valoración desplaza mucho el perfil y olvida rápido los
intereses previos: pandas baja de 0.75 a 0.38 con un clic. Además, todo se pierde
al reiniciar. Mejora propuesta: añadir la acción "No me interesa" con una regla
tipo Rocchio, `p' = normalizar(a*p + b*media(positivos) − c*media(negativos))`,
recortando los pesos negativos a 0, y persistir el historial en un archivo o base
de datos. Esto exige un dato adicional (eventos negativos con marca de tiempo por
usuario) y un cambio en el algoritmo: calcular el perfil a partir del historial
completo en lugar de la última mezcla, y ajustar a, b y c con datos reales de uso
en lugar de valores didácticos.

## Archivos de la entrega

`app.py`, `motor.py`, `datos/recursos.csv`, `datos/perfiles.json`,
`evidencias/reporte_demo.json`, `evidencias/puntuaciones_antes_despues.csv`,
`README.md`, `requirements.txt`, `.gitignore` y `link_repositorio.txt`.
