# Rastro — Respuestas

Base de datos de una app de segunda mano en MongoDB. El código está en [`rastro.js`](rastro.js).

---

## PARTE A — Diseño

### A1. ¿Por qué MongoDB aquí?

**1. Columnas si cada atributo es una columna**

Columnas comunes: `id, titulo, descripcion, precio, categoria, estado, ciudad, fecha_publicacion, visitas, vendedor_id` (10).

Columnas de atributos (las que se comparten, como `marca` o `modelo`, solo cuentan una vez):
`marca, modelo, anio, km, combustible, almacenamiento_gb, bateria_pct, talla, genero, material, ancho, fondo, alto` (13).

Total: **23 columnas**, de las cuales 13 son de atributos.

**2. Fila de la chaqueta**

| titulo | categoria | precio | marca | modelo | anio | km | combustible | almacenamiento_gb | bateria_pct | talla | genero | material | ancho | fondo | alto |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Chaqueta vaquera Levi's | Ropa | 25 | Levi's | NULL | NULL | NULL | NULL | NULL | NULL | M | unisex | NULL | NULL | NULL | NULL |

De las 13 columnas de atributos solo usa 3 (`marca`, `talla`, `genero`): quedan **10 en `NULL`**.

**3. Llega la categoría Deporte**

Habría que hacer un `ALTER TABLE anuncios ADD COLUMN tipo_bici …, ADD COLUMN pulgadas_rueda …` (`talla` ya existe). Es decir, cambiar el esquema de una tabla en producción, y todos los anuncios que no son de deporte tendrían dos `NULL` más. Cada categoría nueva = otra migración.

**4. Una tabla por categoría**

Para listar «todos los anuncios de Valencia con sus datos» haría falta `anuncios` + `LEFT JOIN coches` + `LEFT JOIN moviles` + `LEFT JOIN ropa` + `LEFT JOIN muebles` = **4 `JOIN`** (y uno más cada vez que se añada una categoría). Tienen que ser `LEFT JOIN` porque cada anuncio solo tiene fila en una de ellas.

**5. Documentos del iPhone y de la mesa**

```javascript
{ titulo: "iPhone 13 128 GB", categoria: "Móviles", precio: 380,
  atributos: { marca: "Apple", modelo: "iPhone 13", almacenamientoGB: 128, bateriaPct: 87 } }

{ titulo: "Mesa de comedor de roble", categoria: "Muebles", precio: 120,
  atributos: { material: "roble", ancho: 160, fondo: 90, alto: 75 } }
```

---

### A2. Anuncio y vendedor

1. **No embebemos el usuario completo** porque (a) el usuario no pertenece a un solo anuncio: tiene varios anuncios, ofertas y valoraciones, así que se repetiría en muchos sitios; y (b) si cambia cualquier dato suyo (email, ciudad…) habría que actualizar todas las copias. Además, la ficha solo necesita su nombre.
2. **Copiamos solo el nombre** porque es lo único que pinta la ficha y casi nunca cambia. El email y la ciudad no hacen falta para la ficha y, si los copiáramos, cada vez que cambiaran habría que actualizar muchos documentos. El email además es un dato personal que no conviene repartir por ahí.
3. Si Luis cambia su nombre hay que actualizar: su documento en `usuarios`, todos sus anuncios (`vendedor.nombre`), sus ofertas (`comprador.nombre`) y sus valoraciones escritas (`autor.nombre`). Lo hice en B7.6.
4. En el gimnasio, la clase guardaba `monitor_id` y para mostrar el nombre se hacía un `JOIN` con la tabla de monitores en cada consulta: el nombre solo estaba en un sitio. Aquí lo copiamos en el anuncio: leer es más rápido (una sola consulta, sin `JOIN`), pero cuando cambia hay que actualizarlo en varios sitios.

---

### A3. Ofertas

1. **Colección `ofertas` aparte.** Aplicando las preguntas:
   - ¿Al leer el anuncio necesito las ofertas? No: la ficha pública no las enseña.
   - ¿Pertenecen solo al anuncio? No: también son del comprador, que quiere ver «mis ofertas» de muchos anuncios distintos. Si estuvieran dentro del anuncio, para ver las ofertas de Sara habría que recorrer todos los anuncios.
   - ¿Pueden crecer sin límite? Sí, un anuncio popular puede recibir muchísimas.

   Tres respuestas a favor de referenciar.

2. Documento de oferta:

   ```javascript
   {
     importe: 340,
     estado: "PENDIENTE",
     fecha: new Date("2026-10-27"),
     anuncio:   { id: ObjectId("670000000000000000000003"), titulo: "iPhone 13 128 GB", precio: 380 },
     comprador: { id: ObjectId("660000000000000000000005"), nombre: "Sara López" }
   }
   ```

3. Copiaría dentro de la oferta el **título** y el **precio** del anuncio (`anuncio: { id, titulo, precio }`), además del id para poder abrirlo. Y del comprador, el nombre, para que el vendedor vea quién oferta.
4. **No debe cambiar.** El precio de la oferta es una «foto» del momento en que se hizo: Sara ofreció 340 € cuando pedían 380 €. Igual que en Furniro la línea de pedido guarda el precio al que se compró aunque luego el producto cambie de precio. Es un dato histórico, no una copia que haya que mantener sincronizada. Lo comprobé en B7.5: tras rebajar el Seat a 5850 €, las ofertas siguen guardando 6500 €.

---

### A4. Valoraciones

1. **Colección aparte.** Un usuario puede recibir muchísimas valoraciones (crecen sin límite) y además se relacionan con tres cosas: autor, valorado y anuncio. En el perfil normalmente solo se enseña la media y las últimas, no todas.
2. Documento:

   ```javascript
   {
     anuncioId: ObjectId("670000000000000000000004"),
     autor: { id: ObjectId("660000000000000000000004"), nombre: "Hugo Sanz" },
     valoradoId: ObjectId("660000000000000000000003"),
     estrellas: 5,
     comentario: "Todo perfecto",
     fecha: new Date("2026-09-15")
   }
   ```

3. Índice único sobre **`autor.id` + `anuncioId`**: un mismo autor no puede valorar dos veces el mismo anuncio (la misma compra).
4. Guardaría en el usuario **`valoracionMedia`** y **`numValoraciones`**. Con los dos se puede recalcular la media al llegar una nueva sin recorrer todas: `nuevaMedia = (media × num + estrellas) / (num + 1)`.

---

### A5. Favoritos (N:M)

1. Un usuario normal tendrá **pocos favoritos** (decenas, como mucho cientos). Un anuncio popular puede estar en los favoritos de **miles** de usuarios.
2. Descarto la **opción b)** (array de usuarios en el anuncio): ese array crecería sin límite, haría el documento del anuncio cada vez más pesado (y es el documento que más se lee) y podría acercarse al límite de 16 MB.
3. La **opción a)** (array de ids en el usuario): leo el usuario y ya tengo la lista de ids; con un `find({ _id: { $in: favoritos } })` saco los anuncios.
4. Filtrando por el array: `db.usuarios.countDocuments({ favoritos: ObjectId("…") })`. Un filtro sobre un array encuentra los documentos que **contienen** ese valor. En B5.9 lo usé para ver quién tiene la mesa en favoritos (Ana y Sara).
5. **`$addToSet`**: añade el valor solo si no está. En B7.3 lo ejecuté dos veces y la segunda dio `modifiedCount: 0`.

---

### A6. ¿Dentro o aparte?

| # | Caso | Decisión | Por qué |
|---|---|---|---|
| 1 | Mensajes de chat comprador–vendedor | **Referenciar** (colección `mensajes` con `anuncioId`, emisor y receptor) | Crecen sin límite y se consultan aparte, no al abrir la ficha. |
| 2 | Dirección de envío de una compra | **Embeber** en la compra/oferta aceptada | Se lee siempre con la compra y debe quedar fija como estaba en ese momento, aunque el usuario cambie luego su dirección. |
| 3 | Etiquetas `["urgente", "negociable"]` | **Embeber** en el anuncio (array de strings) | Son pocas, pequeñas y solo tienen sentido en ese anuncio. |
| 4 | Categorías con nombre, icono y atributos obligatorios | **Referenciar** (colección `categorias`) | Se comparten entre miles de anuncios y cambian solas; en el anuncio basta con el nombre de la categoría. |
| 5 | Historial de cambios de precio | **Embeber** en el anuncio (array `historialPrecios`) | Son pocos cambios, pertenecen solo a ese anuncio y se usan para mostrar «ha bajado de precio» en la ficha. |
| 6 | Los anuncios de un usuario dentro del usuario | **Referenciar** (el anuncio guarda `vendedor.id`) | Los anuncios tienen vida propia, se buscan solos y pueden ser muchos. |
| 7 | Furniro: líneas de un pedido | **Embeber** en el pedido | Siempre se leen con el pedido, no tienen sentido sin él y son pocas. |
| 8 | Furniro: reseñas de un producto | **Referenciar** (colección `resenas`) | Pueden ser miles; como mucho embebería en el producto la nota media y el número de reseñas. |

---

### Apartado 9. Diferencias con mi diseño

Mi diseño coincide con el modelo propuesto: cuatro colecciones (`usuarios`, `anuncios`, `ofertas`, `valoraciones`), fotos y atributos embebidos, vendedor referenciado con nombre copiado, favoritos como array de ids en el usuario y resumen de valoraciones en el usuario.

Lo único que añadiría:
- Un array `historialPrecios` embebido en el anuncio (lo hice en el reto 3).
- Una validación `$jsonSchema` en `anuncios` para que `precio` sea siempre numérico (reto 4), porque en B4.4 se vio que sin ella se cuelan precios como texto.

---

## PARTE B

### B2.3 — Categoría nueva «Deporte»

No tuve que cambiar nada: inserté la bicicleta con sus atributos (`talla`, `tipo`, `pulgadasRueda`) directamente y MongoDB la aceptó. En PostgreSQL habría que haber hecho antes un `ALTER TABLE` para añadir las columnas nuevas (o crear una tabla `deporte`); si no, el `INSERT` habría fallado con «column does not exist».

---

### B4 — Romper la base de datos

| | Predicción | Resultado | Explicación |
|---|---|---|---|
| **B4.1** Usuario con el email de Ana | Fallará, porque en B3.1 creé un índice único sobre `email`. | ❌ `E11000 duplicate key error collection: rastro.usuarios index: email_1 dup key: { email: "ana@rastro.es" }` | Lo ha impedido el **índice único** `email_1`. |
| **B4.2** Hugo valora otra vez a Marta | Fallará por el índice único `autor.id` + `anuncioId`. | ❌ `E11000 duplicate key error collection: rastro.valoraciones index: autor.id_1_anuncioId_1 dup key: { autor.id: ObjectId('6600…04'), anuncioId: ObjectId('6700…04') }` | Lo ha impedido el **índice único compuesto**. |
| **B4.3** Vendedor que no existe | Se guardará: MongoDB no tiene claves ajenas. | ✅ `acknowledged: true, insertedId: ObjectId(…)` | Sí es un problema: queda un anuncio «huérfano» que apunta a un usuario inexistente. Al abrir el perfil del vendedor no habría nada. |
| **B4.4** Precio como texto `"75"` | Se guardará, pero no saldrá en `precio < 100`. | ✅ Se insertó. La búsqueda devolvió Xiaomi (90), Chaqueta (25), Zapatillas (40) y Prueba fantasma (10), **pero no la lámpara**. | Es un problema: el anuncio es invisible para los filtros por precio y ordenaría mal. |
| **B4.5** `updateOne` sin operador | Dará error porque falta `$set`. | ❌ `MongoInvalidArgumentError: Update document requires atomic operators` | Lo ha impedido **mongosh / el driver**: `updateOne` exige operadores (`$set`, `$inc`…) para no sustituir el documento entero por error. (Para sustituir existe `replaceOne`). |
| **B4.6** Buscar por id como texto | Devolverá `null`. | `null` (sin error) | El `_id` es de tipo `ObjectId`, y `"6700…03"` es un `string`: son tipos distintos y no coinciden. Hay que escribir `ObjectId("…")`. |

**B4.7.** En el gimnasio, insertar un `plan_id` que no existe habría dado un error de **clave ajena** (`violates foreign key constraint`) y PostgreSQL no lo habría guardado. En MongoDB no existe esa comprobación: en Rastro la tiene que hacer **la aplicación** (el backend), comprobando que el usuario existe antes de insertar el anuncio. Además, como el vendedor es el usuario que ha iniciado sesión, el backend debería sacar su id de la sesión y no confiar en lo que mande el cliente.

**B4.8.** Porque `"75"` es un **string** y `100` es un **número**. En MongoDB las comparaciones como `$lt` solo comparan valores del mismo tipo (comparación por tipos BSON), así que un texto nunca es «menor que 100». No da error: simplemente no coincide.

**B4.9.** En los dos casos MongoDB **no valida los datos por mí**: no comprueba que una referencia apunte a algo que existe (B4.3) ni que un campo tenga el tipo correcto (B4.4). Solo protege lo que yo le pida explícitamente (índices únicos, o una validación con `$jsonSchema` como en el reto 4). El resto es responsabilidad de la aplicación.

---

### B5.12 — Campos que no existen

No dan error. Los documentos que no tienen `atributos.km` (como el iPhone) simplemente **no cumplen** la condición `$lt: 100000` y no salen en el resultado. Por eso en la consulta añadí también `categoria: "Coches"`, que deja más claro qué busco (y aprovecha el índice).

---

### B6.4 — Valor en venta por vendedor

Resultado (antes de las rebajas de B7.5):

| Vendedor | Valor en venta | Anuncios |
|---|---|---|
| Luis Martín | 6590 € | 2 (Seat 6500 + Xiaomi 90) |
| Hugo Sanz | 4620 € | 3 (Ford 4200 + Mesa 120 + Bici 300) |
| Ana García | 380 € | 1 (iPhone) |
| Sara López | 260 € | 1 (Pixel 7) |
| Marta Ruiz | 25 € | 1 (Chaqueta) |

Captura: `captura-b6-4.png`.

---

### B7 — Notas sobre las modificaciones

- **B7.3:** la segunda vez que añadí el Ford Focus con `$addToSet` el resultado fue `modifiedCount: 0`. **No se duplicó.**
- **B7.4:** necesité **dos instrucciones** en dos colecciones distintas (`ofertas` y `anuncios`). Si se corta la luz entre una y otra, la oferta quedaría `ACEPTADA` pero el anuncio seguiría `DISPONIBLE`: un dato incoherente y otra persona podría comprarlo. En MongoDB una operación es atómica **solo dentro de un documento**; para que las dos vayan juntas habría que usar una **transacción** (`session.startTransaction()` … `commitTransaction()`), como en SQL con `BEGIN … COMMIT`.
- **B7.5:** Seat 6500 → 5850 y Ford 4200 → 3780. Las ofertas siguen guardando el precio antiguo, que es lo correcto (ver A3.4).
- **B7.6:** después de cambiar el nombre en `usuarios`, sus anuncios seguían mostrando «Luis Martín», porque el nombre era una copia. Lo arreglé con `updateMany` en `anuncios` (`vendedor.nombre`, 2 documentos), `ofertas` (`comprador.nombre`, 1 documento) y `valoraciones` (`autor.nombre`, 0 documentos en ese momento, pero hay que hacerlo igual por si las hubiera).

---

### B8 — Explícalo con tus palabras

1. **Dos razones por las que MongoDB encaja en Rastro:** los anuncios tienen atributos distintos según la categoría y salen categorías nuevas sin tocar el esquema; y la ficha del anuncio, que se abre millones de veces, se lee de una sola vez en un documento (fotos, atributos y nombre del vendedor) sin `JOIN`.
2. **Qué guardaría en SQL:** los pagos, facturas y saldos. Ahí la estructura es fija, los datos están muy relacionados y necesito que la base de datos garantice la coherencia con claves ajenas, restricciones y transacciones.
3. **Fotos embebidas y ofertas aparte:** las fotos siempre se leen con el anuncio, no tienen sentido sin él y son como máximo 10. Las ofertas pueden ser muchas, también pertenecen al comprador («mis ofertas») y no se muestran en la ficha.
4. **Copia del precio en la oferta:** para guardar el precio que pedían cuando se hizo la oferta (dato histórico) y poder mostrarlo en «mis ofertas» sin abrir el anuncio. Si el vendedor cambia luego el precio, la oferta no debe cambiar.
5. **Desnormalizar** es copiar datos de un documento en otro para no tener que consultarlo aparte. Ventaja: la ficha del anuncio sale con el nombre del vendedor en una sola consulta. Inconveniente: cuando Luis cambió su nombre (B7.6) tuve que actualizarlo en tres colecciones; si se me olvida una, quedan datos inconsistentes.
6. En MongoDB **la aplicación** se encarga de comprobar que las referencias existen: la base de datos no lo hace (B4.3 se guardó sin problema).

---

## Reto opcional

1. **Nota media:** inserté la valoración de Luis a Ana (4 estrellas) y actualicé a Ana con `valoracionMedia = (media × num + 4) / (num + 1)` y `numValoraciones + 1`. La agregación sobre `valoraciones` da la misma media que la guardada: Ana 4 (1 valoración), Marta 5 (1 valoración).
2. **Ficha con vendedor completo:** con `$lookup` uniendo `anuncios.vendedor.id` con `usuarios._id`. Se parece a un **`LEFT JOIN`** de SQL (`$lookup` devuelve un array, que luego se aplana con `$unwind`).
3. **Historial de precios:** embebido en el anuncio (pocas entradas, pertenece solo al anuncio y se muestra en la ficha). Cada bajada se registra con un único `updateOne` que hace `$set` del precio nuevo y `$push` al array `historialPrecios`, así queda atómico.
4. **Validación:** con `collMod` y `$jsonSchema` exigí que `precio` fuese `int`, `long`, `double` o `decimal`. Al repetir B4.4 ahora falla con `MongoServerError: Document failed validation` (`consideredType: 'string'`). Así MongoDB sí protege el tipo, aunque sigue sin comprobar las referencias.
