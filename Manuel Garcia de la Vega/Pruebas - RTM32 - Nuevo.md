# Verificación del Set de Instrucciones Nuevas(RTM32) Manuel Garcia de la Vega


## 1. Aritmético-lógicas registro-registro

### Caso 1 — `ADD`: overflow con signo
**Descripción:** suma de dos registros; se busca el borde clásico de overflow con signo (positivo + positivo → negativo).
**Instrucción:** `ADD $7, $2, $3` → codificado como `0x0086701C`.

**Estado inicial**

| Registro | Valor |
|---|---|
| $2 | `0x7FFFFFFF` (INT_MAX) |
| $3 | `0x00000001` |
| $7 | `0x00000000` |

**Valores inyectados y justificación:** se usa `INT_MAX + 1` porque es el único par de operandos que garantiza cruzar el límite de representación de 32 bits con signo sin necesidad de acarreo previo; expone si la ALU trunca correctamente a 32 bits en complemento a 2 o si el simulador usa aritmética de mayor precisión sin truncar.

**Estado final**

| Registro | Valor | Comentario |
|---|---|---|
| $7 | `0x80000000` | Correcto: INT_MAX+1 desborda a INT_MIN |
| PC | `0x00000004` | Avanza normalmente (+4) |
| Flags | `-----` | Sin cambio |

**Conclusión:** Anduvo. La ALU trunca correctamente a 32 bits; el overflow se manifiesta como wraparound silencioso, sin ninguna bandera visible en el debugger.

---

### Caso 2 — `SUB`: underflow con signo
**Instrucción:** `SUB $7, $2, $3`.

**Estado inicial**

| Registro | Valor |
|---|---|
| $2 | `0x80000000` (INT_MIN) |
| $3 | `0x00000001` |

**Valores inyectados y justificación:** `INT_MIN - 1` es el borde opuesto al Caso 1; si la implementación no trunca en 32 bits, el resultado esperado en aritmética infinita sería `-2147483649`, imposible de representar.

**Estado final**

| Registro | Valor |
|---|---|
| $7 | `0x7FFFFFFF` |
| PC | `0x00000004` |
| Flags | `-----` |

**Conclusión:** Anduvo. `INT_MIN - 1` desborda correctamente a `INT_MAX` (wraparound de complemento a 2).

---

### Caso 3 — `SUB`: cruce de cero a negativo
**Instrucción:** `SUB $7, $2, $3`, con $2=`0x00000000`, $3=`0x00000001`.

**Valores inyectados y justificación:** verifica que `0 - 1` produzca correctamente `0xFFFFFFFF` (representación de `-1`) y no un error de "resultado negativo inválido", algo relevante porque el procesador no tiene signo explícito en el dato, solo en la interpretación.

**Estado final:** $7 = `0xFFFFFFFF`. PC=`0x4`. **Conclusión:** Anduvo.

---

## 2. Aritmética inmediata — `ADDI`

### Caso 4 — Overflow y extensión de signo de inmediato negativo
**Instrucciones:**
- `ADDI $7, $2, 1`
- `ADDI $8, $3, -1`

**Estado inicial**

| Registro | Valor |
|---|---|
| $2 | `0x7FFFFFFF` |
| $3 | `0x00000005` |

**Valores inyectados y justificación:** la primera instrucción reutiliza el borde de overflow del Caso 1 pero por la vía inmediata (formato I), para confirmar que ambos caminos de la ALU (registro-registro y registro-inmediato) comparten el mismo comportamiento de truncado. La segunda usa un inmediato **negativo** (`-1` codificado como `0x1FFFF` en complemento a 2 de 17 bits) para verificar que `SE(imm)` realmente hace *sign extension* y no *zero extension* — si extendiera con ceros, `5 + 0x1FFFF` daría un resultado completamente distinto (`0x20004`) en lugar de `4`.

**Estado final**

| Registro | Valor | Comentario |
|---|---|---|
| $7 | `0x80000000` | Overflow con signo (idéntico a ADD) |
| $8 | `0x00000004` | `5 + (-1) = 4`, confirma SE(imm) correcta |
| PC | `0x00000008` | |

**Conclusión:** Anduvo en ambos sub-casos.

---

## 3. Saltos incondicionales — `J`, `JAL`, `JR`, `JALR`

### Caso 5 — `J`: cálculo de dirección de salto
**Instrucción:** `J 2` (word offset = 2) ejecutada con PC=`0x00000000`.

**Valores inyectados y justificación:** el campo `address` es un offset en **palabras** de memoria (no bytes), por lo que un valor pequeño como `2` ya es suficiente para exponer si el hardware aplica correctamente el `<<2` de la fórmula `E(address) = {PC+4[31:29], address, 2'b0}`. Un error común en estas unidades es tratar `address` directamente como dirección de byte.

**Estado final:** PC = `0x00000008` (`2 × 4`). **Conclusión:** Anduvo — confirma que `address` se interpreta en palabras, no en bytes.

---

### Caso 6 — `JAL`: salto con enlace
**Instrucción:** `JAL 3` ejecutada con PC=`0x00000000`.

**Estado final**

| Registro | Valor | Comentario |
|---|---|---|
| PC | `0x0000000C` | `3 × 4` |
| $31 (`$at`) | `0x00000004` | `PC_anterior + 4` |
| $1 (`$ra`) | `0x00000000` | **sin modificar** |

**Valores inyectados y justificación:** se verificó explícitamente el contenido de **ambos** `$1` y `$31` tras la ejecución. La tabla de convenciones de registros (Sección 1.1) describe a `$1`/`$ra` como el registro de "Dirección de retorno", lo cual sugiere naturalmente que `JAL`/`JALR` deberían enlazar ahí. Sin embargo, la fórmula de la Tabla A.1 indica literalmente `R[31] = PC+4`, y el hardware la respeta al pie de la letra: **el enlace real ocurre en `$31`/`$at`, no en `$1`/`$ra`**. No es un bug (el emulador implementa exactamente lo que dice la tabla de operación), pero es una discrepancia notable entre la convención documentada para el programador y la operación real de la instrucción, que cualquier compilador/ensamblador para este ISA debe tener en cuenta.

**Conclusión:** Anduvo, pero se documenta la discrepancia convención-vs-implementación como hallazgo (ver sección de Hallazgos, ítem H1).

---

### Caso 7 — `JR`: salto por registro
**Instrucción:** `JR $2`, con $2 = `0x00000030`.

**Estado final:** PC = `0x00000030`. **Conclusión:** Anduvo.

---

### Caso 8 — `JALR`: ¿el enlace va a `$31` o a `$rd`?
**Instrucciones (dos variantes del mismo programa):**
- Variante A: `JALR $rd=6, $rs=2, $rt=5` — $2=`0x00000040`
- Variante B: `JALR $rd=0, $rs=2, $rt=5` — $2=`0x00000040` (equivalente a no indicar `rd`, ya que el campo por defecto es 0)

**Valores inyectados y justificación:** la Tabla A.2 documenta `JALR rs rt` con fórmula `R[31] = PC+4; PC = R[rs]`, es decir, **sin usar el campo `rd`** y con destino fijo en `$31`. Para confirmar o refutar esto se diseñaron dos variantes que solo difieren en el valor del campo `rd` de la codificación, mientras `rs` (registro objetivo del salto) y `rt` permanecen iguales.

**Estado final**

| Caso | PC final | $31 | $6 | $5 (rt) | $0 |
|---|---|---|---|---|---|
| Variante A (`rd=6`) | `0x00000040` | `0x00000000` | **`0x00000004`** | `0x00000000` | `0x00000000` |
| Variante B (`rd=0`) | `0x00000040` | `0x00000000` | — | `0x00000000` | **`0x00000004`** |

**Conclusión: NO anduvo según lo documentado.** El salto (`PC = R[rs]`) es correcto, y `rt` efectivamente se ignora (confirmando esa parte del manual). Pero el enlace **no** se escribe en `$31`: se escribe en `R[rd]`, el registro indicado por el campo `rd` de la instrucción. En la Variante B, al no fijar `rd` explícitamente (defecto 0), **la CPU escribió la dirección de retorno directamente en `$0`/`$zero`**, register que el manual describe como "hardwired a cero" e inmodificable. Esto se documenta como hallazgo mayor (ver H2 y H3 en Hallazgos), ya que compromete cualquier programa que use `JALR` asumiendo la semántica documentada de `$31` fijo.

---

## 4. Lógicas inmediatas — `ANDI`/`ANDIH`, `ORI`/`ORIH`, `XORI`/`XORIH`, `LUI`

> El manual advierte explícitamente (nota al pie, Sección 1.2): *"Actualmente hay un bug importante a solucionar en la instrucción ANDI"*. Se diseñaron pruebas específicas para intentar reproducirlo.

### Caso 9 — `ANDI`/`ANDIH`: búsqueda del bug documentado
**Instrucciones y valores:**

| Sub-caso | Instrucción | $2 (rs) | h | ims |
|---|---|---|---|---|
| 9a | `ANDI $7, $2, 0x00FF` | `0xAAAAAAAA` | 0 | `0x00FF` |
| 9b | `ANDIH $7, $2, 0x00FF` | `0xAAAAAAAA` | 1 | `0x00FF` |
| 9c | `ANDI $7, $2, 0x8000` (bit 15 de ims activo) | `0xFFFFFFFF` | 0 | `0x8000` |
| 9d | `ANDI $2, $2, 0x00F0` (rd = rs, in-place) | `0xAAAAAAAA` | 0 | `0x00F0` |
| 9e | `ANDI $7, $2, 0xFFFF` (máscara completa) | `0x12345678` | 0 | `0xFFFF` |
| 9f | `ANDIH $7, $2, 0xFFFF` (máscara completa) | `0x12345678` | 1 | `0xFFFF` |

**Valores inyectados y justificación:** el patrón `0xAAAAAAAA` (bits alternados) hace evidente cualquier corrimiento o mezcla incorrecta de bits entre mitades de la palabra. El sub-caso 9c apunta puntualmente a la hipótesis más probable de bug (confundir `ZE`, extensión con ceros, con `SE`, extensión con signo, ya que ambas funciones ya existen en el datapath para `ADDI`): si `ANDI` reutilizara por error la ruta de `SE(imm)`, el bit 15 activo de `ims` se propagaría como unos hasta el bit 31 antes de aplicar la máscara AND, dando `0xFFFF8000` en lugar de `0x00008000`. El sub-caso 9d prueba *aliasing* de registro destino=origen. Los sub-casos 9e/9f agotan la máscara completa de 16 bits en ambas mitades.

**Estado final**

| Sub-caso | $7 (o $2) obtenido | Esperado según Tabla A.1 (`ZE`/`ZC`) | ¿Coincide? |
|---|---|---|---|
| 9a | `0x000000AA` | `0x000000AA` | ✅ |
| 9b | `0x00AA0000` | `0x00AA0000` | ✅ |
| 9c | `0x00008000` | `0x00008000` (NO `0xFFFF8000`) | ✅ |
| 9d | `0x000000A0` | `0x000000A0` | ✅ |
| 9e | `0x00005678` | `0x00005678` | ✅ |
| 9f | `0x12340000` | `0x12340000` | ✅ |

**Conclusión: Anduvo en los 6 sub-casos.** No se logró reproducir el bug advertido por el manual dentro de esta batería (incluyendo la hipótesis de confusión `SE`/`ZE`, aliasing de registros y máscara completa). Se documenta como hallazgo (H4): o bien el defecto ya fue corregido en la build `RTM32-0.5` provista, o requiere un vector de entrada fuera de los aquí explorados. En cualquier caso, es un resultado válido y reportable: la prueba de regresión diseñada específicamente para el bug conocido **pasa** en esta versión.

---

### Caso 10 — `ORI`/`ORIH`
**Instrucciones:**
- `ORI $7, $2, 0x0F0F` con $2=`0xFFFF0000`
- `ORIH $7, $2, 0xF0F0` con $2=`0x0000FFFF`

**Valores inyectados y justificación:** se usan operandos que **no se solapan** con la máscara para poder distinguir a simple vista qué mitad de la palabra afecta cada variante (h=0 vs h=1).

**Estado final:** 10a → $7=`0xFFFF0F0F`; 10b → $7=`0xF0F0FFFF`. Ambos coinciden con `ZE`/`ZC`. **Conclusión:** Anduvo.

---

### Caso 11 — `XORI`/`XORIH`
**Instrucciones:** `XORI $7,$2,0xFFFF` y `XORIH $7,$2,0xFFFF`, ambas con $2=`0xAAAAAAAA`.

**Valores inyectados y justificación:** al usar una máscara de 16 unos contra un patrón alternado, el XOR invierte exactamente la mitad afectada, lo que permite verificar a simple vista cuál mitad tocó cada variante y confirmar que no hay corrimiento cruzado entre mitades.

**Estado final:** h=0 → $7=`0xAAAA5555` (mitad baja invertida); h=1 → $7=`0x5555AAAA` (mitad alta invertida). **Conclusión:** Anduvo en ambos.

---

### Caso 12 — `LUI` combinada con `ORI` (patrón documentado para cargar constantes de 32 bits)
**Instrucciones:**
- `LUI $7, 0xBEEF`
- `ORI $7, $7, 0xCAFE`

**Valores inyectados y justificación:** el manual (Sección 1.2) señala que `LUI` es "vital para cargar constantes arbitrarias de 32 bits en registros mediante la combinación con ORI". Se probó exactamente ese patrón de uso con una constante que ocupa ambas mitades (`0xBEEFCAFE`) para confirmar que `LUI` deja realmente en cero los 16 bits bajos (de lo contrario, el `ORI` posterior podría "mezclarse" con basura remanente).

**Estado final:** tras `LUI`: $7=`0xBEEF0000`. Tras `ORI`: $7=`0xBEEFCAFE`. **Conclusión:** Anduvo — confirma el idioma de carga de constantes de 32 bits documentado.

---

## 5. Accesos a memoria

### Caso 13 — `SW`/`LW`: ida y vuelta con offset negativo + verificación de endianness
**Instrucciones:**
- `SW $5, $10, -16`
- `LW $6, $10, -16`

**Estado inicial**

| Registro | Valor |
|---|---|
| $10 (puntero base) | `0x00000050` |
| $5 (dato a guardar) | `0xDEADBEEF` |

**Valores inyectados y justificación:** se usa un desplazamiento **negativo** (`-16`, codificado en complemento a 2 de 17 bits) para verificar que el cálculo de dirección efectiva `EA = R[rs] + SE(imm)` también extiende signo correctamente cuando el offset resta en lugar de sumar (EA resultante = `0x40`). El patrón `0xDEADBEEF` no tiene bytes repetidos, lo que permite leer el **orden real de bytes en memoria** sin ambigüedad.

**Estado final**

| Registro/Memoria | Valor |
|---|---|
| $6 | `0xDEADBEEF` (round-trip correcto) |
| `dump hex 0x40 +8` | `EF BE AD DE 00 00 00 00` |

**Conclusión: Anduvo**, y adicionalmente se confirma empíricamente que **la memoria es little-endian**: el byte menos significativo (`EF`) queda en la dirección más baja (`0x40`).

---

### Caso 14 — `LW`: excepción de alineación (dirección no múltiplo de 4)
**Instrucción:** `LW $7, $10, 0`, con $10=`0x00000041` (dirección impar, viola el requisito documentado de alineación a palabra).

**Valores inyectados y justificación:** el manual advierte explícitamente ("ADVERTENCIA", Sección 1.2) que una `EA` no alineada en una instrucción de palabra "generará una excepción deteniendo el procesador". Se eligió `0x41` (offset +1 respecto a un múltiplo de 4) como el caso más simple de desalineación.

**Estado final**

| Campo | Antes | Después (step 1) | Después (step 2 adicional) |
|---|---|---|---|
| PC | `0x00000000` | `0x00000004` | `0x00000004` (sin cambio) |
| CAUSE | `0x00000000` | `0x00000005` | `0x00000005` |
| BADVADR | `0x00000000` | `0x00000041` | `0x00000041` |
| VBR | `0xF0000000` | `0x00000002` | `0x00000002` |
| $7 | `0x00000000` | `0x00000000` (no se completó el load) | — |

**Conclusión: parcialmente anduvo.** El chequeo de alineación **sí** se dispara correctamente: `CAUSE` toma un código distintivo (`5`) y `BADVADR` captura con precisión la dirección ofensora, tal como documenta la Sección 1.2.1 sobre `$ecr`/`$bva`. Sin embargo, el manual (Sección 1.1) también describe `$epc` como el mecanismo para "reanudar la ejecución cuando corresponda" mediante un salto vectorizado — en esta build el `PC` **no** salta a ninguna rutina de manejo (permanece congelado en `PC+4` incluso tras pasos adicionales) y `VBR` cambia a un valor `0x00000002` que no se corresponde con ningún vector documentado. Se documenta como hallazgo H5.

---

### Caso 15 — `SH`/`LH`/`LHU`: extensión de signo vs. extensión con ceros en media palabra
**Instrucciones:**
- `SH $5, $10, 0`
- `LH $6, $10, 0`
- `LHU $7, $10, 0`

**Estado inicial:** $10=`0x00000060`, $5=`0x12348000` (solo importan los 16 bits bajos, `0x8000`, que tienen el bit de signo de media palabra activo).

**Valores inyectados y justificación:** `0x8000` es el único valor de 16 bits que simultáneamente (a) tiene el bit más significativo de la media palabra activo, maximizando la diferencia entre `LH` (debe dar `0xFFFF8000`) y `LHU` (debe dar `0x00008000`), y (b) permite confirmar que `SH` solo escribe 2 bytes y no pisa el resto de la palabra en memoria.

**Estado final**

| Registro/Memoria | Valor |
|---|---|
| `dump hex 0x60 +4` | `00 80 00 00` (little-endian, solo 2 bytes escritos) |
| $6 (`LH`) | `0xFFFF8000` |
| $7 (`LHU`) | `0x00008000` |

**Conclusión:** Anduvo — `SHE()`/`ZE()` de la Tabla A.1 se comportan exactamente como se documentan.

---

### Caso 16 — `SB`/`LB`/`LBU`: extensión de signo vs. ceros en byte
**Instrucciones:** `SB $5,$10,0` / `LB $6,$10,0` / `LBU $7,$10,0`, con $10=`0x00000070`, $5=`0x000000FF`.

**Valores inyectados y justificación:** `0xFF` es el byte con signo negativo más extremo (`-1` en complemento a 2 de 8 bits), maximiza la diferencia entre `LB` (`0xFFFFFFFF`) y `LBU` (`0x000000FF`).

**Estado final:** dump `FF 00 00 00`; $6=`0xFFFFFFFF`; $7=`0x000000FF`. **Conclusión:** Anduvo.

---

### Caso 17 — `LWX`/`LHX`/`LHUX`/`LBX`/`LBUX`: direccionamiento indexado con **tres** registros distintos
**Instrucciones:** `LWX $7,$2,$4` / `LHX $8,$2,$4` / `LHUX $9,$2,$4` / `LBX $10,$2,$4` / `LBUX $11,$2,$4`.

**Estado inicial:** $2 (rs, base)=`0x00000050`, $4 (rd, índice)=`0x00000010`; memoria en `0x60` = `0x12345678`.

**Valores inyectados y justificación:** la Tabla A.2 especifica que la dirección efectiva de estas instrucciones es `R[rs] + R[rd]` — **no** `R[rs] + R[rt]` como sería intuitivo esperar (`rt` es solo el destino del dato). Es un formato de tres registros distintos poco común, así que se probó deliberadamente con `rs`, `rt` y `rd` con valores/roles totalmente diferentes para no dejar lugar a ambigüedad sobre cuál combinación arma la dirección.

**Estado final**

| Registro | Valor | Verificación |
|---|---|---|
| $7 (`LWX`) | `0x12345678` | Palabra completa en EA=`0x60` |
| $8 (`LHX`) | `0x00005678` | Media palabra baja (bit de signo=0, coincide con LHUX) |
| $9 (`LHUX`) | `0x00005678` | ídem |
| $10 (`LBX`) | `0x00000078` | Byte menos significativo (LE) |
| $11 (`LBUX`) | `0x00000078` | ídem |

**Conclusión:** Anduvo — se confirma que la dirección efectiva usa `rs + rd` tal como documenta la tabla, un detalle fácil de pasar por alto al programar en ensamblador.

---

## 6. Saltos condicionales

### Caso 18 — `BLT`: comparación con signo en el borde `INT_MIN` vs `INT_MAX`
**Instrucción:** `BLT $2, $3, 2` (con marcador de verificación después).

**Estado inicial:** $2=`0x80000000` (INT_MIN), $3=`0x7FFFFFFF` (INT_MAX).

**Valores inyectados y justificación:** este es el par de valores que **maximiza la divergencia** entre una comparación con signo y sin signo: en signo, `INT_MIN < INT_MAX` (verdadero, la rama debe tomarse); en una comparación (incorrecta) sin signo, `0x80000000` es un número enorme y **no** sería menor que `0x7FFFFFFF`. Si la ALU de comparación usara por error el comparador unsigned, este caso lo expondría inmediatamente.

**Estado final:** PC salta a `0x0000000C`, $9=`0x00000099` (marcador "rama tomada"). **Conclusión:** Anduvo — `BLT` compara con signo correctamente.

---

### Caso 19 — `BEQ`: salto hacia atrás con offset negativo
**Instrucción:** `BEQ $0, $0, -4`, ejecutada con PC inicial en `0x00000020`.

**Valores inyectados y justificación:** todas las pruebas anteriores de salto fueron "hacia adelante". Un offset negativo ejercita el signo de `Branch(imm) = {13'{imm[16]}, imm, 2'b0}` en sentido descendente, y es la situación real que usaría un lazo (`while`/`for`) en código compilado. Con `imm=-4`, el destino esperado es `PC + 4 + (4×(-4)) = 0x20 + 4 - 16 = 0x14`.

**Estado final:** tras el salto, PC=`0x00000014` (coincide exactamente con el cálculo); la instrucción marcadora en esa dirección ejecuta y deja $9=`0x00000077`. **Conclusión:** Anduvo — confirma la aritmética de complemento a 2 del campo `Branch(imm)` en sentido "hacia atrás".

---

### Caso 20 — Barrido funcional de las 6 condiciones (`BEQ`,`BNE`,`BLT`,`BGT`,`BLE`,`BGE`)
**Diseño:** un único programa con 6 pares `[branch][instrucción-trampa "mala"][instrucción-marcador "buena"]` consecutivos. Cada branch, si evalúa su condición correctamente, debe saltar exactamente sobre la instrucción-trampa (que escribiría un valor `0xBAD` reconocible en $20 si se ejecutara indebidamente) y aterrizar en la instrucción-marcador, incrementando un registro distinto para cada condición ($21…$26).

**Estado inicial:** $2=`5`, $3=`5` (iguales, para `BEQ`/`BLE`/`BGE`), $4=`3` (menor que $2, para `BNE`/`BLT`/`BGT`).

**Valores inyectados y justificación:** este diseño convierte cualquier falla de una condición en una señal inequívoca y localizada: si **cualquiera** de las 6 condiciones evaluara mal o calculara mal el offset de salto, $20 terminaría en `0xBAD` (o alguna variante) y/o alguno de $21-$26 quedaría en 0. Permite verificar las 6 instrucciones en una sola corrida sin ambigüedad cruzada.

**Estado final**

| Registro | Valor esperado | Valor obtenido |
|---|---|---|
| $20 (trampa, no debería tocarse) | `0x00000000` | `0x00000000` ✅ |
| $21 (`BEQ` tomó) | `0x00000001` | `0x00000001` ✅ |
| $22 (`BNE` tomó) | `0x00000001` | `0x00000001` ✅ |
| $23 (`BLT` tomó) | `0x00000001` | `0x00000001` ✅ |
| $24 (`BGT` tomó) | `0x00000001` | `0x00000001` ✅ |
| $25 (`BLE` tomó) | `0x00000001` | `0x00000001` ✅ |
| $26 (`BGE` tomó) | `0x00000001` | `0x00000001` ✅ |

**Conclusión:** Anduvo — las 6 condiciones de salto evalúan y calculan el destino correctamente, sin falsos positivos/negativos ni corrimiento de dirección.

---

## 7. Comparaciones — `SLTI`, `SLTIU`, `SLT`, `SLTU`

### Caso 21 — `SLTI` vs `SLTIU`: mismo patrón de bits, resultado opuesto
**Instrucciones:** `SLTI $7,$2,1` / `SLTIU $8,$2,1`, con $2=`0xFFFFFFFF`.

**Valores inyectados y justificación:** `0xFFFFFFFF` es simultáneamente `-1` (con signo) y `4294967295` (sin signo, el mayor valor representable). Comparado contra `1`, produce el resultado **opuesto** en cada interpretación: con signo, `-1 < 1` (verdadero); sin signo, `4294967295 < 1` (falso). Es la prueba más compacta posible para confirmar que ambas instrucciones realmente usan comparadores distintos y no comparten accidentalmente la misma lógica.

**Estado final:** $7 (`SLTI`) = `1`; $8 (`SLTIU`) = `0`. **Conclusión:** Anduvo.

### Caso 22 — `SLT` vs `SLTU` (forma registro-registro)
**Instrucciones:** `SLT $7,$2,$3` / `SLTU $8,$2,$3`, con $2=`0xFFFFFFFF`, $3=`1`.

**Valores inyectados y justificación:** réplica del Caso 21 en formato R, para confirmar consistencia entre la ruta inmediata y la de registro-registro.

**Estado final:** $7=`1`, $8=`0`. **Conclusión:** Anduvo.

---

## 8. Desplazamientos — `SLL`, `SRL`, `SRA`, `SLLR`, `SRLR`, `SRAR`

### Caso 23 — `SLL`/`SRL`/`SRA` con cantidad inmediata: ¿la SRA realmente usa `$rs` como dice la tabla?
**Instrucciones:** `SLL $7,$2,$3,4` / `SRL $8,$2,$3,4` / `SRA $9,$2,$3,4`.

**Estado inicial:** $2 (rs) = `0xA0000000`, $3 (rt) = `0xF0000001` — **deliberadamente distintos** para poder distinguir cuál de los dos registros usa cada instrucción.

**Valores inyectados y justificación:** la Tabla A.2 documenta `SLL`/`SRL` operando sobre `R[rt]`, pero **`SRA` sobre `R[rs]`** — una asimetría llamativa entre tres instrucciones que deberían compartir formato. Usando dos valores fácilmente distinguibles en `rs` y `rt`, el resultado de `SRA` revela sin ambigüedad cuál registro usó realmente el hardware: desplazar aritméticamente `$2=0xA0000000` da `0xFA000000`, mientras que desplazar `$3=0xF0000001` da `0xFF000000` — valores claramente distintos.

**Estado final**

| Registro | Valor | Fuente real (deducida) |
|---|---|---|
| $7 (`SLL`) | `0x00000010` | `$3 << 4` (usa `rt`, ✅ coincide con tabla) |
| $8 (`SRL`) | `0x0F000000` | `$3 >> 4` lógico (usa `rt`, ✅ coincide con tabla) |
| $9 (`SRA`) | **`0xFF000000`** | `$3 >>> 4` aritmético (**usa `rt`, NO `rs`**) |

**Conclusión: NO anduvo según lo documentado en la Tabla A.2.** El resultado de `SRA` coincide exactamente con desplazar `$3` (`rt`), no `$2` (`rs`) como indica la fórmula `R[rd] = R[rs] ⋙ aux`. Es decir, en la implementación real `SRA` es consistente con `SLL`/`SRL` (las tres operan sobre `rt`), y es la tabla del manual la que tiene el error tipográfico puntual en esta fila. Se documenta como hallazgo H6.

---

### Caso 24 — `SLLR`/`SRLR`/`SRAR`: enmascarado del monto de desplazamiento a 5 bits
**Instrucciones:** `SLLR $7,$2,$3` / `SRLR $8,$2,$4` / `SRAR $9,$2,$4`.

**Estado inicial:** $2 (fuente del monto de shift) = `0xFFFFFFA5` (bits altos "basura" deliberadamente distintos de cero; los 5 bits bajos son `00101` = 5), $3=`0x00000001`, $4=`0x80000000`.

**Valores inyectados y justificación:** la operación usa `R[rs][4:0]` como monto de desplazamiento. Cargar el registro fuente con **todos los bits altos en 1** (`0xFFFFFF_`) y solo los 5 bits bajos con el valor de interés es la forma más directa de verificar si el hardware realmente enmascara a 5 bits o si un valor "grande" en ese registro produce un desplazamiento erróneo o un comportamiento indefinido.

**Estado final**

| Registro | Valor | Verificación |
|---|---|---|
| $7 (`SLLR`) | `0x00000020` | `1 << 5` — confirma máscara a 5 bits |
| $8 (`SRLR`) | `0x04000000` | `0x80000000 >> 5` lógico |
| $9 (`SRAR`) | `0xFC000000` | `0x80000000 >>> 5` aritmético |

**Conclusión:** Anduvo — a diferencia del Caso 23, aquí `rs`/`rt` se usan exactamente como documenta la tabla (`rs` para el monto enmascarado a 5 bits, `rt` para el valor), sin discrepancias.

---

## 9. Lógicas registro-registro — `AND`, `OR`, `XOR`, `NOR`

### Caso 25 — Patrón de bits parcialmente solapado
**Instrucciones:** `AND $7,$2,$3` / `OR $8,$2,$3` / `XOR $9,$2,$3` / `NOR $10,$2,$3`.

**Estado inicial:** $2=`0xF0F0F0F0`, $3=`0x0FF00FF0`.

**Valores inyectados y justificación:** se evitó deliberadamente usar un par estrictamente complementario (como `0xAAAAAAAA`/`0x55555555`), porque en ese caso particular `OR` y `XOR` dan el mismo resultado (`0xFFFFFFFF`) y no permiten distinguir errores de implementación entre ambas. El patrón elegido (`0xF0F0F0F0`/`0x0FF00FF0`) tiene solapamiento parcial nibble a nibble, de forma que las 4 operaciones producen 4 resultados numéricamente distintos y cada uno queda inequívocamente verificable a mano.

**Estado final**

| Registro | Valor | Esperado |
|---|---|---|
| $7 (`AND`) | `0x00F000F0` | ✅ |
| $8 (`OR`) | `0xFFF0FFF0` | ✅ |
| $9 (`XOR`) | `0xFF00FF00` | ✅ |
| $10 (`NOR`) | `0x000F000F` | ✅ (`~OR`) |

**Conclusión:** Anduvo en las 4 instrucciones.

---

## 10. Multiplicación y división

### Caso 26 — `MUL`/`MULH`/`MULHU`: producto de dos `-1`
**Instrucciones:** `MUL $7,$2,$3` / `MULH $8,$2,$3` / `MULHU $9,$2,$3`, con $2=$3=`0xFFFFFFFF`.

**Valores inyectados y justificación:** `(-1)×(-1)=1` con signo, un resultado "pequeño" que cabe perfecto en los 32 bits bajos — bueno para confirmar `MUL`/`MULH` (mitad alta debería ser `0`). Pero interpretados sin signo, `0xFFFFFFFF × 0xFFFFFFFF` es el producto sin signo **más grande posible** en 32×32, forzando el límite superior de `MULHU` (mitad alta del producto de 64 bits).

**Estado final**

| Registro | Valor | Esperado |
|---|---|---|
| $7 (`MUL`) | `0x00000001` | `(-1)×(-1)=1`, 32 bits bajos ✅ |
| $8 (`MULH`) | `0x00000000` | Producto con signo cabe en 32 bits, mitad alta = 0 ✅ |
| $9 (`MULHU`) | `0xFFFFFFFE` | Mitad alta de `0xFFFFFFFE00000001` (producto sin signo de 64 bits) ✅ |

**Conclusión:** Anduvo — confirma que `MULH` interpreta con signo y `MULHU` sin signo, con resultados de mitad alta radicalmente distintos para el mismo par de bits de entrada.

### Caso 27 — `MUL`/`MULH`/`MULHU`: `INT_MAX × 2`
**Estado inicial:** $2=`0x7FFFFFFF`, $3=`2`. **Estado final:** $7=`0xFFFFFFFE`, $8=`0x00000000`, $9=`0x00000000`. **Conclusión:** Anduvo — el producto (`4294967294`) cabe en 32 bits sin signo pero desborda el rango con signo de 32 bits; ambas mitades altas dan 0 porque el valor de 64 bits total sigue siendo pequeño.

---

### Caso 28 — `DIV`: división por cero
**Instrucción:** `DIV $7,$2,$3`, con $2=`0x00000064` (100), $3=`0x00000000`.

**Valores inyectados y justificación:** la división por cero es indefinida matemáticamente y un caso obligatorio en cualquier verificación de una unidad de división de hardware — puede manifestarse como excepción, como resultado saturado, o (peor) como un cuelgue del pipeline.

**Estado final:** CAUSE=`0x00000003`, $7 sin modificar (`0x00000000`), PC avanza a `0x4` sin completar el registro destino. **Conclusión:** Anduvo (de forma seria) — la división por cero se detecta como condición excepcional (`CAUSE=3`) en lugar de producir un resultado basura silencioso o colgar la ejecución.

---

### Caso 29 — `DIV`: `INT_MIN / -1` (el otro overflow clásico de división)
**Instrucción:** `DIV $7,$2,$3`, con $2=`0x80000000` (INT_MIN), $3=`0xFFFFFFFF` (-1).

**Valores inyectados y justificación:** este caso es la otra condición excepcional "clásica" de las unidades de división con signo (junto a la división por cero): el resultado matemático (`2147483648`) no es representable en un `int32` con signo (el rango llega hasta `2147483647`). En arquitecturas como x86 esto genera una excepción de hardware (`#DE`) tan real como la división por cero, y muchas implementaciones ingenuas lo pasan por alto.

**Estado final:** CAUSE=`0x00000003` (mismo código que la división por cero), $7 sin modificar. **Conclusión:** Anduvo — el hardware detecta también este segundo caso de overflow de división como excepción, reutilizando el mismo código de causa que la división por cero (posible indicio de que ambas condiciones comparten el mismo manejador genérico de "excepción aritmética").

---

### Caso 30 — `DIV`/`DIVU`: mismo patrón de bits, cociente opuesto
**Instrucciones:** `DIV $7,$2,$3` / `DIVU $8,$2,$3`, con $2=`0xFFFFFFFE` (-2 con signo / gigante sin signo), $3=`2`.

**Estado final:** $7=`0xFFFFFFFF` (`-2/2=-1`, con signo ✅); $8=`0x7FFFFFFF` (`4294967294/2`, sin signo ✅). **Conclusión:** Anduvo.

---

### Caso 31 — `REST`/`RESTU`: signo del resto en operandos negativos
**Instrucciones:** `REST $7,$2,$3` / `RESTU $8,$2,$3`, con $2=`0xFFFFFFFB` (-5 con signo), $3=`3`.

**Valores inyectados y justificación:** el resto de una división con operandos negativos es ambiguo entre convenciones de lenguaje (resto "trunca hacia cero", como C, vs. resto "estilo matemático", siempre positivo, como Python). Este caso expone cuál convención implementa el hardware.

**Estado final:** $7 (`REST`, con signo) = `0xFFFFFFFE` (`-2`, consistente con truncamiento estilo C: `-5 % 3 = -(5 truncado) = -2`); $8 (`RESTU`, sin signo) = `0x00000002` (`4294967291 % 3 = 2`). **Conclusión:** Anduvo — confirma convención de resto "con signo del dividendo" (estilo C), relevante para cualquier compilador que apunte a este ISA.

---

## 11. Registro `$zero` y registros especiales (`CFS`/`CTS`)

### Caso 32 — ¿Es realmente inmodificable `$0`?
**Instrucción:** `ADD $0, $2, $3`, con $2=`0xAAAAAAAA`, $3=`0x55555555` (complementarios exactos, para que el resultado `0xFFFFFFFF` sea inconfundible si la escritura llega a ocurrir).

**Valores inyectados y justificación:** la Sección 1.1 del manual afirma categóricamente: *"[$zero] está cableado en el hardware, así que no se puede modificar, aunque puede usarse como destino en cualquier instrucción."* Esto exige una prueba dedicada: usar `$0` explícitamente como registro destino (`rd=0`) de una operación cuyo resultado sea inconfundible.

**Estado final:** $0 = `0xFFFFFFFF`. **Conclusión: NO anduvo según lo documentado.** El registro `$0` **sí fue modificado**; no existe protección de hardware efectiva contra escritura en esta build. Esto es consistente con — y explica — el resultado del Caso 8 (Variante B), donde `JALR` con `rd=0` sobrescribió `$zero` con la dirección de retorno. Se documenta como hallazgo mayor H3.

---

### Caso 33 — `CTS`/`CFS`: ida y vuelta a registros especiales
**Instrucciones:** `CTS $1, aux` seguido de `CFS $2, aux`, para `aux` = 0, 1, 2, 3, 4 (los 5 registros especiales documentados: `$psw`, `$ecr`, `$epc`, `$bva`, `$vbr`, en ese orden), y también variando qué campo de la codificación R (`aux`, `rt`, `rd`) transporta el índice.

**Valores inyectados y justificación:** se probaron valores distintos por cada índice para poder rastrear inequívocamente si el valor "pega" en el registro especial correspondiente, y se probaron variantes de codificación por si el índice del registro especial no viaja en el campo `aux` sino en otro.

**Estado final:** en las 5×3 = 15 combinaciones probadas, `CFS` devolvió siempre `0x00000000`, y ninguno de los registros especiales visibles (`CAUSE`, `EPC`, `BADVADR`, `VBR`) ni el bloque `Mode/Flags` (que se infiere corresponde a `$psw` desplegado) cambiaron de valor tras `CTS`.

**Conclusión: NO anduvo.** `CFS`/`CTS` no producen ningún efecto observable en esta build del emulador, independientemente de la codificación de campos probada. Se documenta como hallazgo H7 (posible instrucción no implementada en `RTM32-0.5`).

---

## 12. Excepciones por software — `TRAP`/`RFT`

### Caso 34 — `TRAP`: vector de excepción por software
**Instrucciones:**
- `0x00`: `TRAP 4` (debería saltar a `M[4<<2] = M[0x10]`)
- `0x10`: `ADDI $9, $0, 0x55` (cuerpo del "manejador")
- `0x14`: `RFT` (debería retornar a `EPC`)

**Valores inyectados y justificación:** `aux=4` es un índice de vector simple y fácil de verificar a mano (`4<<2=0x10`); se sitúa además un valor reconocible (`0x55`) en el cuerpo del supuesto manejador para detectar sin ambigüedad si efectivamente se alcanza a ejecutar.

**Estado final**

| Paso | PC | CAUSE | EPC | $9 |
|---|---|---|---|---|
| Antes | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| Tras step 1 (`TRAP`) | `0x00000004` | `0x00000003` | `0x00000000` | `0x00000000` |
| Tras step 2 | `0x00000004` (congelado) | `0x00000003` | `0x00000000` | `0x00000000` |
| Tras step 3 | `0x00000004` (congelado) | `0x00000003` | `0x00000000` | `0x00000000` |

**Conclusión: NO anduvo según lo documentado.** Según la Tabla A.2, `TRAP` debería fijar `EPC = PC+4` (`0x00000004`) y saltar a `PC = M[aux<<2]` (`0x00000010`). En cambio: `EPC` nunca se actualiza (queda en `0`), `PC` no salta al vector y **queda congelado** en `0x00000004` incluso tras pasos adicionales — el mismo patrón de "congelamiento" observado en el Caso 14 (excepción de alineación). El manejador nunca se alcanza (`$9` nunca toma `0x55`), y por lo tanto tampoco pudo ejercitarse `RFT` de forma significativa en este flujo. Se documenta como hallazgo H8.

---

## Hallazgos y anomalías detectadas

Resumen consolidado de discrepancias entre el manual `rtm32.pdf` y el comportamiento observado en el emulador `RTM32-0.5`, todas reproducidas de forma determinística:

| # | Instrucción | Hallazgo | Severidad |
|---|---|---|---|
| H1 | `JAL` | El enlace ocurre en `$31`/`$at`, no en `$1`/`$ra` como sugiere la tabla de convenciones de registros (Sección 1.1). Consistente con la fórmula de la Tabla A.1, pero contradice la convención de nombres documentada. | Baja (documentación ambigua, no bug de ejecución) |
| H2 | `JALR` | El destino del enlace es `R[rd]` (campo de la codificación), **no** `$31` fijo como indica la Tabla A.2 (`R[31] = PC+4`). `rt` se confirma ignorado. | **Alta** — cambia semántica de la instrucción |
| H3 | `$zero` / `JALR` | El registro `$0` puede ser modificado por cualquier instrucción que lo use como destino (probado con `ADD` y con `JALR` de rd=0 por defecto), contradiciendo "hardwired a cero, no se puede modificar" (Sección 1.1). | **Alta** — puede corromper silenciosamente cualquier programa que asuma `$zero` constante |
| H4 | `ANDI`/`ANDIH` | El bug advertido por el manual (nota al pie, Sección 1.2) no se reprodujo en 6 vectores de prueba distintos (incluyendo la hipótesis de confusión `SE`/`ZE`, aliasing de registro y máscara completa). | Informativo — posible corrección ya aplicada en esta build |
| H5 | `LW` (y excepciones en general) | Ante una excepción (alineación, división), `CAUSE`/`BADVADR` se fijan correctamente, pero el `PC` no salta al vector de excepción (`VBR`) y queda congelado; `VBR` además se sobreescribe con `0x00000002`, un valor no documentado, en lugar de preservar `0xF0000000`. | Media-Alta — el mecanismo de excepciones no está completo en esta build |
| H6 | `SRA` | La Tabla A.2 documenta `R[rd] = R[rs] ⋙ aux`, pero el hardware real usa `R[rt]`, igual que `SLL`/`SRL`. Es un error tipográfico en la tabla, no en la implementación. | Baja (documentación), pero crítica si alguien programa siguiendo la tabla al pie de la letra |
| H7 | `CFS`/`CTS` | Ningún efecto observable en 15 combinaciones de codificación probadas. | Media — instrucción aparentemente no implementada |
| H8 | `TRAP`/`RFT` | `TRAP` no actualiza `EPC` ni salta al vector `M[aux<<2]`; el PC queda congelado (mismo patrón que H5), por lo que `RFT` no pudo verificarse en un flujo real. `CAUSE` toma el mismo código (`3`) que las excepciones de división. | Media-Alta — mecanismo de excepciones por software no operativo |
| H9 | Todas las aritméticas | Ningún caso (incluyendo overflows de `ADD`/`SUB`/`DIV`) modificó el campo `Flags` del debugger (`Mode: KERNEL \| Flags: [-----]`), pese a que el manual describe `$psw` conteniendo "banderas aritméticas". | Informativo |

---
