# Plantilla: Mappers (MapStruct)

Los mappers MapStruct se instancian directamente con `Mappers.getMapper()` — sin Spring, sin Mockito. Son tests puramente unitarios que verifican la lógica de conversión entre entidades y DTOs.

## Plantilla completa

```java
package com.ejemplo.mapper;

import org.junit.jupiter.api.*;
import org.mapstruct.factory.Mappers;

import java.time.LocalDate;
import java.util.List;

import static org.assertj.core.api.Assertions.*;

@DisplayName("ProductoMapper - Unit Tests")
class ProductoMapperTest {

    private ProductoMapper sut;   // System Under Test

    @BeforeEach
    void setUp() {
        sut = Mappers.getMapper(ProductoMapper.class);
    }

    // ── toDto ──────────────────────────────────────────────────────────────

    @Test
    @DisplayName("toDto: mapea correctamente todos los campos")
    void toDto_todosLosCampos_mapeoCompleto() {
        Producto entidad = new Producto();
        entidad.setId(1L);
        entidad.setNombre("Producto A");
        entidad.setFecha(LocalDate.of(2024, 6, 15));
        entidad.setPrecio(new BigDecimal("19.99"));

        ProductoDTO result = sut.toDto(entidad);

        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getNombre()).isEqualTo("Producto A");
        assertThat(result.getFecha()).isEqualTo(LocalDate.of(2024, 6, 15));
        assertThat(result.getPrecio()).isEqualByComparingTo("19.99");
    }

    @Test
    @DisplayName("toDto: devuelve null cuando la entidad es null")
    void toDto_entidadNull_devuelveNull() {
        assertThat(sut.toDto((Producto) null)).isNull();
    }

    @Test
    @DisplayName("toDto: maneja correctamente campos opcionales nulos")
    void toDto_camposOpcionalesNulos_mapeoSinExcepcion() {
        Producto entidad = new Producto();
        entidad.setId(1L);
        entidad.setNombre("Sin fecha");
        // fecha y precio son null

        ProductoDTO result = sut.toDto(entidad);

        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getFecha()).isNull();
        assertThat(result.getPrecio()).isNull();
    }

    // ── toEntity ───────────────────────────────────────────────────────────

    @Test
    @DisplayName("toEntity: mapea correctamente DTO a entidad")
    void toEntity_todosLosCampos_mapeoCompleto() {
        ProductoDTO dto = new ProductoDTO();
        dto.setId(1L);
        dto.setNombre("Producto A");
        dto.setFecha(LocalDate.of(2024, 6, 15));

        Producto result = sut.toEntity(dto);

        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getNombre()).isEqualTo("Producto A");
    }

    @Test
    @DisplayName("toEntity: devuelve null cuando el DTO es null")
    void toEntity_dtoNull_devuelveNull() {
        assertThat(sut.toEntity((ProductoDTO) null)).isNull();
    }

    // ── updateFromDto (NullValuePropertyMappingStrategy.IGNORE) ───────────

    @Test
    @DisplayName("updateFromDto: actualiza solo los campos no nulos del DTO")
    void updateFromDto_camposNoNulos_sobreescribeCorrectamente() {
        Producto entidad = new Producto();
        entidad.setNombre("Original");
        entidad.setOrden(5);

        ProductoUpdateDTO updateDTO = new ProductoUpdateDTO();
        updateDTO.setNombre("Actualizado");
        updateDTO.setOrden(null);    // null → debe ignorarse (IGNORE strategy)

        sut.updateFromDto(updateDTO, entidad);

        assertThat(entidad.getNombre()).isEqualTo("Actualizado");
        assertThat(entidad.getOrden()).isEqualTo(5);    // sin cambios
    }

    @Test
    @DisplayName("updateFromDto: no modifica la entidad si todos los campos del DTO son null")
    void updateFromDto_todosNull_entidadSinCambios() {
        Producto entidad = new Producto();
        entidad.setNombre("Original");
        entidad.setOrden(3);

        sut.updateFromDto(new ProductoUpdateDTO(), entidad);

        assertThat(entidad.getNombre()).isEqualTo("Original");
        assertThat(entidad.getOrden()).isEqualTo(3);
    }

    // ── Listas ─────────────────────────────────────────────────────────────

    @Test
    @DisplayName("toDtoList: mapea todos los elementos de la lista")
    void toDtoList_listaConElementos_mapeaTodos() {
        List<Producto> entidades = List.of(
            crearProducto(1L, "A"),
            crearProducto(2L, "B"),
            crearProducto(3L, "C")
        );

        List<ProductoDTO> result = sut.toDtoList(entidades);

        assertThat(result).hasSize(3);
        assertThat(result).extracting(ProductoDTO::getNombre)
            .containsExactly("A", "B", "C");
    }

    @Test
    @DisplayName("toDtoList: devuelve lista vacía cuando la entrada es vacía")
    void toDtoList_listaVacia_devuelveListaVaciaNoNull() {
        List<ProductoDTO> result = sut.toDtoList(Collections.emptyList());

        assertThat(result).isNotNull().isEmpty();   // isNotNull mata mutante null-return
    }

    @Test
    @DisplayName("toDtoList: devuelve null cuando la entrada es null")
    void toDtoList_null_devuelveNull() {
        assertThat(sut.toDtoList(null)).isNull();
    }

    // ── Helpers ────────────────────────────────────────────────────────────

    private Producto crearProducto(Long id, String nombre) {
        Producto p = new Producto();
        p.setId(id);
        p.setNombre(nombre);
        return p;
    }
}
```

---

## Qué verificar en cada mapper

| Caso | Por qué |
|------|---------|
| Todos los campos mapeados | Detecta campos olvidados en la configuración del mapper |
| Entidad/DTO null → null | MapStruct lo genera automáticamente pero vale verificarlo |
| Campos opcionales null | Evita NullPointerException en transformaciones de campos anidados |
| `updateFromDto` con null | Verifica `NullValuePropertyMappingStrategy.IGNORE` |
| Lista vacía → lista vacía (no null) | Mata mutante de retorno null en colecciones |
| ID compuesto | Verifica extracción de campos desde objetos anidados |

---

## Mappers con dependencias de Spring (componentes)

Si el mapper usa `@Autowired` internamente (p.ej. otro mapper):

```java
// En lugar de Mappers.getMapper(), usar Spring en los tests de ese mapper
@ExtendWith(SpringExtension.class)
@Import({ProductoMapper.class, CategoriaMapper.class})  // mapper + sus dependencias
class ProductoMapperSpringTest {

    @Autowired
    private ProductoMapper sut;

    // mismos tests...
}
```
