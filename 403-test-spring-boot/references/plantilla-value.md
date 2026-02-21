# Plantilla: Clases con @Value

Cuando la clase inyecta propiedades con `@Value`, usar `ReflectionTestUtils.setField` para inyectarlas manualmente sin levantar contexto Spring.

## Plantilla completa

```java
package com.ejemplo.service;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("EmailService - Unit Tests")
class EmailServiceTest {

    @Mock
    private SmtpClient smtpClient;

    @InjectMocks
    private EmailService emailService;

    @BeforeEach
    void setUp() {
        // Inyectar @Value manualmente — sin contexto Spring
        ReflectionTestUtils.setField(emailService, "senderEmail", "no-reply@ejemplo.com");
        ReflectionTestUtils.setField(emailService, "maxRetries", 3);
        ReflectionTestUtils.setField(emailService, "enabled", true);
    }

    @Test
    @DisplayName("sendWelcomeEmail - debe enviar email cuando el servicio está habilitado")
    void sendWelcomeEmail_WhenEnabled_ShouldSendEmail() {
        // Arrange
        doNothing().when(smtpClient).send(any());

        // Act
        emailService.sendWelcomeEmail("destino@ejemplo.com");

        // Assert
        verify(smtpClient, times(1)).send(any());
    }

    @Test
    @DisplayName("sendWelcomeEmail - no debe enviar email cuando el servicio está deshabilitado")
    void sendWelcomeEmail_WhenDisabled_ShouldNotSendEmail() {
        // Arrange — sobreescribir el valor inyectado en setUp
        ReflectionTestUtils.setField(emailService, "enabled", false);

        // Act
        emailService.sendWelcomeEmail("destino@ejemplo.com");

        // Assert
        verifyNoInteractions(smtpClient);
    }

    @Test
    @DisplayName("sendWelcomeEmail - debe reintentar hasta maxRetries en caso de fallo")
    void sendWelcomeEmail_WhenSmtpFails_ShouldRetry() {
        // Arrange
        doThrow(new SmtpException("Connection refused"))
                .when(smtpClient).send(any());

        // Act & Assert
        assertThatThrownBy(() -> emailService.sendWelcomeEmail("destino@ejemplo.com"))
                .isInstanceOf(EmailSendException.class);

        // Verifica que reintentó exactamente maxRetries (3) veces
        verify(smtpClient, times(3)).send(any());
    }
}
```

## Referencia de ReflectionTestUtils

```java
// Inyectar un valor en un campo privado
ReflectionTestUtils.setField(objeto, "nombreCampo", valor);

// Leer el valor de un campo privado (útil para assertions)
Object valor = ReflectionTestUtils.getField(objeto, "nombreCampo");

// Invocar un método privado
Object resultado = ReflectionTestUtils.invokeMethod(objeto, "nombreMetodo", arg1, arg2);
```
