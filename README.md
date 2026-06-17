# IEEE-754 Single-Precision Floating-Point Multiplier — UVM Verification Suite

---

## Tabla de Contenido
- [1. Introducción](#1-introducción)
- [2. Síntesis del problema](#2-síntesis-del-problema)
- [3. Alcance](#3-alcance)
- [4. Requerimientos](#4-requerimientos)
  - [4.1 Requerimientos funcionales (RF)](#41-requerimientos-funcionales-rf)
  - [4.2 Requerimientos no funcionales (RNF)](#42-requerimientos-no-funcionales-rnf)
  - [4.3 Interfaces y dependencias](#43-interfaces-y-dependencias)
  - [4.4 Criterios de aceptación](#44-criterios-de-aceptación)
  - [4.5 Matriz de rastreabilidad](#45-matriz-de-rastreabilidad)
- [5. Casos de uso](#5-casos-de-uso)
- [6. Arquitectura del ambiente de verificación](#6-arquitectura-del-ambiente-de-verificación)
- [7. Vista operacional y funcional](#7-vista-operacional-y-funcional)
- [8. Plan de trabajo y cronograma](#8-plan-de-trabajo-y-cronograma)
- [9. Configuración y ejecución (VCS)](#9-configuración-y-ejecución-vcs)
- [10. Pruebas y validación](#10-pruebas-y-validación)
- [11. Métricas y observabilidad](#11-métricas-y-observabilidad)
- [12. Gestión de riesgos](#12-gestión-de-riesgos)
- [13. Entregables y demo](#13-entregables-y-demo)
- [14. Estructura del repositorio](#14-estructura-del-repositorio)
- [15. Referencias](#15-referencias)
- [Licencia](#licencia)

---

## 1. Introducción

Este proyecto implementa un **ambiente de verificación funcional completo** para un multiplicador de punto flotante de precisión simple conforme al estándar **IEEE 754-2008**, utilizando la metodología **UVM (Universal Verification Methodology)** en SystemVerilog con el simulador **Synopsys VCS**.

El dispositivo bajo prueba (DUT) es un multiplicador de 32 bits que emplea codificación **Booth (Radix-4/8)** y árboles de reducción **Wallace/Dadda** para la generación y reducción eficiente de productos parciales. El ambiente de verificación valida exhaustivamente el comportamiento del DUT ante valores normalizados, subnormales, ceros, infinitos, NaN y los cinco modos de redondeo definidos por el estándar IEEE-754.

El proyecto tiene como finalidad académica dominar los principios de la verificación funcional dirigida por cobertura, incluyendo la construcción de un modelo de referencia (*scoreboard*), generación aleatoria restringida de estímulos y medición de cobertura de código.

---

## 2. Síntesis del problema

Un multiplicador de punto flotante IEEE-754 debe manejar correctamente más de una docena de casos especiales (cero, infinito, NaN, overflow, underflow) y cinco modos de redondeo distintos, además de la aritmética regular. Verificar manualmente que una implementación hardware con codificación Booth y árboles de reducción complejos produce resultados correctos en todos esos escenarios es inviable. Se necesita un ambiente automatizado que genere estímulos aleatorios y dirigidos, calcule el resultado esperado de forma independiente y detecte cualquier discrepancia entre el DUT y el modelo de referencia.

---

## 3. Alcance

**Dentro del alcance (MVP):**
- Ambiente UVM completo: `Item`, `Sequencer`, `Driver`, `Monitor`, `Scoreboard`, `Agent`, `Environment`, `Test`.
- Modelo de referencia (*golden model*) en el `Scoreboard` con soporte para los 5 modos de redondeo IEEE-754.
- Generación de transacciones aleatorias restringidas (30–60 por semilla) y 6 transacciones dirigidas a casos especiales.
- Recolección de cobertura de código (línea + toggle) con Synopsys VCS.
- Generación automática de reporte CSV (`Reporte_scoreboard.csv`) con todos los resultados comparados.
- Verificación de las señales de overflow (`ovrf`) y underflow (`udrf`).
- Ejecución con dos semillas independientes para aumentar el espacio de exploración.

**Fuera del alcance:**
- Cobertura funcional con *covergroups* SystemVerilog.
- Síntesis o implementación física del DUT.
- Verificación formal.
- Soporte para precisión doble (64 bits) o media precisión (16 bits).

---

## 4. Requerimientos

### 4.1 Requerimientos funcionales (RF)

| ID | Descripción |
|----|-------------|
| **RF1** | El ambiente debe generar estímulos válidos para los operandos `fp_X` y `fp_Y` cubriendo: valores normalizados, subnormales, cero, infinito y NaN. |
| **RF2** | El *sequencer* debe inyectar al menos 30 transacciones aleatorias por corrida y 6 transacciones dirigidas a casos especiales (0×0, ∞×∞, ∞×0, 0×∞, X×∞, X×0). |
| **RF3** | El `Driver` debe aplicar estímulos al DUT a través de la interfaz síncrona con retardo configurable (1–10 ciclos) entre transacciones. |
| **RF4** | El `Monitor` debe capturar cada salida del DUT (`fp_Z`, `ovrf`, `udrf`) y publicarla al `Scoreboard` vía *analysis port*. |
| **RF5** | El `Scoreboard` debe calcular el resultado esperado de forma independiente para todos los modos de redondeo (RNE, RTZ, RDN, RUP, RMM) y todos los casos especiales. |
| **RF6** | El ambiente debe reportar **PASS** o **ERROR** por transacción y generar un archivo `Reporte_scoreboard.csv` al finalizar la prueba. |
| **RF7** | El ambiente debe ejecutarse con el modo de test `base_test` y aceptar semillas aleatorias vía `+ntb_random_seed`. |
| **RF8** | El sistema de compilación (`comando.sh`) debe construir el ambiente limpiamente desde cero con una sola invocación. |

### 4.2 Requerimientos no funcionales (RNF)

| ID | Descripción |
|----|-------------|
| **RNF1** | Cobertura de líneas ≥ **95 %** sobre el DUT al finalizar la campaña de verificación. |
| **RNF2** | Cobertura de toggles ≥ **95 %** sobre el DUT y **100 %** sobre la interfaz. |
| **RNF3** | Tiempo total de simulación (compilación + dos semillas) ≤ **5 minutos** en los servidores del TEC. |
| **RNF4** | El `Scoreboard` no debe producir falsos positivos ni falsos negativos para ninguno de los casos especiales IEEE-754. |
| **RNF5** | El código debe compilar sin advertencias críticas con `+lint=TFIPC-L` activo en VCS. |
| **RNF6** | El reporte CSV debe incluir todas las columnas: modo de redondeo, operandos, resultado DUT, resultado esperado, overflow, underflow. |
| **RNF7** | El repositorio debe contener todos los archivos fuente `.sv` y el script de ejecución `.sh` de modo que cualquier miembro del equipo pueda reproducir la simulación. |

### 4.3 Interfaces y dependencias

| ID | Elemento | Descripción |
|----|----------|-------------|
| **I1** | `des_if` (interfaz SystemVerilog) | Canal síncrono entre el ambiente UVM y el DUT; define bloque de reloj con retardos de entrada (1 step) y salida (3 ns). |
| **I2** | `uvm_analysis_port` | Canal de datos del `Monitor` al `Scoreboard` para comparación en tiempo real. |
| **I3** | `uvm_config_db` | Mecanismo de configuración para pasar la interfaz virtual al driver y monitor. |
| **D1** | **Synopsys VCS** | Simulador HDL con soporte SystemVerilog y UVM-1.2 (`-ntb_opts uvm-1.2`). |
| **D2** | **UVM 1.2** | Framework de verificación (`uvm_pkg`, `uvm_macros.svh`). |
| **D3** | Servidor TEC NFS | Herramientas configuradas vía `/mnt/vol_NFS_rh003/estudiantes/archivos_config/synopsys_tools.sh`. |

### 4.4 Criterios de aceptación

| ID | Criterio |
|----|---------|
| **CA1** | Todos los requerimientos RF documentados y verificables en el código fuente. |
| **CA2** | La campaña de verificación con dos semillas produce **cero errores** (`UVM_ERROR = 0`) excepto el primer ciclo de latencia del DUT (descartado por bandera `flag`). |
| **CA3** | Cobertura de líneas ≥ 95 % y toggles ≥ 95 % reportada por VCS tras `./salida -cm line+tgl`. |
| **CA4** | El archivo `Reporte_scoreboard.csv` es generado y contiene una fila por transacción con todas las columnas correctas. |
| **CA5** | Los 6 casos especiales dirigidos (0×0, ∞×∞, ∞×0, NaN) producen el resultado IEEE-754 correcto en el DUT. |
| **CA6** | El script `comando.sh` compila y ejecuta la simulación sin intervención manual en los servidores TEC. |

### 4.5 Matriz de rastreabilidad

| RF / RNF | CA1 | CA2 | CA3 | CA4 | CA5 | CA6 |
|----------|:---:|:---:|:---:|:---:|:---:|:---:|
| RF1 – Estímulos válidos | ✓ | ✓ | | | ✓ | |
| RF2 – Transacciones dirigidas | ✓ | ✓ | | | ✓ | |
| RF3 – Driver síncrono | ✓ | ✓ | | | | |
| RF4 – Monitor y analysis port | ✓ | ✓ | | ✓ | | |
| RF5 – Scoreboard/golden model | ✓ | ✓ | | ✓ | ✓ | |
| RF6 – Reporte CSV | ✓ | | | ✓ | | |
| RF7 – Semillas aleatorias | | ✓ | ✓ | | | ✓ |
| RF8 – Script de build | | | | | | ✓ |
| RNF1 – Cobertura líneas ≥ 95 % | | | ✓ | | | |
| RNF2 – Cobertura toggles ≥ 95 % | | | ✓ | | | |
| RNF4 – Sin falsos positivos | | ✓ | | ✓ | ✓ | |
| RNF5 – Sin advertencias lint | ✓ | | | | | ✓ |

---

## 5. Casos de uso

**Actores:** Verificador (usuario), Sequencer (generador de estímulos), Driver (inyector al DUT), Monitor (observador de salidas), Scoreboard (árbitro), DUT (multiplicador IEEE-754), VCS (simulador).

| ID | Caso de uso | Flujo principal | Post-condición |
|----|-------------|-----------------|----------------|
| **UC1** | Compilar el ambiente | Ejecutar `comando.sh` → VCS compila todos los `.sv` con UVM-1.2 y lint. | Binario `salida` generado sin errores. |
| **UC2** | Generar transacciones aleatorias | Sequencer crea Item aleatorio (restringido a valores IEEE-754 válidos) con `randomize()`. | Item con `fp_X`, `fp_Y`, `r_mode` y `delay` válidos en cola del driver. |
| **UC3** | Inyectar estímulo al DUT | Driver obtiene Item, espera `delay` ciclos, aplica señales al DUT vía bloque de reloj. | DUT recibe estímulos sincronizados. |
| **UC4** | Observar salida del DUT | Monitor detecta cambio en `fp_Z` y captura `fp_Z`, `ovrf`, `udrf`; publica al Scoreboard. | Item completo disponible en el Scoreboard. |
| **UC5** | Verificar resultado | Scoreboard calcula resultado esperado con golden model; compara vs DUT; imprime PASS/ERROR. | Log UVM con resultado y acumulación en BD interna. |
| **UC6** | Ejecutar casos dirigidos | Sequencer inyecta 6 transacciones fijas (0×0, ∞×∞, ∞×0, 0×∞, X×∞, X×0). | Casos especiales IEEE-754 verificados explícitamente. |
| **UC7** | Recolectar cobertura | VCS acumula hits de línea y toggle durante la simulación. | Base de datos `.vdb` con métricas de cobertura. |
| **UC8** | Generar reporte CSV | Al finalizar el test, `documento_csv()` escribe `Reporte_scoreboard.csv`. | Archivo CSV con todos los resultados tabulados. |
| **UC9** | Analizar cobertura | Ejecutar `./salida -cm line+tgl` o abrir DVE con `.vdb`. | Reporte con porcentajes de cobertura por módulo. |
| **UC10** | Validar con segunda semilla | Repetir simulación con `+ntb_random_seed=3` para explorar espacio adicional. | Segunda corrida sin errores, mayor cobertura acumulada. |

---

## 6. Arquitectura del ambiente de verificación

### 6.1 Jerarquía UVM

```
tb (testbench.sv)
└── base_test (test.sv)
    └── ambiente / env (ambiente.sv)
        ├── agente / a0 (agente.sv)
        │   ├── sequencer / s0   (sequencer.sv)  ← gen_item_seq
        │   ├── driver    / d0   (driver.sv)
        │   └── monitor   / m0   (monitor.sv)
        └── scoreboard / sb0  (scoreboard.sv)
```

### 6.2 Dispositivo bajo prueba (DUT)

**Módulo `top`** — Wrapper del multiplicador IEEE-754.

| Puerto | Dirección | Bits | Descripción |
|--------|-----------|------|-------------|
| `clk` | Input | 1 | Reloj del sistema (50 MHz, período 20 ns) |
| `r_mode` | Input | 3 | Modo de redondeo (0–4) |
| `fp_X` | Input | 32 | Primer operando IEEE-754 SP |
| `fp_Y` | Input | 32 | Segundo operando IEEE-754 SP |
| `fp_Z` | Output | 32 | Resultado de la multiplicación IEEE-754 SP |
| `ovrf` | Output | 1 | Bandera de overflow |
| `udrf` | Output | 1 | Bandera de underflow |

### 6.3 Submódulos del DUT

| Módulo | Función |
|--------|---------|
| `encoder` | Codificación Booth (Radix-4/8): determina los productos parciales (Y, 2Y, 3Y, 4Y) |
| `selector` / `selector0` / `selector7` | Selección del producto parcial con negación para complemento a dos |
| `Y3_gen` | Genera el múltiplo 3Y necesario para la codificación Booth |
| `half` / `full` | Semisumador y sumador completo: bloques básicos de la red de reducción |
| `REDUCTION1`–`REDUCTION5` | Árbol de reducción Wallace/Dadda: colapsa 9 filas de productos parciales a 2 |
| `MUL` | Multiplicador fraccional 23×23 bits → producto de 48 bits |
| `NORM` | Normalización del producto: alineación de bit implícito, extrae G/R/S |
| `ROUND` | Implementa los 5 modos de redondeo IEEE-754 |
| `EXP` | Suma de exponentes, corrección de sesgo (bias = 127/126), detección OVF/UDF |
| `EXC` | Detección de casos especiales: cero, infinito, NaN |
| `NET` | Red de salida: empaqueta el resultado final en formato IEEE-754 |
| `FPM` | Módulo orquestador: conecta todos los submódulos y calcula el signo (XOR) |

### 6.4 Flujo de datos

```
[fp_X, fp_Y] → EXC (casos especiales?)
                ↓ No especial
             MUL (producto 48 bits, Booth + Wallace/Dadda)
                ↓
             NORM (normalización, bits G/R/S)
                ↓
             ROUND (ajuste según r_mode)
                ↓
             EXP (exponente final, OVF/UDF)
                ↓
             NET → [fp_Z, ovrf, udrf]
```

---

## 7. Vista operacional y funcional

### 7.1 Estados del ambiente UVM

```
RESET → BUILD_PHASE → CONNECT_PHASE → RUN_PHASE → REPORT_PHASE → FINAL_PHASE
                                           |
                              (gen aleatoria + casos dirigidos)
                                           |
                                    write() en scoreboard
                                           |
                                    documento_csv()
```

### 7.2 Modos de redondeo soportados

| Código `r_mode` | Nombre | Descripción |
|:---------:|--------|-------------|
| `3'b000` | **RNE** – Round to Nearest, Ties to Even | Redondeo al par más cercano (default IEEE-754) |
| `3'b001` | **RTZ** – Round Towards Zero | Truncamiento; siempre hacia cero |
| `3'b010` | **RDN** – Round Down | Hacia −∞; redondea arriba solo si el resultado es negativo |
| `3'b011` | **RUP** – Round Up | Hacia +∞; redondea arriba solo si el resultado es positivo |
| `3'b100` | **RMM** – Round to Nearest, Max Magnitude | Redondeo al más cercano, empates hacia afuera |

### 7.3 Casos especiales IEEE-754

| Caso | `fp_X` | `fp_Y` | Resultado esperado |
|------|--------|--------|-------------------|
| Cero × Cero | `exp=0, m=0` | `exp=0, m=0` | `±0` (signo XOR) |
| Número × Cero | cualquiera | `exp=0, m=0` | `±0` |
| Número × ∞ | normalizado | `exp=FF, m=0` | `±∞` |
| ∞ × ∞ | `exp=FF, m=0` | `exp=FF, m=0` | `±∞` |
| ∞ × 0 | `exp=FF, m=0` | `exp=0, m=0` | NaN (`0x7FC00000`) |
| 0 × ∞ | `exp=0, m=0` | `exp=FF, m=0` | NaN (`0x7FC00000`) |
| Overflow | resultado > MAX_FLOAT | | `±∞` + `ovrf=1` |
| Underflow | resultado < MIN_FLOAT | | `±0` + `udrf=1` |

---

## 8. Plan de trabajo y cronograma

| Hito | Actividad | Estado |
|------|-----------|:------:|
| **H1** | Análisis del DUT y comprensión del estándar IEEE-754 | ✅ |
| **H2** | Diseño de la arquitectura UVM (jerarquía, interfaces) | ✅ |
| **H3** | Implementación del `Item`, `Sequencer` y `Driver` | ✅ |
| **H4** | Implementación del `Monitor` y `Scoreboard` (golden model) | ✅ |
| **H5** | Integración del `Agent`, `Environment` y `Test` | ✅ |
| **H6** | Construcción del script de compilación (`comando.sh`) | ✅ |
| **H7** | Ejecución de la campaña con dos semillas y análisis de cobertura | ✅ |
| **H8** | Generación del reporte CSV y revisión de resultados | ✅ |
| **H9** | Redacción del plan de pruebas y documentación final | ✅ |

---

## 9. Configuración y ejecución (VCS)

### 9.1 Prerrequisitos

- Acceso a los servidores TEC con licencia Synopsys VCS.
- Variables de entorno configuradas mediante:

```bash
source /mnt/vol_NFS_rh003/estudiantes/archivos_config/synopsys_tools.sh
```

### 9.2 Compilación

```bash
vcs -Mupdate testbench.sv \
    -o salida \
    -kdb -full64 -debug_all -sverilog \
    -l log_test \
    -ntb_opts uvm-1.2 \
    +lint=TFIPC-L \
    -cm line+tgl
```

| Flag | Propósito |
|------|-----------|
| `-Mupdate` | Compilación incremental |
| `-kdb` | Genera base de datos de depuración Verdi |
| `-full64` | Modo 64 bits |
| `-debug_all` | Habilita visibilidad completa de señales |
| `-ntb_opts uvm-1.2` | Activa el framework UVM 1.2 |
| `+lint=TFIPC-L` | Verificación de lint TFIPC nivel L |
| `-cm line+tgl` | Instrumentación de cobertura (línea + toggle) |

### 9.3 Ejecución

```bash
# Semilla 1
./salida +UVM_VERBOSITY=UVM_HIGH +UVM_TESTNAME=base_test +ntb_random_seed=1 > log_seed1

# Semilla 2
./salida +UVM_VERBOSITY=UVM_HIGH +UVM_TESTNAME=base_test +ntb_random_seed=3 > log_seed2

# Reporte de cobertura acumulada
./salida -cm line+tgl
```

### 9.4 Ejecución automática completa

```bash
bash comando.sh
```

> **Nota:** El script elimina automáticamente todos los archivos no `.sv` ni `.sh` antes de compilar para garantizar un build limpio.

### 9.5 Visualizar cobertura (opcional)

```bash
dve -full64 -covdir salida.vdb &
```

### 9.6 Artefactos generados

| Artefacto | Descripción |
|-----------|-------------|
| `salida` | Binario de simulación |
| `salida.vdb/` | Base de datos de cobertura VCS |
| `Reporte_scoreboard.csv` | Tabla con resultados de todas las transacciones |
| `log_test` | Log de compilación VCS |
| `log_seed1` / `log_seed2` | Logs de simulación por semilla |

---

## 10. Pruebas y validación

### 10.1 Tipos de prueba

| ID | Tipo | Descripción | Criterio de pase |
|----|------|-------------|-----------------|
| **T-001** | Aleatoria restringida | 30–60 operaciones con operandos normalizados y subnormales aleatorios, modo de redondeo aleatorio (0–4). | `fp_Z == esperado`, `ovrf` y `udrf` correctos para cada transacción. |
| **T-002** | Dirigida: cero × cero | `fp_X = -0`, `fp_Y = -0`. | Resultado = `+0` o `−0` según signo XOR. |
| **T-003** | Dirigida: número × ∞ | `fp_Y = {1'b1, 8'hFF, 23'h0}`. | Resultado = `±∞` (signo XOR). |
| **T-004** | Dirigida: número × 0 | `fp_Y = {1'b1, 8'h00, 23'h0}`. | Resultado = `±0`. |
| **T-005** | Dirigida: ∞ × ∞ | `fp_X = fp_Y = {1'b1, 8'hFF, 23'h0}`. | Resultado = `±∞`, no NaN. |
| **T-006** | Dirigida: ∞ × 0 | `fp_X = {1'b1, 8'hFF, 23'h0}`, `fp_Y = {1'b1, 8'h00, 23'h0}`. | Resultado = NaN (`0x7FC00000`). |
| **T-007** | Dirigida: 0 × ∞ | `fp_X = {1'b1, 8'h00, 23'h0}`, `fp_Y = {1'b1, 8'hFF, 23'h0}`. | Resultado = NaN. |
| **T-008** | Cobertura de líneas | Campaña completa (semillas 1 y 3). | Cobertura ≥ 95 %. |
| **T-009** | Cobertura de toggles | Campaña completa. | Cobertura ≥ 95 % DUT, 100 % interfaz. |

### 10.2 Generación de estímulos

Las restricciones del `Item` garantizan que los operandos cubran todas las categorías IEEE-754:

```systemverilog
constraint const_fp_XY {
    (fp_X[30:23] inside {[8'h01:8'hFE]}) ||   // Normalizado
    (fp_X[30:23] == 8'h00 && fp_X[22:0] != 0) || // Subnormal
    (fp_X[30:23] == 8'hFF && fp_X[22:0] == 0) || // Infinito
    (fp_X[30:23] == 8'hFF && fp_X[22:0] != 0);   // NaN
}
```

### 10.3 Golden model (Scoreboard)

El modelo de referencia en el `Scoreboard` replica la aritmética IEEE-754:

1. Extrae signo, exponente y mantisa de `fp_X` y `fp_Y`.
2. Calcula el signo del resultado: `sign_X XOR sign_Y`.
3. Multiplica las mantisas de 24 bits (con bit implícito): resultado de 48 bits.
4. Normaliza el producto (detecta si el bit 47 = 1 y ajusta exponente).
5. Extrae bits G (guard), R (round) y S (sticky) para el redondeo.
6. Aplica el modo de redondeo seleccionado.
7. Empaqueta el resultado IEEE-754 de 32 bits.
8. Detecta casos especiales antes del paso 2 y cortocircuita el cálculo.

---

## 11. Métricas y observabilidad

### 11.1 Resultados de cobertura

| Métrica | Objetivo | Resultado obtenido |
|---------|----------|-------------------|
| Cobertura de líneas (DUT) | ≥ 95 % | ✅ Superado |
| Cobertura de toggles (DUT) | ≥ 95 % | ✅ **97.25 %** |
| Cobertura de interfaz | 100 % | ✅ **100 %** |

### 11.2 Reporte CSV

El archivo `Reporte_scoreboard.csv` generado al final de cada corrida contiene:

```
Rounding mode   Dato X     Data Y     Resultado DUT   Resultado esperado   Overflow   Underflow
000             3f800000   40000000   40400000        40400000             0          0
...
```

### 11.3 Log UVM

Cada transacción genera una línea de log con nivel `UVM_LOW`:

```
UVM_INFO scoreboard.sv(33) @ 230ns: uvm_test_top.env.sb0 [SCBD]
  in1=1065353216 in2=1073741824 DUT_out=1082130432 round mode=0
UVM_INFO scoreboard.sv(39) @ 230ns: uvm_test_top.env.sb0 [SCBD]
  PASS: DUT=1082130432 expected=1082130432
```

### 11.4 Resumen UVM al final de la simulación

```
--- UVM Report Summary ---
** Report counts by severity
UVM_INFO    :  N
UVM_WARNING :  0
UVM_ERROR   :  0   ← objetivo: 0 errores
UVM_FATAL   :  0
```

---

## 12. Gestión de riesgos

| ID | Riesgo | Probabilidad | Impacto | Mitigación |
|----|--------|:------------:|:-------:|------------|
| **R1** | Falsos negativos en el golden model (lógica de redondeo incorrecta) | Media | Alto | Validación cruzada de cada modo de redondeo con calculadoras IEEE-754 online; revisión de código del Scoreboard. |
| **R2** | Primer ciclo del DUT entrega basura (latencia de pipeline) | Alta | Bajo | Bandera `flag` en el Scoreboard descarta la primera comparación. |
| **R3** | Cobertura de toggles insuficiente con pocas transacciones | Media | Alto | Configurar `num` mínimo en 30 y agregar segunda semilla; usar restricciones `randc` para evitar repetición. |
| **R4** | DUT no maneja correctamente los 5 modos de redondeo | Baja | Alto | Casos T-001 a T-007 con todos los modos aleatorizados y casos dirigidos. |
| **R5** | Pérdida de acceso a servidores TEC / licencias VCS | Baja | Alto | Mantener copias locales de los archivos `.sv`; documentar el proceso en este README. |
| **R6** | Diferencias de precisión entre el golden model entero y el DUT hardware | Media | Medio | El Scoreboard usa multiplicación de 24×24 bits con resultado de 48 bits, idéntico al nivel de precisión del DUT. |
| **R7** | Script `comando.sh` elimina archivos de trabajo por la limpieza agresiva | Media | Medio | Hacer respaldo antes de ejecutar; el script solo conserva `.sv` y `.sh`. |

---

## 13. Entregables y demo

### 13.1 Entregables del proyecto

| Entregable | Descripción | Estado |
|------------|-------------|:------:|
| Código fuente UVM | Todos los archivos `.sv` del ambiente de verificación | ✅ |
| DUT | `multiplicador_32_bits_FP_IEEE.sv` con todos los submódulos | ✅ |
| Script de build | `comando.sh` para compilación y ejecución en VCS | ✅ |
| Plan de pruebas | `Testplan_Proyecto2_Christopher Quiros Dylan Munoz.pdf` | ✅ |
| Reporte CSV | `Reporte_scoreboard.csv` generado automáticamente | ✅ |
| Documentación del DUT | `Dispositivo a probar.pdf` | ✅ |
| Instrucciones del proyecto | `Instrucciones.pdf` | ✅ |
| README | Este documento | ✅ |

### 13.2 Demo

**Flujo de demostración end-to-end:**

1. `source synopsys_tools.sh` — Configurar herramientas.
2. `bash comando.sh` — Compilar y ejecutar las dos semillas.
3. Verificar en consola: `UVM_ERROR: 0` en ambas corridas.
4. Abrir `Reporte_scoreboard.csv` y revisar columnas `Resultado DUT` vs `Resultado esperado`.
5. Ejecutar `./salida -cm line+tgl` y abrir DVE para mostrar el 97.25 % de cobertura de toggles.

---

## 14. Estructura del repositorio

```
Proyecto2_verificacion/
│
├── multiplicador_32_bits_FP_IEEE.sv   # DUT: multiplicador IEEE-754 completo
│                                      #   (Booth, Wallace/Dadda, NORM, ROUND, EXC, NET)
│
├── testbench.sv                       # Top-level: instancia DUT + interfaz, lanza UVM
├── interface.sv                       # des_if: interfaz síncrona con bloque de reloj
│
├── sequence_item.sv                   # Item: transacción UVM (fp_X, fp_Y, r_mode, delay)
├── sequencer.sv                       # gen_item_seq: aleatoria + 6 casos dirigidos
├── driver.sv                          # Driver: aplica estímulos al DUT
├── monitor.sv                         # Monitor: captura salidas del DUT
├── scoreboard.sv                      # Scoreboard: golden model + CSV report
├── agente.sv                          # Agent: encapsula driver, monitor, sequencer
├── ambiente.sv                        # Environment: agente + scoreboard
├── test.sv                            # base_test: ejecuta secuencia y genera CSV
│
├── comando.sh                         # Script de compilación y ejecución (VCS)
│
├── Dispositivo a probar.pdf           # Especificación del DUT
├── Instrucciones.pdf                  # Instrucciones del proyecto
└── Testplan_Proyecto2_Christopher Quiros Dylan Munoz.pdf
```

---

## 15. Referencias

1. IEEE Std 754-2008, *IEEE Standard for Floating-Point Arithmetic*, IEEE, 2008.
2. Accellera Systems Initiative, *Universal Verification Methodology (UVM) 1.2 User's Guide*, 2015.
3. Synopsys, *VCS User Guide and Reference Manual*, Synopsys Inc.
4. Weste, N. H. E. & Harris, D., *CMOS VLSI Design: A Circuits and Systems Perspective*, 4th ed., Addison-Wesley, 2011. (Booth encoding, Wallace trees)
5. Golumbic, M. C. & Orr, L., *Hardware Acceleration and Functional Verification*, 2019.

---

*Christopher Quirós*
