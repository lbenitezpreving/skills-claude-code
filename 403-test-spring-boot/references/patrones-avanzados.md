# Patrones avanzados de testing

## 1. ArgumentCaptor — verificar datos exactos enviados al mock

Útil cuando necesitas comprobar qué objeto concreto se pasó a un mock, no solo que fue llamado.

```java
@Test
void create_ShouldSaveUserWithCorrectData() {
    // Arrange
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
    when(userRepository.save(captor.capture())).thenReturn(testUser);

    // Act
    service.create(new UserRequest("Juan", "juan@ejemplo.com"));

    // Assert — verificar los datos exactos persistidos
    User savedUser = captor.getValue();
    assertThat(savedUser.getName()).isEqualTo("Juan");
    assertThat(savedUser.getEmail()).isEqualTo("juan@ejemplo.com");
    assertThat(savedUser.isActive()).isTrue();
    assertThat(savedUser.getCreatedAt()).isNotNull();
}
```

## 2. Tests parametrizados — múltiples inputs con un solo test

```java
// Con @ValueSource para valores simples
@ParameterizedTest(name = "email={0} debe ser inválido")
@ValueSource(strings = {"noesvalido", "falta@", "@sindominio.com", "espacios @x.com"})
void validate_InvalidEmails_ShouldThrowException(String email) {
    assertThatThrownBy(() -> service.validateEmail(email))
            .isInstanceOf(InvalidEmailException.class);
}

// Con @NullAndEmptySource para null y ""
@ParameterizedTest
@NullAndEmptySource
void findByName_WhenBlank_ShouldThrowException(String name) {
    assertThatThrownBy(() -> service.findByName(name))
            .isInstanceOf(IllegalArgumentException.class);
}

// Con @CsvSource para múltiples parámetros
@ParameterizedTest(name = "rol={0}, activo={1} → puedeAcceder={2}")
@CsvSource({
    "ADMIN,  true,  true",
    "USER,   true,  true",
    "USER,   false, false",
    "GUEST,  true,  false"
})
void canAccess_ShouldReturnExpectedResult(String rol, boolean activo, boolean esperado) {
    assertThat(service.canAccess(rol, activo)).isEqualTo(esperado);
}
```

## 3. Verificar NO-interacciones

```java
// El mock nunca fue llamado
verifyNoInteractions(emailService);

// El mock no fue llamado con un método específico
verify(emailService, never()).sendWelcomeEmail(anyString());

// El mock fue llamado exactamente una vez y nada más
verify(userRepository, times(1)).save(any());
verifyNoMoreInteractions(userRepository);
```

## 4. Excepciones — aserciones completas

```java
// Opción preferida: assertThatThrownBy (más expresiva)
assertThatThrownBy(() -> service.metodo(param))
        .isInstanceOf(MiExcepcion.class)
        .hasMessageContaining("texto esperado")
        .hasCauseInstanceOf(SQLException.class);

// Opción alternativa: assertThrows + aserciones adicionales
MiExcepcion ex = assertThrows(MiExcepcion.class,
        () -> service.metodo(param));
assertThat(ex.getErrorCode()).isEqualTo("ERR_001");
```

## 5. @Spy — mock parcial (solo cuando es necesario)

Usar cuando quieres mockear solo algunos métodos de la clase, dejando el resto con implementación real.

```java
@Spy
private AuditService auditService;   // implementación real por defecto

@Test
void doSomething_ShouldAudit() {
    // Solo mockear el método que hace I/O
    doNothing().when(auditService).writeToFile(any());

    service.doSomething();

    // El resto de métodos de auditService se ejecutaron de verdad
    verify(auditService).writeToFile(any());
}
```

## 6. Ordenar tests con @TestMethodOrder (para flujos secuenciales)

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserWorkflowTest {

    @Test @Order(1)
    void create_ShouldPersistUser() { ... }

    @Test @Order(2)
    void activate_ShouldChangeStatus() { ... }

    @Test @Order(3)
    void delete_ShouldRemoveUser() { ... }
}
```

## 7. Aserciones avanzadas con AssertJ

```java
// Colecciones
assertThat(result)
        .hasSize(3)
        .extracting(User::getEmail)
        .containsExactlyInAnyOrder("a@b.com", "c@d.com", "e@f.com");

// Objetos complejos
assertThat(result)
        .usingRecursiveComparison()
        .ignoringFields("id", "createdAt")
        .isEqualTo(expectedUser);

// Excepciones con causa
assertThatThrownBy(() -> service.call())
        .isInstanceOf(ServiceException.class)
        .hasMessageContaining("timeout")
        .hasCauseInstanceOf(SocketTimeoutException.class);

// Soft assertions (reporta todos los fallos, no solo el primero)
SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(result.getName()).isEqualTo("Juan");
    softly.assertThat(result.getEmail()).isEqualTo("juan@ejemplo.com");
    softly.assertThat(result.isActive()).isTrue();
});
```
