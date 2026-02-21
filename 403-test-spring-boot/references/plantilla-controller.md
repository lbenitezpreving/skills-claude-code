# Plantilla: @RestController

Usar `standaloneSetup` para testear el controller de forma unitaria **sin levantar contexto Spring ni servidor**.

## Plantilla completa

```java
package com.ejemplo.controller;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;

import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("UserController - Unit Tests")
class UserControllerTest {

    @Mock
    private UserService userService;

    @InjectMocks
    private UserController userController;

    private MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders
                .standaloneSetup(userController)   // ← sin WebApplicationContext
                .build();
    }

    @Test
    @DisplayName("GET /api/users/{id} - debe retornar 200 con usuario")
    void getUser_WhenExists_ShouldReturn200() throws Exception {
        // Arrange
        UserResponse response = new UserResponse(1L, "Juan", "juan@ejemplo.com");
        when(userService.findById(1L)).thenReturn(response);

        // Act & Assert
        mockMvc.perform(get("/api/users/1")
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1L))
                .andExpect(jsonPath("$.email").value("juan@ejemplo.com"));

        verify(userService, times(1)).findById(1L);
    }

    @Test
    @DisplayName("GET /api/users/{id} - debe retornar 404 cuando no existe")
    void getUser_WhenNotFound_ShouldReturn404() throws Exception {
        // Arrange
        when(userService.findById(99L)).thenThrow(new UserNotFoundException(99L));

        // Act & Assert
        mockMvc.perform(get("/api/users/99"))
                .andExpect(status().isNotFound());
    }

    @Test
    @DisplayName("POST /api/users - debe retornar 201 con usuario creado")
    void createUser_WhenValidRequest_ShouldReturn201() throws Exception {
        // Arrange
        String requestBody = """
                {"name": "Juan", "email": "juan@ejemplo.com"}
                """;
        when(userService.create(any())).thenReturn(testUser);

        // Act & Assert
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(requestBody))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1L));
    }

    @Test
    @DisplayName("POST /api/users - debe retornar 400 con body inválido")
    void createUser_WhenInvalidRequest_ShouldReturn400() throws Exception {
        // Arrange
        String requestBody = """
                {"name": "", "email": ""}
                """;

        // Act & Assert
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(requestBody))
                .andExpect(status().isBadRequest());
    }
}
```

## Nota sobre ExceptionHandler

Si el controller tiene un `@ControllerAdvice` / `@ExceptionHandler`, registrarlo en el `standaloneSetup`:

```java
mockMvc = MockMvcBuilders
        .standaloneSetup(userController)
        .setControllerAdvice(new GlobalExceptionHandler())   // ← añadir aquí
        .build();
```
