# Plantilla: Tests de POJOs con MeanBean

MeanBean automatiza la verificación de getters, setters, `equals`, `hashCode` y builders en todas las clases de dominio. Con `@TestFactory` + `DynamicTest` se genera un test por cada POJO del paquete sin escribirlos manualmente.

## Dependencias

```groovy
testImplementation 'com.github.meanbeanlib:meanbean:3.0.0-M9'
```

---

## Plantilla: test dinámico para todo un paquete

```java
package com.ejemplo.domain;

import com.github.meanbeanlib.meanbean.BeanVerifier;
import org.junit.jupiter.api.*;

import java.util.stream.Stream;

@DisplayName("POJO Tests - Dominio")
class TodosLosPojoTest {

    private static final String BASE_PACKAGE = "com.ejemplo.domain";

    @TestFactory
    @DisplayName("Validación automática de getters/setters/equals/hashCode")
    Stream<DynamicTest> testTodosLosPojos() {
        return MeanBeanUtils.listarClasesTesteables(BASE_PACKAGE).stream()
            .map(clazz -> DynamicTest.dynamicTest(
                "POJO: " + clazz.getSimpleName(),
                () -> MeanBeanUtils.testBean(clazz)
            ));
    }
}
```

---

## Clase utilitaria MeanBeanUtils

Crear en `src/test/java/.../util/MeanBeanUtils.java`:

```java
package com.ejemplo.util;

import com.github.meanbeanlib.meanbean.BeanVerifier;
import io.github.classgraph.ClassGraph;
import io.github.classgraph.ScanResult;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.math.BigDecimal;
import java.sql.Timestamp;
import java.time.*;
import java.util.Arrays;
import java.util.List;

public class MeanBeanUtils {

    /**
     * Lista todas las clases del paquete que son instanciables (tienen constructor por defecto).
     */
    public static List<Class<?>> listarClasesTesteables(String paquete) {
        try (ScanResult scanResult = new ClassGraph()
                .enableClassInfo()
                .acceptPackages(paquete)
                .scan()) {
            return scanResult.getAllClasses()
                .loadClasses()
                .stream()
                .filter(MeanBeanUtils::tieneConstructorPorDefecto)
                .filter(c -> !c.isInterface() && !c.isEnum() && !c.isAnnotation())
                .toList();
        }
    }

    /**
     * Verifica getters/setters y equals/hashCode si están sobreescritos.
     */
    public static void testBean(Class<?> clazz) {
        BeanVerifier verifier = BeanVerifier.forClass(clazz)
            .withSettings(settings -> {
                settings.setFactoryCollection(MiFactoryRepository.getInstance());
                Arrays.stream(clazz.getDeclaredFields())
                    .filter(f -> !MiFactoryRepository.getInstance().hasFactory(f.getGenericType()))
                    .forEach(f -> settings.addIgnoredPropertyName(f.getName()));
            });

        verifier.verifyGettersAndSetters();

        if (tieneMetodoSobreescrito(clazz, "equals")) {
            verifier.verifyEqualsAndHashCode();
        }
    }

    private static boolean tieneConstructorPorDefecto(Class<?> clazz) {
        return Arrays.stream(clazz.getDeclaredConstructors())
            .anyMatch(c -> c.getParameterCount() == 0);
    }

    private static boolean tieneMetodoSobreescrito(Class<?> clazz, String nombre) {
        try {
            Method m = clazz.getMethod(nombre, Object.class);
            return m.getDeclaringClass() != Object.class;
        } catch (NoSuchMethodException e) {
            return false;
        }
    }
}
```

---

## Alternativa: test individual sin MeanBean

Para POJOs simples sin dependencias especiales:

```java
@Test
@DisplayName("ProductoDTO: getters y setters funcionan correctamente")
void productoDto_gettersSetters_funcionan() {
    ProductoDTO dto = new ProductoDTO();
    dto.setId(1L);
    dto.setNombre("Test");
    dto.setFecha(LocalDate.now());

    assertThat(dto.getId()).isEqualTo(1L);
    assertThat(dto.getNombre()).isEqualTo("Test");
    assertThat(dto.getFecha()).isNotNull();
}

@Test
@DisplayName("ProductoDTO: equals y hashCode son consistentes")
void productoDto_equalsHashCode_consistentes() {
    ProductoDTO dto1 = new ProductoDTO(1L, "Test", null);
    ProductoDTO dto2 = new ProductoDTO(1L, "Test", null);
    ProductoDTO dto3 = new ProductoDTO(2L, "Otro", null);

    assertThat(dto1).isEqualTo(dto2);
    assertThat(dto1).hasSameHashCodeAs(dto2);
    assertThat(dto1).isNotEqualTo(dto3);
}
```

---

## Qué cubre MeanBean automáticamente

| Verificación | Descripción |
|--------------|-------------|
| `verifyGettersAndSetters()` | Que `setX(v)` + `getX()` devuelven el mismo valor para cada campo |
| `verifyEqualsAndHashCode()` | Que dos objetos iguales tienen el mismo hashCode y equals devuelve true |
| Builder pattern | Si la clase usa Lombok `@Builder`, también se verifica |

> **Nota:** `@TestFactory` genera un test dinámico por clase en el reporte JUnit, lo que hace fácil identificar qué POJO falla específicamente.
