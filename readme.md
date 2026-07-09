# Javier Alejandro Cáceres Campos - 00068223

## Indicaciones

Recientemente, se utilizó AI para crear un sistema de gestion de una biblioteca, el cual ha generado varios errores, su trabajo es arreglarlo. Dado el siguiente caso de uso, explique y/o resuelva cada problema según se le pida.

---

## Consideraciones

La libreria crea automaticamente un correo con los nombres de la persona

---

## Problemas

### 1. Filtro por autor y género (10%)

QA ha reportado que el endpoint para obtener los libros puede filtrar por **autor** y por **género**, o por cualquiera de los dos de manera individual.

Actualmente:

- Filtrar únicamente por autor funciona correctamente.
- Filtrar únicamente por género funciona correctamente.
- Filtrar por **autor y género al mismo tiempo** provoca que el servidor falle.

**Instrucción:** Explique la causa del problema y resuélvalo.

**R//** Había dos errores en la combinación de filtros. Primero, el método findByAuthorAndGenre en BookRepository declaraba el parámetro genre como String, pero en la entidad Book el campo genre es el enum Genre. Al ejecutar la consulta, Hibernate no puede convertir el String al tipo esperado y lanza una excepción. Segundo, en BookService.getAllBooks los argumentos se pasaban invertidos: findByAuthorAndGenre(genre, author). Se corrigió la firma del repositorio a findByAuthorAndGenre(String author, Genre genre) y la llamada del servicio a findByAuthorAndGenre(author, Genre.valueOf(genre)).

---

### 2. Error al volver a prestar un libro (10%)

Un usuario reportó que al pedir prestado el libro **The Selfish Gene**, devolverlo e intentar pedirlo prestado nuevamente, el servidor falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

**R//** The Selfish Gene tiene avaliable_count inicializado en 1 en data.sql. Al prestarlo, basicamente en el service cuando se crea el movemente, se decrementa a 0 y marca available como false. Al devolverlo, solo incrementaba el contador pero nunca se volvia a poner el estado de available como true. Por eso cuando se intentaba de nuevo prestarlo, la validacion si estaba disponible, lanzaba una runtime exception. 

Se corrigio simplemmente agregando book.setAvailable(true) en la rama de devolucion.

---

### 3. Cantidad de libros por género (10%)

Existe un endpoint que devuelve la cantidad de libros disponibles por género. Sin embargo, actualmente dicho endpoint falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

**R//** En data.sql existe un libro con género NULL (The Art of War). El método getGenresAvailable de BookService llama book.getGenre().name() sobre cada libro, por lo que ese registro provoca un NullPointerException. Se corrigió omitiendo del conteo los libros sin género asignado.

---

### 4. Error al consultar un libro por ID (10%)

Un miembro del equipo de frontend reporta que la siguiente llamada falla:

```http
GET /books?id=ed16ed1e-7017-4697-a08a-d28c09a74acf
```

**Instrucción:** Explique la causa del problema.

**R//** La llamada esta mal construida en sí, el endpoint para consultar un libro por ID esta definido como GET /books/{id}, no como query param. Al llamar GET /books?id=..., la petición entra al endpoint getAllBooks, que solo declara los parámetros opcionales author y genre, ignorando el param id y el endpoint devuelve la lista completa de libros en lugar del libro solicitado. La llamada correcta en todo caso seria asi:

GET /books/ed16ed1e-7017-4697-a08a-d28c09a74acf

Sin el query param.

---

### 5. Error al crear un libro (10%)

QA ha reportado que el siguiente payload enviado al endpoint `POST /books` provoca un error:

```json
{
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "genre": "classic",
  "isbn": "978-0132350884",
  "available": true,
  "availableCount": 5
}
```

**Instrucción:** Explique la causa del problema.

**R//** El payload envía "genre": "classic" en minusculas. BookService.createBook hace Genre.valueOf(dto.getGenre()), y valueOf busca una coincidencia exacta con el nombre de la constante del enum el cual es CLASSIC, por lo que lanza IllegalArgumentException: No enum constant Genre.classic. Enviando "genre": "CLASSIC" la petición funciona correctamente.

---

### 6. Devolución de libros no prestados (20%)

QA ha reportado que un usuario es capaz de devolver libros que nunca ha solicitado en préstamo.

**Instrucción:**

- Confirme si este comportamiento es realmente posible.
- Si es posible, explique la causa y resuelva el problema.
- Si no es posible, explique por qué, haciendo referencia al código correspondiente.

**R//** Sí, el comportamiento era posible. En MovementService.createMovement, la rama de RETURN no validaba nada, solo incrementaba availableCount y registraba el movimiento. Cualquier usuario podía devolver un libro que nunca pidió prestado. Se corrigió agregando countByLectorAndBookAndType en MovementRepository y validando en la devolución que el número de préstamos del lector para ese libro sea mayor que sus devoluciones. Si no tiene un préstamo activo, se lanza una excepción y la devolución se rechaza.

---