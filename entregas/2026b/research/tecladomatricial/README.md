
# Tema: Interfaz de teclado matricial y antirrebote (debounce) por software

**Alumno:** Axel José Mejía Salinas

**Número de control:** 24210504

## 1. Introducción

Los teclados son una excelente manera de permitir que los usuarios interactúen con sus proyectos. Se pueden usar para navegar por los menús, introducir contraseñas o controlar los juegos y robots.

Un teclado matricial permite la conexión de múltiples botones utilizando un menor número de pines en un sistema. Debido a la naturaleza mecánica de los interruptores, se requiere procesar las señales para evitar lecturas erróneas. Ya sea que el sistema se base en microcontroladores de 8 bits clásicos o en arquitecturas modernas de 32 y 64 bits (como ARM y RISC-V), los principios físicos de la matriz se mantienen, aunque la forma de interactuar con el hardware a bajo nivel cambia significativamente.

## 2. Principio de funcionamiento del Teclado Matricial

Las _teclas_ de un teclado están organizadas en filas y columnas. Existen múltiples teclados con diferente número de teclas, siendo los más habituales las configuraciones de **3×3**, **3×4** y **4×4**.

Este tipo de teclados están constituidos por 3 membranas superpuestas, dos membranas con material conductor y una en medio no conductora, para separarlas. En condiciones normales, el interruptor se encuentra abierto, pero al presionar la tecla, la membrana superior e inferior entran en contacto permitiendo la circulación de la corriente.

Los pulsadores están distribuidos en _filas_ y _columnas_. Para detectar la pulsación de una tecla tendremos que conocer la posición **(X, Y)**. Por ejemplo, la tecla del número 5 corresponde a la fila 2 y columna 2, por lo que se encuentra en la posición **(2,2)**.
![Teclado matricial de 4x4 - Guia de trabajo para micro:bit](https://fgcoca.github.io/Guia-de-trabajo-para-microbit/img/conceptos/teclado/esquema.png)

### 2.1 Algoritmo de detección e identificación a bajo nivel

El algoritmo de escaneo consiste en un **ciclo iterativo**. El programa de control se divide en dos partes: una subrutina de "detección" (que detecta que se oprimió una tecla) y una subrutina de "identificación" (que determina cuál fue).

El proceso lógico estándar es el siguiente:

-   Se programa un puerto (o varios), asignando la mitad de las señales como salidas y la otra mitad como entradas.
    
-   La técnica consiste en escribir en los bits del puerto en forma secuencial un "CERO" lógico en las columnas y leer cada vez el estado de los renglones.
    
-   Cuando una tecla es oprimida, la lectura en alguno de los renglones será también un "CERO".
    
-   El código obtenido de esta lectura se convierte en el código ASCII de la tecla oprimida mediante el uso de una tabla de equivalencias.
    

#### 2.1.1 Ejemplo práctico de escaneo (Enfoque tradicional de 8 bits)

Tomando como referencia un diagrama típico de conexión hacia el **Puerto B** de un microcontrolador (por ejemplo, un PIC 16F88), los 8 pines se distribuyen de la siguiente manera:

-   **Salidas (Columnas Y):** El pin `RB0` se conecta a `Y1`, `RB1` a `Y2`, `RB2` a `Y3` y `RB3` a `Y4`.
    

**Escenario: El usuario presiona la tecla "0" (Intersección X1, Y1):**

> 1.  El microcontrolador, en su ciclo de escaneo, envía un `0` lógico a la primera columna (`Y1`) y un `1` al resto de las columnas. Por lo tanto, el estado de los 4 bits más bajos del puerto será `1110`.
>     
> 2.  El sistema procede a leer los 4 bits más altos correspondientes a los renglones. Debido a la configuración de resistencias _pull-up_, los renglones normalmente leen un `1`.
>     
> 3.  Como la tecla **"0"** está presionada, el circuito se cierra. El `0` lógico fluye hacia el renglón `X1`, causando que un pin de entrada lea un `0`.
>     
> 4.  Al concatenar la lectura completa del puerto, el microcontrolador obtiene un byte específico (ej. **0xEE**).
>     
> 5.  El software busca este valor `0xEE` en su tabla de traducción (_Look-up Table_) y lo identifica exitosamente.
>     

#### 2.1.2 Implementación del escaneo en código ensamblador (Arquitecturas PIC - 8 bits)

En microcontroladores tradicionales, el acceso a los puertos se realiza mediante instrucciones directas a registros de función especial.

Fragmento de código

```
; Fragmento de barrido en PIC
vuelta1:
    movlw   H'EF'        ; Coloca un 0 lógico en la primera columna
    movwf   PORTB        ; Escribe el patrón de salida
    call    delay        ; Retardo breve
    movfw   PORTB        ; Lee el estado actual del puerto B (Renglones)
    iorlw   H'0F'        ; Máscara OR
    ; ... [Lógica de salto según bandera Z] ...

```

#### 2.1.3 Implementación en arquitecturas modernas (ARM32/64 y RISC-V)
![What is RV32I and How Does it Work ?](https://icstutorial.com/wp-content/uploads/2026/04/RV32IRisc-VCPU15-IPCORE.jpg)
A diferencia de los microcontroladores de 8 bits, las arquitecturas modernas como **ARM (Cortex-M o Cortex-A)** y **RISC-V** utilizan I/O Mapeado en Memoria (_Memory-Mapped I/O_). Esto significa que los periféricos GPIO no tienen instrucciones especiales (como `movwf` hacia un puerto de hardware), sino que se manejan leyendo y escribiendo en direcciones de memoria específicas usando instrucciones estándar de carga y almacenamiento (`LDR`/`STR` en ARM, `lw`/`sw` en RISC-V).

Además, en procesadores de 32/64 bits, los puertos GPIO suelen estar divididos en múltiples registros especializados: uno para establecer el estado de salida (ej. _Output Data Register_ o _BSRR_), otro para leer el estado de entrada (_Input Data Register_), y otros para configurar los modos de _Pull-up/Pull-down_, lo que requiere cargar punteros de memoria base antes de operar.

**Ejemplo conceptual en ARM (Thumb-2 para Cortex-M - 32 bits):**

Fragmento de código

```
    ; Supongamos que las columnas están en GPIOA y las filas en GPIOB
    LDR R0, =GPIOA_BASE      ; Carga la dirección base del puerto de salidas
    LDR R1, =GPIOB_BASE      ; Carga la dirección base del puerto de entradas
    
    ; 1. Activar la columna 1 (escribiendo un patrón en ODR o BSRR)
    MOV R2, #0xFFFE          ; Patrón para poner el bit 0 en bajo y el resto en alto
    STR R2, [R0, #0x14]      ; Almacena en el registro de salida (Offset de ODR)
    
    ; 2. Pequeño retardo (usualmente con NOPs o un timer de hardware)
    NOP
    NOP
    
    ; 3. Leer las filas
    LDR R3, [R1, #0x10]      ; Carga el valor del Input Data Register (IDR) en R3
    AND R3, R3, #0x0F        ; Aplica máscara para aislar los bits de las filas
    CMP R3, #0x0F            ; Compara si todos están en 1 (ninguna tecla presionada)
    BNE teclazo              ; Si no son iguales, salta a la subrutina de identificación

```

**Ejemplo conceptual en RISC-V (RV32I):**

Fragmento de código

```
    # RISC-V utiliza registros base y offsets inmediatos para acceder a memoria
    li t0, 0x40010800        # Carga la dirección base de GPIOA (Ejemplo genérico)
    li t1, 0x40010C00        # Carga la dirección base de GPIOB
    
    # 1. Escribir a columnas
    li t2, 0xFFFE            # Patrón para activar columna 1 (0 lógico)
    sw t2, 0x14(t0)          # Store Word en el offset de salida de GPIOA
    
    # 2. Leer filas
    lw t3, 0x10(t1)          # Load Word desde el offset de entrada de GPIOB
    andi t3, t3, 0x0F        # Máscara para obtener solo los bits de fila
    li t4, 0x0F
    bne t3, t4, teclazo      # Branch if Not Equal: si t3 != 0x0F, hay tecla presionada

```

_Nota: En ARM64 (AArch64), la filosofía es la misma, pero se utilizan registros de 64 bits (`X0`, `X1`) y un mapa de direcciones de memoria más amplio, típicamente corriendo sobre un sistema operativo que gestiona los GPIO a través del kernel, en lugar de acceso directo desde nivel de usuario._

### 2.2 Requisitos de hardware

_Para realizar la lectura de un teclado matricial e implementar la técnica de antirrebote (debounce) por software, el programa o firmware debe gestionar los siguientes elementos independientemente de la arquitectura:_

**Configuración de puertos de E/S:** Asignar un grupo de pines como salidas digitales y otro como entradas. En ARM y RISC-V esto implica configurar registros adicionales de habilitación de reloj del bus (ej. AHB/APB en microcontroladores STM32).

**Habilitación de resistencias de pulso:** Activar las resistencias internas de _pull-up_ o _pull-down_.

**Manejo de temporización:** En sistemas avanzados (ARM/RISC-V), en lugar de ciclos de espera por software (delay), es estándar utilizar interrupciones por _SysTick Timer_ o temporizadores de hardware periférico para no bloquear el procesador principal.

**Operaciones a nivel de bits (Bitwise operations):** Emplear máscaras lógicas.

**Tabla de traducción (Look-up Table):** Matriz bidimensional en código.

## 3. El fenómeno del rebote

Los contactos metálicos de un pulsador no cierran de forma instantánea. Generan múltiples transiciones rápidas antes de estabilizarse mecánicamente. Para evitar lecturas erróneas (como registrar 5 pulsaciones cuando el usuario solo tocó la tecla una vez), se emplean técnicas de eliminación de rebotes mediante hardware (capacitores/filtros RC) o software.

## 4. Debounce (Antirrebote)

Un debounce _(o desparasitado)_ es una técnica que elimina las lecturas falsas causadas por el rebote mecánico.

### 4.1 Método por retardo (Delay)

Consiste en detectar un cambio de estado, aplicar una pausa temporal (ej. 10 a 50 ms) y realizar una segunda verificación.

-   **Consecutive Runs:** Espera a que el rebote desaparezca contando lecturas estables. Introduce retrasos.
    
-   **Quick Draw:** Valida el primer cambio e ignora las lecturas siguientes durante un tiempo. Minimiza el retraso pero es susceptible a ruido electromagnético.
    

### 4.2 Método por máquina de estados

Evita detener el flujo del programa principal. Este método es especialmente relevante en procesadores ARM y RISC-V de alto rendimiento, donde bloquear la CPU con funciones `delay()` desperdicia miles o millones de ciclos de reloj. En su lugar, la máquina de estados se actualiza periódicamente dentro de la interrupción de un _Timer_.

El sistema opera bajo cuatro estados discretos:

-   **Estado 0:** Estado estacionario bajo.
    
-   **Estado 1:** Transición de bajo a alto.
    
-   **Estado 2:** Estado estacionario alto.
    
-   **Estado 3:** Transición de alto a bajo.
    

El siguiente algoritmo en C/C++ es altamente portable y puede ser compilado eficientemente tanto por GCC como por LLVM/Clang para ejecutarse en arquitecturas AVR, ARM32, ARM64 o RISC-V sin modificación, ya que depende enteramente del paso paramétrico del tiempo y lógica condicional:
C++
```
// Algoritmo de debounce basado en integrador y máquina de estados
bool update(bool input) 
{
    // Integrador discreto para evaluar estabilidad
    if (input) 
    { 
        if (counter < +thres_steady) 
            ++counter; 
    } 
    else 
    { 
        if (counter > -thres_steady) 
            --counter;
    }

    // Máquina de estados
    switch (state) 
    { 
        case 0: // steady-state lo
            if (counter >= -thres_transient_abs) 
            { 
                // => transient lo-hi 
                counter = 0; 
                state = 1; 
                return true; 
            } 
            else
            { 
                return false;
            } 

        case 1: // transient lo-hi 
            switch (counter)
            { 
                case +thres_steady: 
                    // => steady-state hi 
                    state = 2;
                    return false;

                case -thres_steady:
                    // => steady-state lo 
                    state = 0;
                    return true;

                default: 
                    return false;
            }
            // ... [casos 2 y 3 omitidos para brevedad] ... 
    }
}

```

## 5. Conclusiones

La implementación de una interfaz de teclado matricial optimiza de forma sustancial el uso de pines en cualquier sistema digital. Si bien los conceptos originales nacieron con microcontroladores de 8 bits, las arquitecturas modernas de 32 y 64 bits (ARM y RISC-V) requieren adaptar estas técnicas al paradigma de memoria mapeada y a una gestión más estricta de los relojes y registros del sistema.

El uso de algoritmos de antirrebote por software demuestra ser una solución eficiente al sustituir filtros de hardware. La correcta integración entre el escaneo matricial y un algoritmo de filtrado no bloqueante (como máquinas de estados operadas mediante interrupciones de Timer) es fundamental para desarrollar sistemas embebidos modernos, garantizando operaciones en tiempo real, robustas y de alta eficiencia sin importar la arquitectura del procesador.

## 6. Referencias

-   El octavo, B. (2020, noviembre 13). _Teclados matriciales_. El Octavo Bit. [https://eloctavobit.com/modulos-sensores/teclados-matriciales](https://eloctavobit.com/modulos-sensores/teclados-matriciales?utm_source=gemini)
    
-   Rubén. (2013, julio 26). _Teclado Matricial con PIC_. Geek Factory. [https://www.geekfactory.mx/tutoriales-pic/teclado-matricial-con-pic/](https://www.geekfactory.mx/tutoriales-pic/teclado-matricial-con-pic/?utm_source=gemini)
    
-   summivox. (2016, junio 3). _Keyboard matrix scanning and debouncing_. Frog in the Well. [https://summivox.wordpress.com/2016/06/03/keyboard-matrix-scanning-and-debouncing/](https://summivox.wordpress.com/2016/06/03/keyboard-matrix-scanning-and-debouncing/?utm_source=gemini)
    
-   Punto Flotante S.A. (s.f.). _Conexión de un teclado matricial hexadecimal con microcontroladores PIC 16F84, 16F628, 16F88, 18F2550_. Recuperado el 14 de septiembre de 2026, de [https://www.puntoflotante.net/PROY_TECL.htm](https://www.puntoflotante.net/PROY_TECL.htm?utm_source=gemini)
    
-   ARM Limited. (s.f.). _Cortex-M System Design Kit_. Referencias de GPIO y Memory-Mapped I/O.
    
-   RISC-V International. (s.f.). _RISC-V Instruction Set Manual_. Referencias sobre acceso a periféricos base e instrucciones Load/Store.
