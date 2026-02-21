# Plantilla: @Service / @Component

## Ubicación del archivo

```
src/main/java/com/ejemplo/service/UserService.java
src/test/java/com/ejemplo/service/UserServiceTest.java   ← espejo exacto
```

## Plantilla completa

```java
package com.ejemplo.service;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.NullAndEmptySource;
import org.junit.jupiter.params.provider.ValueSource;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)      // ← NUNCA @SpringBootTest
@DisplayName("UserService - Unit Tests")
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;

    private User testUser;

    @BeforeEach
    void setUp() {
        testUser = User.builder()
                .id(1L)
                .name("Juan García")
                .email("juan@ejemplo.com")
                .active(true)
                .build();
    }

    // ── Happy path ───────────────────────────────

    @Test
    @DisplayName("findById - debe retornar usuario cuando existe")
    void findById_WhenUserExists_ShouldReturnUser() {
        // Arrange
        when(userRepository.findById(1L)).thenReturn(Optional.of(testUser));

        // Act
        User result = userService.findById(1L);

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getEmail()).isEqualTo("juan@ejemplo.com");
        verify(userRepository, times(1)).findById(1L);
    }

    // ── Excepciones ──────────────────────────────

    @Test
    @DisplayName("findById - debe lanzar NotFoundException cuando no existe")
    void findById_WhenUserNotFound_ShouldThrowNotFoundException() {
        // Arrange
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        // Act & Assert
        assertThatThrownBy(() -> userService.findById(99L))
                .isInstanceOf(UserNotFoundException.class)
                .hasMessageContaining("99");

        verify(userRepository, times(1)).findById(99L);
    }

    @ParameterizedTest
    @NullAndEmptySource
    @DisplayName("create - debe lanzar excepción si el email es nulo o vacío")
    void create_WhenEmailIsNullOrEmpty_ShouldThrowIllegalArgumentException(String email) {
        // Arrange
        UserRequest request = new UserRequest(null, email);

        // Act & Assert
        assertThatThrownBy(() -> userService.create(request))
                .isInstanceOf(IllegalArgumentException.class);

        verifyNoInteractions(userRepository);
    }

    // ── Verificación de interacciones ────────────

    @Test
    @DisplayName("create - debe guardar usuario y enviar email de bienvenida")
    void create_WhenValidRequest_ShouldSaveAndSendWelcomeEmail() {
        // Arrange
        UserRequest request = new UserRequest("Juan", "juan@ejemplo.com");
        when(userRepository.save(any(User.class))).thenReturn(testUser);

        // Act
        User result = userService.create(request);

        // Assert
        assertThat(result).isNotNull();
        verify(userRepository, times(1)).save(any(User.class));
        verify(emailService, times(1)).sendWelcomeEmail("juan@ejemplo.com");
        verifyNoMoreInteractions(userRepository, emailService);
    }
}
```
