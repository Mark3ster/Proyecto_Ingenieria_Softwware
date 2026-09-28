# Backlog — Base de datos clave-valor (Fase 1)

Origen: entrevista al cliente en clase (S03, 16 sep 2026). Cliente: responsable de negocio de una
empresa con sistemas de paquetería. **Usuario real: los desarrolladores Python de la empresa**, que
usarán la base de datos desde sus aplicativos (no hay interfaz de usuario final).

Formato: *Como <usuario>, quiero <objetivo>, para <razón>*. Cada historia cumple INVEST y tiene
criterios de aceptación verificables (Dado / Cuando / Entonces).

---

## Requisitos funcionales

### HU-01 · Guardar un valor
**Como** desarrollador de la empresa, **quiero** guardar un valor de texto asociado a una clave
(`set(key, value)`), **para** poder recuperarlo más tarde desde mis aplicativos.

- Dado una base de datos vacía, cuando hago `set("age", "15")`, entonces se añade una línea al final
  del fichero de datos y ninguna línea existente se modifica.
- Dado cualquier clave y valor de tipo `str`, cuando hago `set`, entonces la operación no lanza error.
- Dado un valor que no es `str` (p. ej. `bytes` o un `dict`), cuando hago `set`, entonces se lanza
  `TypeError` y no se escribe nada en el fichero.

### HU-02 · Leer un valor
**Como** desarrollador, **quiero** obtener el valor de una clave (`get(key)`), **para** usarlo en mi
aplicativo.

- Dado `set("age", "15")`, cuando hago `get("age")`, entonces devuelve `"15"`.
- Dado una clave que nunca se ha guardado, cuando hago `get("nope")`, entonces devuelve `None`
  (o lanza `KeyError`; el equipo decide y lo documenta).

### HU-03 · Actualizar un valor
**Como** desarrollador, **quiero** que al volver a guardar una clave se devuelva el valor más
reciente, **para** mantener los datos al día.

- Dado `set("age", "15")` y luego `set("age", "28")`, cuando hago `get("age")`, entonces devuelve `"28"`.
- Tras la actualización, el fichero contiene ambas líneas (append-only): no se reescribe la antigua.

### HU-04 · Borrar una clave
**Como** desarrollador, **quiero** borrar una clave (`delete(key)`), **para** que deje de existir en
la base de datos (el equipo legal del cliente exige poder borrar datos).

- Dado `set("age", "15")` y luego `delete("age")`, cuando hago `get("age")`, entonces se comporta
  como una clave inexistente (HU-02).
- El borrado se registra como una línea nueva (*tombstone*); ninguna línea previa se modifica.
- Dado un borrado y un reinicio del proceso, cuando hago `get("age")`, la clave sigue borrada.

### HU-05 · Búsqueda sin recorrer el fichero
**Como** desarrollador, **quiero** que `get` vaya directo al dato, **para** que el tiempo de lectura
no crezca con el tamaño del fichero.

- El índice en memoria es un `dict` que guarda `clave → byte offset`, **no el valor**.
- `get` hace un único `seek` + lectura de línea; no recorre el fichero.
- Dado un fichero con 1 000 000 de claves, cuando hago `get` de la última, entonces responde en
  menos de 500 ms (ver HNF-01).

### HU-06 · Recuperar el índice al reiniciar
**Como** desarrollador, **quiero** que al arrancar la base de datos se reconstruya el índice a partir
del fichero, **para** no perder el acceso a los datos tras un reinicio.

- Dado `set("a", "1")`, `set("a", "2")`, `delete("b")`, y un reinicio del proceso, cuando hago
  `get("a")`, entonces devuelve `"2"`.
- La reconstrucción recorre el fichero una sola vez (O(n)) al arrancar.

### HU-07 · Texto en cualquier idioma
**Como** desarrollador con clientes internacionales (p. ej. Rusia), **quiero** guardar claves y
valores con caracteres no ASCII sin transliterar, **para** que los nombres se lean tal como se
escribieron («Lombraña», no «Lombrana»).

- Dado `set("apellido", "Lombraña")`, cuando hago `get("apellido")`, entonces devuelve exactamente
  `"Lombraña"`.
- Lo mismo con cirílico (`"Иванов"`), chino (`"李"`) y emojis.
- El fichero se escribe y se lee en UTF-8.

### HU-08 · Valores de cualquier longitud
**Como** desarrollador, **quiero** guardar desde un carácter (`"S"` / `"N"`) hasta textos largos
(identificadores de paquete de 32-64 caracteres, claves criptográficas de 512 o más), **para** no
tener que partir los datos.

- Dados valores de 1, 64, 512 y 1 000 000 caracteres, cuando hago `set` y luego `get`, entonces
  cada uno devuelve el valor idéntico.
- Valores con saltos de línea, tabuladores o el carácter separador del fichero se recuperan intactos.

### HU-09 · Módulo fácil de usar *(petición del profesor en la corrección)*
**Como** desarrollador Python, **quiero** importar la base de datos como un módulo e instalarla con
las herramientas habituales, **para** integrarla en mis aplicativos sin fricción.

- Se instala con `pip install .` y se importa con `from <paquete> import ...`.
- El README incluye un ejemplo de uso de `set`, `get` y `delete` que funciona copiándolo tal cual.
- Python 3.12+; código formateado con `black` y sin avisos de `flake8`.

---

## Requisitos no funcionales

### HNF-01 · Rendimiento
El cliente dijo «instantáneo» (**no verificable**). Reescrito:
- `get` y `set` responden en **menos de 500 ms** (objetivo del cliente: «por debajo del segundo»),
  medido con 1 000 000 de claves en el portátil del equipo. Se comprueba con un test de rendimiento.

### HNF-02 · Durabilidad
«No se puede perder ni la última línea registrada», aunque se vaya la luz.
- Cuando `set` o `delete` devuelve, el dato ya está en disco (`flush` + `os.fsync`).
- Test: escribir, matar el proceso (`kill -9`) sin cerrar, reabrir y comprobar que la última
  escritura está.

### HNF-03 · Disponibilidad
Se escribe y consulta también de noche: la base de datos debe estar disponible 24/7.
- *Pendiente:* ventanas de mantenimiento → reunión con el equipo técnico del cliente.

### HNF-04 · Uso de memoria
La RAM no es infinita (el servicio puede tener poca memoria).
- El índice guarda solo offsets: su tamaño crece con el número de claves, no con el tamaño de los
  valores. Test: un valor de 10 MB no aumenta el tamaño del índice.

### HNF-05 · Auditoría *(deseable, no obligatorio)*
**Como** responsable, **quiero** saber quién y cuándo añadió, actualizó o borró cada dato, **para**
tener trazabilidad.
- Cada línea del fichero incluye una marca de tiempo. (El «quién» queda pendiente de definir.)

---

## Contradicciones y dudas abiertas

| # | Tema | Qué pasa | Qué hacer |
|---|------|----------|-----------|
| C1 | **Clave-valor vs. JSON** | Se habló de «JSON», pero el cliente quiere clave-valor plano: **sin datos anidados**. | Los valores son `str`. Sin anidación. |
| C2 | **Rápido vs. seguro** | Quiere las dos cosas. `fsync` en cada escritura es seguro pero más lento. | Medir el coste de `fsync` y valorarlo con el cliente. |
| C3 | **Append-only vs. borrado legal** | El *tombstone* oculta el dato, pero **sigue físicamente en el fichero**. Legal puede exigir que se borre de verdad. | Borrado físico con compactación (Fase 2). Preguntar a legal. |
| C4 | **24/7 vs. mantenimiento** | No sabe si hay ventanas de mantenimiento. | Reunión con el equipo técnico. |
| C5 | **Tamaño máximo** | «Desde un carácter hasta lo que se nos ocurra». | Preguntar al equipo técnico un límite máximo. |
| C6 | **Proporción lectura/escritura** | No la sabe (alguien propuso 90/10). | Preguntar al equipo técnico. |
| C7 | **Legal y ciberseguridad** | El equipo legal y el CISO revisarán «seguro». | Esperar requisitos de seguridad. |

**Siguiente paso recomendado:** pedir una reunión con el equipo de desarrollo del cliente (el usuario
real) y enviarle antes C2-C6 por escrito.
