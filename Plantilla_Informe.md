# Informe de Laboratorio: Estructura de Computadores

**Nombre del Estudiante:** Ricardo Urueta 
**Fecha:** 02/03/2026  
**Asignatura:** Estructura de Computadores
 
**Enlace del repositorio en GitHub:** https://github.com/Rjuc/Estructura-de-computadores# 
 

---

## 1. Análisis del Código Base

### 1.1. Evidencia de Ejecución
Adjunte aquí las capturas de pantalla de la ejecución del `programa_base.asm` utilizando las siguientes herramientas de MARS:
*   **MIPS X-Ray** (Ventana con el Datapath animado).
> ![MIPS X-Ray Base](MIP-X-Ray-Programa-base.png)
*   **Instruction Counter** (Contador de instrucciones totales).
![Instruction Counter Base](Instruction-Counter-Programa-base.png)
*   **Instruction Statistics** (Desglose por tipo de instrucción).
![Instruction Statistics Base](Instruction-Statistics-Programa-base.png)


### 1.2. Identificación de Riesgos (Hazards)
Completa la siguiente tabla identificando las instrucciones que causan paradas en el pipeline:

| Instrucción Causante | Instrucción Afectada | Tipo de Riesgo (Load-Use, etc.) | Ciclos de Parada |
|----------------------|----------------------|---------------------------------|------------------|
| `lw $t6, 0($t5)`     | `mul $t7, $t6, $t0`  | Load-Use                        |       2          |
|  `mul $t7, $t6, $t0` | `addu $t8, $t7, $t1` | RAW (mul)                       |        1         |

### 1.2. Estadísticas y Análisis Teórico
Dado que MARS es un simulador funcional, el número de instrucciones ejecutadas será igual en ambas versiones. Sin embargo, en un procesador real, el tiempo de ejecución (ciclos) varía. Completa la siguiente tabla de análisis teórico:

| Métrica | Código Base | Código Optimizado |
|---------|-------------|-------------------|
| Instrucciones Totales (según MARS) |      94       |         94          |
| Stalls (Paradas) por iteración |       3      |          1         |
| Total de Stalls (8 iteraciones) |       24      |        8           |
| **Ciclos Totales Estimados** (Inst + Stalls) |     118        |         102          |
| **CPI Estimado** (Ciclos / Inst) |       1.26      |          1.09         |

---

## 2. Optimización Propuesta

### 2.1. Evidencia de Ejecución (Código Optimizado)
Adjunte aquí las capturas de pantalla de la ejecución del `programa_optimizado.asm` utilizando las mismas herramientas que en el punto 1.1:
*   **MIPS X-Ray** (Ventana con el Datapath animado).
> ![MIPS X-Ray Optimizado](MIP-X-Ray-Programa-optimizado.png)
*   **Instruction Counter** (Contador de instrucciones totales).
![Instruction Counter Optimizado](Instruction-Counter-Programa-optimizado.png)
*   **Instruction Statistics** (Desglose por tipo de instrucción).
![Instruction Statistics Optimizado](Instruction-Statistics-Programa-optimizado.png)

**Nota:** Como se puede observar, los contadores muestran los mismos valores que en el código base (94 instrucciones totales). Esto es  porque MARS es un simulador funcional que solo cuenta instrucciones ejecutadas, no ciclos de CPU. La mejora en rendimiento (reducción de stalls) solo sería visible en un procesador real con pipeline o en un simulador de ciclos.

### 2.2. Código Optimizado
Pega aquí el fragmento de tu bucle `loop` reordenado:

```asm
loop:
    # --- Condición de salida ---
    beq $t3, $t2, fin     # Si i == tamano, salir del bucle
    
    # --- Cálculo de dirección de memoria ---
    sll $t4, $t3, 2       # Desplazamiento: t4 = i * 4
    addu $t5, $s0, $t4    # t5 = dirección de X[i]
    
    # --- Carga de dato ---
    lw $t6, 0($t5)        # Leer X[i]
    
    # -- Aquí es donde se ve realmente la reordenación para "rellenar" los ciclos de espera --
    addi $t3, $t3, 1      # i = i + 1 (MOVIDO desde el final)
    addu $t9, $s1, $t4    # t9 = dirección de Y[i] (MOVIDO desde abajo)
    
    # --- Operación aritmética ---
    # Ahora $t6 ya está disponible (sin stall de load-use)
    mul $t7, $t6, $t0     # t7 = X[i] * A
    addu $t8, $t7, $t1    # t8 = t7 + B
    
    # --- Almacenamiento de resultado ---
    sw $t8, 0($t9)        # Guardar resultado en Y[i]
    
    # --- Salto ---
    j loop
```

### 2.2. Justificación Técnica de la Mejora
Explica qué instrucción moviste y por qué colocarla entre el `lw` y el `mul` elimina el riesgo de datos:

> Moví dos instrucciones que no dependen del valor cargado por `lw $t6, 0($t5)`: el incremento del índice (`addi $t3, $t3, 1`) y el cálculo de la dirección de Y[i] (`addu $t9, $s1, $t4`). Al colocarlas inmediatamente después del `lw` y antes del `mul`, estas instrucciones "llenan" los ciclos de espera que normalmente causaría el riesgo load-use. Cuando llega el momento de ejecutar `mul $t7, $t6, $t0`, el valor de `$t6` ya está disponible porque las dos instrucciones intermedias le dieron tiempo suficiente para completarse. Esto elimina los 2 momentos en los que el procesador "estaba quieto" que existían en el código base.

---

## 3. Comparativa de Resultados

| Métrica | Código Base | Código Optimizado | Mejora (%) |
|---------|-------------|-------------------|------------|
| Ciclos Totales |     118        |         102          |   13.6%         |
| Stalls (Paradas) |       24      |         8          |     66.7%       |
| CPI |     1.26        |        1.09           |        13.05    |

---

## 4. Conclusiones
¿Qué impacto tiene la segmentación en el diseño de software de bajo nivel? ¿Es siempre posible eliminar todas las paradas?

### El Impacto de la Arquitectura en el Desarrollo de Software

Este laboratorio demuestra que la arquitectura del procesador hace que las buenas prácticas de programación tomen mucha más relevancia en lenguaje ensamblador. No se trata solo de la estética del código o de seguir convenciones por tradición, sino del uso de recursos reales que puede verse afectado notablemente por algo tan sencillo como el orden de las instrucciones. Un simple reordenamiento redujo el tiempo de ejecución en un 13.6%.

### Limitaciones de la Optimización

No logramos eliminar todas las paradas porque, aunque movimos instrucciones independientes para cubrir el stall del load-use, no teníamos más instrucciones disponibles para cubrir el tiempo de ejecución de la multiplicación. En clase mencionaron que estos stalls son tiempo en el que el procesador literalmente no hace nada útil, solo espera. Aunque estamos en nuestros primeros pasos con assembler, ya podemos ver cómo esos conceptos teóricos se vuelven muy reales: cada ciclo perdido es un ciclo donde el hardware está inactivo.

### Balance entre Complejidad y Rendimiento

La segmentación definitivamente vale la pena, a pesar de introducir riesgos de datos. En programación, los trade-offs siempre existen: mayor costo puede implicar menor tiempo de espera, mayor trabajo puede significar mayor seguridad, y así sucesivamente. En este caso, nos enfrentamos a un trade-off donde mayor atención a los detalles del ordenamiento de instrucciones nos ofrece mayor eficiencia en el uso del procesador.

### Aplicación Práctica

En entornos profesionales, a veces será necesario dejar pasar por alto este tipo de optimizaciones para trabajos poco repetitivos y con bajo impacto. Sin embargo, cuando más usuarios utilizan un sistema, o cuando es un sistema del que dependen otros procesos críticos y/o se usa muy frecuentemente, hay que tener estas consideraciones muy en cuenta al escribir código. En estos casos, incluso sería recomendable pensar en procesos de QA que hagan un double-check para asegurarnos de que estamos sacando el máximo provecho de los recursos disponibles.
