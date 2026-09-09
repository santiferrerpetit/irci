Mi proyecto consiste en un operador ADDIU, que es el operador ADD normal de los procesadores tipo MIPS, pero unsigned e inmediate.
Se diferencia de los otros add por el arrastre del signo y el ignorar el overflow.

¿Qué es el operador ADDIU?
El ADDIU es una instrucción de suma que combina dos tipos de valores:

RS: Un valor que viene de un registro (en tu caso, un Input de 16 bits).

Immediate: Un valor constante (un número "fijo") que viene directamente de la instrucción.

Dato importante: Aunque se llame "Unsigned" (sin signo), en arquitecturas como MIPS, el valor inmediato suele extenderse con signo para permitir restas, pero el operador no genera una excepción si hay desbordamiento (overflow). Para tu circuito en CircuitVerse, simplemente lo trataremos como una suma binaria estándar.
