---
name: 403-test-spring-boot
description: >
  Creates comprehensive unit tests for Spring Boot projects using JUnit 5 and Mockito.
  Ensures zero database interaction, follows AAA pattern, and generates JaCoCo coverage reports.
  Use when the user says "create tests", "write unit tests", "add tests for", "generate tests",
  "test coverage", or when reviewing untested code.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Unit Testing — Spring Boot + Gradle

## Proceso (seguir en orden)

**Paso 1 — Analizar la clase objetivo**
- Identifica el tipo: `@Service`, `@Component`, `@RestController`, `@Repository`, Mapper, POJO, etc.
- Lee las dependencias inyectadas (`@Autowired`, constructor, `@Value`)
- Detecta llamadas a BD, APIs externas u otro I/O que deba mockearse
- Lista métodos públicos y sus caminos: happy path, edge cases, excepciones
- Comprueba si ya existen tests para no duplicarlos

**Paso 2 — Verificar configuración Gradle**
Lee `references/gradle-jacoco.md` para la config de dependencias y JaCoCo.

**Paso 3 — Escribir los tests**
Selecciona la plantilla según el tipo de componente:

| Tipo | Plantilla |
|------|-----------|
| `@Service` / `@Component` | `references/plantilla-service.md` |
| `@RestController` sin seguridad | `references/plantilla-controller.md` |
| `@RestController` con `@PreAuthorize`, roles o `@Valid` | `references/plantilla-controller-security.md` |
| Clase con `@Value` | `references/plantilla-value.md` |
| Mapper MapStruct | `references/plantilla-mapper.md` |
| POJOs / DTOs / Entidades | `references/plantilla-pojo-meanbean.md` |
| ArgumentCaptor, parametrizados, Spy | `references/patrones-avanzados.md` |
| Mutation testing (Pitest) / MockUtils | `references/patrones-mutation-testing.md` |

**Paso 4 — Ejecutar y verificar cobertura**
```bash
./gradlew clean check                  # tests + reporte JaCoCo + verificación 80%
./gradlew jacocoTestReport             # solo reporte HTML
./gradlew mutation                     # cobertura de mutaciones con Pitest
```
Reportes:
- JaCoCo: `build/reports/jacoco/html/index.html`
- Pitest:  `build/reports/pitest/index.html`

---

## Reglas de oro (NO NEGOCIABLES)

### ❌ NUNCA en tests unitarios
```java
@SpringBootTest       // levanta contexto Spring completo — muy lento
@AutoConfigureMockMvc // requiere contexto Spring completo
@Transactional        // implica contexto Spring
```

### ✅ SIEMPRE en services y componentes
```java
@ExtendWith(MockitoExtension.class)  // Mockito puro, sin Spring
@Mock                                 // para cada dependencia
@InjectMocks                          // para la clase bajo test
ReflectionTestUtils.setField(...)     // para @Value sin Spring
```

### ✅ En controllers: elegir según lo que se necesite probar

```java
// Opción A: standaloneSetup — controller sin seguridad ni Bean Validation
MockMvcBuilders.standaloneSetup(controller).build()

// Opción B: @WebMvcTest — cuando el controller tiene @PreAuthorize, @Valid o roles
// Usar con inner TestConfig + @EnableMethodSecurity + @WithMockUser
@WebMvcTest
@ContextConfiguration(classes = MiControllerTest.TestConfig.class)
@WithMockUser
// Ver plantilla completa en: references/plantilla-controller-security.md
```

### Mockeo de repositorios (nunca BD real)
```java
@Mock private UserRepository userRepository;

when(userRepository.findById(anyLong())).thenReturn(Optional.of(testUser));
when(userRepository.save(any())).thenReturn(testUser);
doThrow(DataIntegrityViolationException.class).when(userRepository).save(any());
```

---

## Checklist de calidad

**Estructura general**
- [ ] No usa `@SpringBootTest`
- [ ] Todas las dependencias tienen `@Mock` / `@MockBean`
- [ ] La clase bajo test tiene `@InjectMocks` (o se construye manualmente)
- [ ] Al menos un test por método público (happy path + error)
- [ ] Mocks de repositorios con `when(...).thenReturn(...)`, nunca BD real
- [ ] Métodos con convención `metodo_Condicion_ResultadoEsperado`
- [ ] Cada test tiene `@DisplayName` descriptivo
- [ ] Interacciones verificadas con `verify(mock, times(n))`
- [ ] Assertions con `assertThat` de AssertJ
- [ ] `@BeforeEach` inicializa datos de prueba reutilizables
- [ ] Tests parametrizados para inputs múltiples donde aplique

**Controllers con seguridad**
- [ ] Usa `@WebMvcTest` + inner `TestConfig` si hay `@PreAuthorize` o `@Valid`
- [ ] POST/PUT/PATCH/DELETE usan `.with(csrf())`
- [ ] Hay test de 403 Forbidden cuando el usuario no tiene el rol requerido

**Mappers**
- [ ] Se instancia con `Mappers.getMapper(XxxMapper.class)`, no con `@Autowired`
- [ ] Se prueba null input, todos los campos, y actualización parcial si aplica

**POJOs / DTOs**
- [ ] Existe `TodosLosPojoTest` con `@TestFactory` + MeanBean para el paquete de dominio

**Mutation testing (Pitest)**
- [ ] Se usa `hasSize(N)` exacto en listas, no solo `isNotEmpty()`
- [ ] `ArgumentCaptor` verifica valores exactos pasados a los mocks
- [ ] Hay tests para el caso true y false de cada condición importante
- [ ] JaCoCo muestra ≥ 80% de cobertura de líneas
- [ ] Pitest muestra ≥ 80% de mutation score

---

## Convención de nombres

| Elemento       | Convención                   | Ejemplo                          |
|----------------|------------------------------|----------------------------------|
| Clase de test  | `{ClaseOriginal}Test`        | `UserServiceTest`                |
| Método de test | `metodo_Condicion_Resultado` | `findById_WhenNotFound_ShouldThrow` |
| Mock           | Mismo nombre que dependencia | `userRepository`, `emailService` |
| SUT            | Mismo nombre que la clase    | `userService`                    |
| Datos de prueba| `test` + tipo                | `testUser`, `testRequest`        |
