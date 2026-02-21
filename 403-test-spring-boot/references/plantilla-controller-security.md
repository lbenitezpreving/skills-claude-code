# Plantilla: @RestController con Seguridad (@WebMvcTest)

Usar `@WebMvcTest` cuando el controller tiene **cualquiera** de estos elementos:
- `@PreAuthorize` / `@Secured` (seguridad a nivel de método)
- `@Valid` / `@Validated` (Bean Validation en parámetros o body)
- Lógica que depende del rol del usuario autenticado
- Necesidad de probar códigos HTTP 403 Forbidden

Para controllers **sin** seguridad ni validaciones usa `standaloneSetup` (ver `plantilla-controller.md`).

---

## Plantilla completa

```java
package com.ejemplo.controller;

import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.context.annotation.*;
import org.springframework.http.MediaType;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.web.servlet.MockMvc;
import com.fasterxml.jackson.databind.ObjectMapper;

import static org.mockito.Mockito.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest
@ContextConfiguration(classes = ProductoControllerTest.TestConfig.class)
@WithMockUser   // usuario autenticado por defecto en todos los tests
class ProductoControllerTest {

    // ── Configuración interna ────────────────────────────────────────────────
    // Inner class estática: carga SOLO el controller bajo test + seguridad
    @Configuration
    @EnableMethodSecurity                    // habilita @PreAuthorize en el controller
    @Import(ProductoController.class)        // importar solo el controller bajo test
    static class TestConfig { }

    // ── Infraestructura de test ──────────────────────────────────────────────
    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    // ── Mocks de dependencias ────────────────────────────────────────────────
    @MockBean
    private ProductoService productoService;

    // Si el controller usa un servicio de seguridad propio (ej: comprueba rol)
    // usar name= con el nombre exacto del bean Spring
    @MockBean(name = "securityService")
    private SecurityService securityService;

    @BeforeEach
    void setUp() {
        // Por defecto, el usuario tiene el rol necesario
        when(securityService.tieneRol()).thenReturn(true);
    }

    // ── Tests GET ────────────────────────────────────────────────────────────

    @Test
    @DisplayName("GET /api/productos - 200 OK con lista de resultados")
    void listar_conResultados_deberiaDevolver200() throws Exception {
        List<ProductoDTO> mockDTOs = List.of(
            new ProductoDTO(1L, "Producto A"),
            new ProductoDTO(2L, "Producto B")
        );
        when(productoService.listar()).thenReturn(mockDTOs);

        mockMvc.perform(get("/api/productos")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$", hasSize(2)))
            .andExpect(jsonPath("$[0].id").value(1L))
            .andExpect(jsonPath("$[0].nombre").value("Producto A"));
    }

    @Test
    @DisplayName("GET /api/productos - 204 No Content cuando no hay resultados")
    void listar_sinResultados_deberiaDevolver204() throws Exception {
        when(productoService.listar()).thenReturn(Collections.emptyList());

        mockMvc.perform(get("/api/productos"))
            .andExpect(status().isNoContent());
    }

    @Test
    @DisplayName("GET /api/productos - 403 Forbidden cuando usuario no tiene rol")
    void listar_sinRol_deberiaDevolver403() throws Exception {
        when(securityService.tieneRol()).thenReturn(false);

        mockMvc.perform(get("/api/productos"))
            .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("GET /api/productos - 400 Bad Request por parámetro inválido")
    void listar_parametroInvalido_deberiaDevolver400() throws Exception {
        mockMvc.perform(get("/api/productos")
                .param("id", "-1"))        // @Min(1) en el controller
            .andExpect(status().isBadRequest());
    }

    @Test
    @DisplayName("GET /api/productos/{id} - 404 Not Found cuando no existe")
    void obtenerPorId_noExiste_deberiaDevolver404() throws Exception {
        doThrow(new NotFoundRestApiException("Producto no encontrado"))
            .when(productoService).obtenerPorId(99L);

        mockMvc.perform(get("/api/productos/99"))
            .andExpect(status().isNotFound());
    }

    // ── Tests POST/PUT/PATCH/DELETE (requieren .with(csrf())) ────────────────

    @Test
    @DisplayName("POST /api/productos - 201 Created con datos válidos")
    void crear_datosValidos_deberiaDevolver201() throws Exception {
        ProductoCreateDTO createDTO = new ProductoCreateDTO("Nuevo Producto");
        ProductoDTO responseDTO = new ProductoDTO(1L, "Nuevo Producto");
        when(productoService.crear(any(ProductoCreateDTO.class))).thenReturn(responseDTO);

        mockMvc.perform(post("/api/productos")
                .with(csrf())                                           // ← OBLIGATORIO en POST
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(createDTO)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1L));
    }

    @Test
    @DisplayName("POST /api/productos - 409 Conflict cuando ya existe")
    void crear_yaExiste_deberiaDevolver409() throws Exception {
        ProductoCreateDTO createDTO = new ProductoCreateDTO("Duplicado");
        doThrow(new ConflictRestApiException("Ya existe un producto con ese nombre"))
            .when(productoService).crear(any());

        mockMvc.perform(post("/api/productos")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(createDTO)))
            .andExpect(status().isConflict());
    }

    @Test
    @DisplayName("PATCH /api/productos/{id} - 403 Forbidden sin rol de administrador")
    void actualizar_sinRolAdmin_deberiaDevolver403() throws Exception {
        when(securityService.tieneRolAdmin()).thenReturn(false);

        mockMvc.perform(patch("/api/productos/1")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("DELETE /api/productos/{id} - 204 No Content al eliminar correctamente")
    void eliminar_existente_deberiaDevolver204() throws Exception {
        doNothing().when(productoService).eliminar(1L);

        mockMvc.perform(delete("/api/productos/1")
                .with(csrf()))
            .andExpect(status().isNoContent());
    }

    // ── Test con ArgumentCaptor (matar mutantes en filtrado de parámetros) ───

    @Test
    @DisplayName("POST /api/productos/importar - filtra parámetros reservados antes de enviar al servicio")
    void importar_deberiaFiltrarParametrosReservados() throws Exception {
        Map<String, Object> params = new HashMap<>();
        params.put("file", "ignorar");        // debe filtrarse
        params.put("origenId", "1");          // debe filtrarse
        params.put("customParam", "valor");   // debe pasar

        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);
        when(productoService.importar(any(), anyMap())).thenReturn(ResponseEntity.ok("OK"));

        productoController.importar(mockFile, 1, params);

        verify(productoService).importar(any(), captor.capture());
        assertThat(captor.getValue())
            .doesNotContainKeys("file", "origenId")
            .containsKey("customParam")
            .hasSize(1);   // exacto para matar mutantes de tamaño
    }
}
```

---

## EasyRandom para datos de prueba en controllers

Cuando el servicio devuelve colecciones y no importan los valores concretos:

```java
private EasyRandom easyRandom;

@BeforeEach
void setUp() {
    easyRandom = new EasyRandom();
    when(securityService.tieneRol()).thenReturn(true);
}

@Test
void listar_conResultados_deberiaDevolver200() throws Exception {
    List<ProductoDTO> mockDTOs = easyRandom.objects(ProductoDTO.class, 3)
        .collect(Collectors.toList());
    when(productoService.listar()).thenReturn(mockDTOs);

    mockMvc.perform(get("/api/productos"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$", hasSize(3)));
}
```

Dependencia:
```groovy
testImplementation 'io.github.classgraph:classgraph:4.8.149'
testImplementation 'org.jeasy:easy-random-core:5.0.0'
```

---

## Tests con paginación (`Page<T>`)

```java
@Test
void listar_conPaginacion_deberiaDevolver200() throws Exception {
    List<ProductoDTO> content = List.of(new ProductoDTO(1L, "A"), new ProductoDTO(2L, "B"));
    Page<ProductoDTO> page = new PageImpl<>(content, PageRequest.of(0, 10), 2);
    when(productoService.listar(any(Pageable.class))).thenReturn(page);

    mockMvc.perform(get("/api/productos")
            .param("page", "0")
            .param("size", "10"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.content", hasSize(2)))
        .andExpect(jsonPath("$.totalElements").value(2));
}
```

---

## Cuándo usar cada enfoque

| Situación | Enfoque |
|-----------|---------|
| Controller con `@PreAuthorize` / roles | `@WebMvcTest` + inner `TestConfig` + `@WithMockUser` |
| Controller con `@Valid` en body/params | `@WebMvcTest` (Bean Validation solo funciona con contexto web) |
| Controller sin seguridad ni validaciones | `standaloneSetup` (ver `plantilla-controller.md`) |
| Solo verificar lógica interna del controller | Mocks directos + `@InjectMocks` |
