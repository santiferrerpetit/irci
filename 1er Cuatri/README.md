# Implementación de ADDIL

## 1. ¿Qué es ADDIL y por qué lo usamos?

Basado en la lógica de MIPS, la instrucción `ADDIL` (**Add Immediate Low**) es una variante de las instrucciones de tipo I (Inmediatas). Su función es sumar un valor constante (inmediato) de 8 bits a la parte baja de un registro de 16 bits.

En MIPS, las operaciones inmediatas permiten que el programador incluya un número directamente en la instrucción en lugar de tener que cargarlo previamente en un registro. Esto ahorra ciclos de reloj y espacio en memoria.

---
## 2.Sumador de 1 bit

La computadora realiza operaciones binarias. Para que el hardware entienda la instrucción `ADDIL`, necesitamos primero un **Sumador Completo (Full Adder)** de un solo bit.

Este es el "átomo" de nuestra unidad aritmética porque no solo suma los bits actuales ($A$ y $B$), sino que es capaz de procesar el **Acarreo de Entrada (Carry-in)**. Sin esta capacidad de recibir el acarreo del bit anterior, sería imposible realizar sumas de números grandes, ya que no habría comunicación entre las columnas de bits.
![[FullAdder.svg]]

---
## 3. Escalamiento a 16 bits

Los registros tienen un ancho fijo (en nuestro caso, 16 bits). Para procesar estos datos, conectamos 16 sumadores de un bit en una cadena de **Acarreo Propagado (Ripple Carry)**.

Cada bit calcula su suma y "pasa la bola" (el acarreo) al siguiente bit. Esto replica físicamente el proceso manual de suma donde "nos llevamos uno" a la siguiente columna. Es lo que permite que una instrucción aritmética sea consistente en todo el bus de datos de 16 bits.

![[Sumador 16bits.svg]]

---

## 4. Implementación del ADDIL mediante Zero Extension

Una de las claves que menciona la teoría de MIPS es que los inmediatos suelen ser más cortos que los registros (en tu caso, un inmediato de 8 bits para un procesador de 16).

Para que el **Sumador de 16 bits** funcione correctamente sin alterar los bits superiores de forma errónea, implementamos la **Extensión de Ceros (Zero Extension)**.

- Los 8 bits del inmediato se conectan a los 8 bits bajos del sumador.
    
- Los 8 bits superiores del sumador se fuerzan a `0`.
    
- De esta manera, el `ADDIL` cumple su misión: modifica la parte baja del registro y solo altera la parte alta si existe un acarreo real que viaje desde el bit 7 hacia arriba.
    
![[ADDIL.svg]]

---

## Fuentes y Bibliografía

* **Kann, C. W. (2015).** *Introduction To MIPS Assembly Language Programming*. Gettysburg College.  [Sección 3.2: Addition in MIPS Assembly](https://eng.libretexts.org/Bookshelves/Computer_Science/Programming_Languages/Introduction_To_MIPS_Assembly_Language_Programming_(Kann)/03%3A_MIPS_Arithmetic_and_Logical_Operators/3.02%3A_Addition_in_MIPS_Assembly)
* **IBM Documentation.** _ADDI (Add Immediate) or CAL (Compute Address Lower) Instruction_. [AIX 7.2 Instruction Set](https://www.ibm.com/docs/es/aix/7.2.0?topic=is-addi-add-immediate-cal-compute-address-lower-instruction)


