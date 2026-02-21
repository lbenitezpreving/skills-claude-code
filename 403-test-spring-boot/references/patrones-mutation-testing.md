# Patrones para Mutation Testing (Pitest)

La cobertura de líneas (JaCoCo) mide si el código se ejecuta. La cobertura de mutaciones (Pitest) mide si los tests **detectan cambios** en el código. Un test débil puede ejecutar código sin verificar su resultado — Pitest lo descubre.

## Configuración de Pitest en build.gradle

```groovy
plugins {
    id 'info.solidsoft.pitest' version '1.15.0'
}

dependencies {
    testImplementation 'org.pitest:pitest-junit5-plugin:1.2.1'
}

pitest {
    junit5PluginVersion    = '1.2.1'
    targetClasses          = ['com.ejemplo.*']
    excludedClasses        = [
        'com.ejemplo.domain.*',         // POJOs — cubiertos por MeanBean
        'com.ejemplo.repository.*',     // interfaces JPA — no tienen lógica
        'com.ejemplo.config.OpenApiConfig',
        '**.*Application'
    ]
    threads                = 4
    outputFormats          = ['HTML', 'XML']
    mutators               = ['DEFAULTS']
    timeoutConstant        = 10000
    failWhenNoMutations    = false
}
```

Ejecutar:
```bash
./gradlew mutation -Dspring.profiles.active=dev
# Reporte en: build/reports/pitest/index.html

# Para un paquete específico:
./gradlew mutation "-PcheckTests=com.ejemplo.service.*"
```

---

## Técnica 1: `hasSize(N)` exacto en lugar de `isNotEmpty()`

Pitest genera mutantes que devuelven `Collections.emptyList()` en lugar de la lista real. `isNotEmpty()` no los detecta; `hasSize(N)` sí.

```java
// ❌ Débil — no mata el mutante "retorna lista vacía"
assertThat(resultado).isNotEmpty();

// ✅ Fuerte — mata el mutante
assertThat(resultado).hasSize(3);
assertThat(resultado).containsExactlyInAnyOrder(elem1, elem2, elem3);
```

---

## Técnica 2: `ArgumentCaptor` para verificar transformaciones

Cuando un servicio transforma datos antes de guardarlos, verificar los valores exactos con `ArgumentCaptor`.

```java
@Test
@DisplayName("crear: persiste la entidad con los valores correctos tras transformación")
void crear_debeGuardarConValoresTransformados() {
    ArgumentCaptor<Producto> captor = ArgumentCaptor.forClass(Producto.class);
    when(productoRepository.save(any())).thenReturn(productoMock);

    productoService.crear(new ProductoCreateDTO("nombre", "ACTIVO"));

    verify(productoRepository).save(captor.capture());
    Producto guardado = captor.getValue();
    assertThat(guardado.getNombre()).isEqualTo("nombre");         // mata mutante en asignación
    assertThat(guardado.getEstado()).isEqualTo(Estado.ACTIVO);    // mata mutante en enum
    assertThat(guardado.getActivo()).isTrue();                    // mata mutante booleano
    assertThat(guardado.getOrden()).isEqualTo(1);                 // mata mutante en constante
    assertThat(guardado.getId()).isNull();                        // id null antes de guardar
}
```

---

## Técnica 3: `ReflectionTestUtils.getField()` para estado privado

Cuando la clase tiene estado interno privado (flags, locks, contadores):

```java
@Test
@DisplayName("importar: libera el lock tras completar la importación")
void importar_seLiberaLockAlFinalizar() {
    when(importService.procesar(any())).thenReturn(ResponseEntity.ok("OK"));

    importController.importar(mockFile, 1);

    // Verificar que el lock quedó en false (no bloqueado)
    AtomicBoolean lock = (AtomicBoolean) ReflectionTestUtils.getField(importController, "importLocked");
    assertThat(lock.get()).isFalse();
}

@Test
@DisplayName("importar: devuelve 422 cuando ya hay una importación en curso")
void importar_lockActivo_devuelve422() {
    AtomicBoolean lock = (AtomicBoolean) ReflectionTestUtils.getField(importController, "importLocked");
    lock.set(true);

    ResponseEntity<?> response = importController.importar(mockFile, 1);

    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNPROCESSABLE_ENTITY);
    verifyNoInteractions(importService);
}
```

---

## Técnica 4: Cubrir todas las ramas de condicionales

Pitest genera mutantes que eliminan condiciones o las invierten. Para cada `if (a && b)` necesitas al menos 3 tests:

```java
// Código de producción:
// if (entidad.isActivo() && entidad.getTipoId() == TIPO_PRINCIPAL) { hacer algo }

@Test
@DisplayName("procesar: ejecuta lógica cuando activo=true Y tipo=PRINCIPAL")
void procesar_activoYPrincipal_ejecutaLogica() {
    Entidad e = crearEntidad(true, TIPO_PRINCIPAL);
    productoService.procesar(e);
    verify(otroServicio, times(1)).ejecutar(any());
}

@Test
@DisplayName("procesar: no ejecuta lógica cuando activo=false (aunque tipo=PRINCIPAL)")
void procesar_inactivo_noEjecuta() {
    Entidad e = crearEntidad(false, TIPO_PRINCIPAL);
    productoService.procesar(e);
    verifyNoInteractions(otroServicio);
}

@Test
@DisplayName("procesar: no ejecuta lógica cuando activo=true pero tipo≠PRINCIPAL")
void procesar_activoPeroTipoSecundario_noEjecuta() {
    Entidad e = crearEntidad(true, TIPO_SECUNDARIO);
    productoService.procesar(e);
    verifyNoInteractions(otroServicio);
}
```

---

## Técnica 5: Verificar valores de retorno exactos

Pitest muta constantes y literales. Verificar valores concretos, no solo que no son null.

```java
// ❌ Débil
assertThat(response.getStatusCode()).isNotNull();

// ✅ Fuerte
assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
assertThat(response.getBody()).isNotNull();
assertThat(response.getHeaders().getLocation()).isNotNull();
```

---

## Indicadores de bajo Mutation Score y sus causas

| Síntoma en reporte Pitest | Causa probable | Solución |
|---------------------------|----------------|----------|
| Mutante "removed call" sobrevive | `verify()` no tiene `times(N)` | Añadir `verify(mock, times(1)).metodo()` |
| Mutante "replaced return value" sobrevive | Solo se verifica `isNotNull()` | Añadir `isEqualTo(valorConcreto)` |
| Mutante "negated conditional" sobrevive | Falta test del caso negativo | Añadir test con condición falsa |
| Mutante "replaced boolean" sobrevive | No se verifica el booleano | `assertThat(result.isActivo()).isTrue()` |
| Mutante "replaced integer" sobrevive | Solo se verifica que el número existe | `assertThat(result.getOrden()).isEqualTo(5)` |
