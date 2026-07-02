# Manual de Laboratorio FODISEGUA
## Flujo completo Synopsys: VCS, Verdi, Coverage, UVM, Design Compiler, Design Vision y DFT

**Proyecto base:** CPU/MCU en SystemVerilog (FODISEGUA 2026, PabloJLM)
**Herramientas:** VCS W-2024.09, Verdi/Novas, Design Compiler (dcnxt_shell W-2024.09-SP5-1), librería Sky130
**Fuente:** construido exclusivamente a partir de los proyectos, scripts, cheatsheets, labs y reportes reales del curso.

---

## Índice

1. Preparación del ambiente
2. Flujo básico con VCS (módulo por módulo del CPU)
3. Simulación con simv
4. Verdi: guía completa de la interfaz
5. FSDB y Novas
6. Coverage
7. Proyecto UVM (ALU)
8. Design Compiler y Design Vision
9. Comparación de síntesis (compare_reports.sh)
10. DFT
11. Scripts finales reutilizables

---

## Convenciones del manual

Todos los comandos se ejecutan en Linux (bash o tcsh según el servidor). El prompt `UNIX%` indica shell de Linux, `dc_shell>` indica el shell interactivo de Design Compiler.

Estructura de proyecto de referencia (la usada en el curso):

```
FodiseguaPJLM/
├── CPU/                      Proyecto principal
│   ├── rtl/                  ALU.sv  control.sv  memory.sv  mux4.sv  register_bank.sv  top.sv
│   └── tb/                   ALU_tb.sv  control_tb.sv  memory_tb.sv  mux4_tb.sv  register_bank_tb.sv  top_tb.sv
├── uvm_alu/
│   ├── rtl/ALU.v
│   └── tb/  alu_if.sv  alu_pkg.sv  alu_seq_item.sv  alu_sequence.sv  alu_driver.sv
│             alu_monitor.sv  alu_agent.sv  alu_env.sv  alu_test.sv  tb_top.sv
└── LABS_DC/
    ├── LAB1/  rtl/  libs/  scripts/dc.tcl  reports/  outputs/  logs/
    └── LAB2/  (copia de LAB1 con técnicas avanzadas)
```

Los reportes reales citados en el capítulo 9 provienen de `/home/synuser18/FodiseguaPJLM/LABS_DC/` (corridas del 1 de julio de 2026).

---

# Capítulo 1 — Preparación del ambiente

## Objetivo

Dejar el entorno Linux listo para trabajar con VCS, Verdi y Design Compiler: variables de entorno correctas, licencias visibles, herramientas en el PATH y una estructura de carpetas ordenada. Al final de este capítulo debes poder ejecutar `vcs`, `verdi` y `dc_shell` desde cualquier directorio sin errores.

## Explicación conceptual

Las herramientas de Synopsys se instalan en rutas propias (por ejemplo `/usr/synopsys/vcs/W-2024.09-SP2-2/`) y no en las rutas estándar del sistema. Para que la shell las encuentre y para que las herramientas encuentren sus propias librerías y licencias, se definen variables de entorno. En el servidor del curso estas variables ya vienen configuradas por el administrador (normalmente en el archivo de inicio de la shell: `.bashrc` en bash o `.cshrc` en tcsh); tu trabajo es **verificarlas**, no crearlas desde cero.

Puedes saber qué shell estás usando con:

```bash
echo $0
```

Muestra el shell activo (bash, tcsh, etc.). Esto importa porque la sintaxis para definir variables cambia: `export VAR=valor` en bash, `setenv VAR valor` en tcsh.

## Variables de entorno

| Variable | Qué contiene | Quién la usa |
|---|---|---|
| `PATH` | Lista de directorios donde la shell busca ejecutables. Debe incluir los `bin/` de VCS, Verdi y Design Compiler | La shell, para resolver `vcs`, `verdi`, `dc_shell` |
| `VCS_HOME` | Ruta raíz de la instalación de VCS (ej. `/usr/synopsys/vcs/W-2024.09-SP2-2`) | VCS; también tú, para localizar documentación y ejemplos |
| `VERDI_HOME` | Ruta raíz de la instalación de Verdi | Verdi y la integración VCS↔Verdi (kdb/fsdb) |
| `NOVAS_HOME` | Ruta raíz de Novas (históricamente el visor de waveforms; en versiones modernas apunta dentro de la instalación de Verdi) | Los PLI/DPI de `$fsdbDumpfile`/`$fsdbDumpvars` |
| `SNPSLMD_LICENSE_FILE` | Puerto y host del servidor de licencias, formato `puerto@servidor` (ej. `27020@licserver`) | Todas las herramientas Synopsys al arrancar |
| `LD_LIBRARY_PATH` | Directorios adicionales de librerías compartidas (`.so`) que las herramientas cargan en runtime | Verdi/Novas y los binarios `simv` que enlazan las librerías FSDB |

Cómo inspeccionar cualquiera de ellas:

```bash
echo $VCS_HOME
echo $VERDI_HOME
echo $SNPSLMD_LICENSE_FILE
echo $LD_LIBRARY_PATH
```

`echo $VARIABLE` imprime el valor de una variable de entorno. Si alguna imprime vacío, la variable no está definida en tu sesión: revisa tu `.bashrc`/`.cshrc` o consulta al administrador del servidor. **No inventes rutas**: las rutas exactas dependen de la instalación de cada servidor.

## Verificación de las herramientas

Ejecuta estos comandos uno por uno. Cada uno debe responder con una ruta o una versión, nunca con "command not found".

```bash
which vcs        # Muestra la ruta del ejecutable VCS
which verdi      # Muestra la ruta del ejecutable Verdi
which dc_shell   # Muestra la ruta del shell de Design Compiler
```

En tcsh puedes usar la alternativa `where vcs` para localizar ejecutables.

Verificación de versiones y licencia:

```bash
vcs -full64 -ID   # Imprime versión e ID de licencia de VCS
verdi -id         # Muestra información de versión de Verdi
```

`vcs -full64 -ID` además de la versión valida implícitamente que la herramienta puede hablar con el servidor de licencias. Si la licencia falla, aquí es donde lo verás primero.

Localizar la documentación y ejemplos oficiales incluidos en la instalación:

```bash
echo $VCS_HOME
realpath $VCS_HOME/doc/examples
```

`realpath` muestra la ruta absoluta de un directorio. En la instalación del curso, por ejemplo, los ejemplos de assertions viven en:

```
/usr/synopsys/vcs/W-2024.09-SP2-2/doc/examples/assertion/Generic_Assertion/assertion_control/assertion
```

## Estructura de carpetas del proyecto

Crea la estructura de trabajo con `mkdir -p`, que crea directorios anidados y no da error si ya existen:

```bash
mkdir -p ~/FodiseguaPJLM/CPU/rtl ~/FodiseguaPJLM/CPU/tb
mkdir -p ~/FodiseguaPJLM/uvm_alu/rtl ~/FodiseguaPJLM/uvm_alu/tb
mkdir -p ~/FodiseguaPJLM/LABS_DC/LAB1/{rtl,libs,scripts,reports,outputs,logs}
```

Comandos básicos de navegación que usarás constantemente:

| Comando | Función |
|---|---|
| `pwd` | Muestra la ruta absoluta actual |
| `cd <dir>` | Entra al directorio |
| `cd` | Vuelve al home (`~`) |
| `ls` | Lista el contenido |
| `ls -la` | Lista detallada con archivos ocultos |
| `clear` | Limpia la terminal |
| `history` | Muestra el historial de comandos |
| `rm -rf <dir>` | Elimina directorio y todo su contenido (CUIDADO) |

## Qué hacer si alguna herramienta falla

| Síntoma | Causa probable | Solución |
|---|---|---|
| `vcs: command not found` | El `bin/` de VCS no está en `PATH` | Revisar `.bashrc`/`.cshrc`; verificar con `echo $PATH` |
| VCS arranca pero muere con error de licencia | `SNPSLMD_LICENSE_FILE` vacío o servidor caído | `echo $SNPSLMD_LICENSE_FILE`; probar `vcs -full64 -ID`; avisar al administrador |
| `simv` compila pero al ejecutar falla cargando librerías FSDB (`.so` no encontrada) | `LD_LIBRARY_PATH` sin las rutas de Verdi/Novas | `echo $LD_LIBRARY_PATH`; verificar que `VERDI_HOME`/`NOVAS_HOME` estén definidos |
| Verdi abre pero no encuentra la base de datos del diseño | Se compiló sin `-kdb` | Recompilar con `-kdb` (ver capítulo 2) |
| Verdi no muestra ondas | El testbench no tiene `$fsdbDumpfile`/`$fsdbDumpvars` | Ver capítulo 5 |

## Checklist final del capítulo 1

- [ ] `echo $0` te dice qué shell usas
- [ ] `which vcs`, `which verdi`, `which dc_shell` devuelven rutas válidas
- [ ] `vcs -full64 -ID` imprime versión y licencia sin errores
- [ ] `verdi -id` imprime la versión de Verdi
- [ ] `echo $VCS_HOME`, `$VERDI_HOME`, `$SNPSLMD_LICENSE_FILE` no están vacíos
- [ ] La estructura de carpetas del proyecto existe

---

# Capítulo 2 — Flujo básico con VCS

## Objetivo

Compilar y simular cada módulo del CPU con VCS, empezando por la ALU y explicando cada comando, cada flag y cada archivo generado. Al terminar sabrás exactamente qué produce VCS, qué puede borrarse y qué debe conservarse.

## Explicación conceptual

VCS (Verilog Compiler Simulator) trabaja en dos fases:

1. **Compilación** (`vcs ...`): traduce el RTL y el testbench a código C intermedio, lo compila con gcc y genera un ejecutable llamado `simv`.
2. **Simulación** (`./simv`): ese ejecutable es el que realmente corre la simulación, ejecuta el testbench y produce la salida.

Esto significa que si cambias el RTL o el testbench, debes **recompilar** antes de volver a simular: `simv` es una foto congelada del código en el momento de la compilación.

## 2.1 Flujo con la ALU

La ALU (`rtl/ALU.sv`) es combinacional pura: recibe `in1`, `in2` (8 bits), `op[3:0]` e `invalid_data`, y produce `out` (16 bits), `zero` y `error`. Decodifica `op[2:0]`: `000`=ADD, `001`=SUB, `010`=MUL, `011`=DIV (con protección de división por cero que activa `error` y satura `out` a todo unos), y trata `invalid_data` como error prioritario. Su testbench `ALU_tb.sv` usa una task `apply` llamada dentro de `repeat(n)` con estímulos `$urandom_range`, fuerza el corner case de división por cero (`op=0011, in2=0`) y el de `invalid_data=1`, imprime con `$monitor` y vuelca ondas con `$fsdbDumpfile("novas.fsdb")` / `$fsdbDumpvars`.

### Comando de compilación

Desde el directorio del proyecto CPU:

```bash
vcs -sverilog -full64 -debug_access+all -kdb tb/ALU_tb.sv rtl/ALU.sv
```

Explicación de cada elemento:

| Elemento | Explicación |
|---|---|
| `vcs` | Ejecuta el compilador Verilog Compiler Simulator. Compila los archivos HDL y genera el ejecutable `simv`, que es el que realmente realiza la simulación |
| `-sverilog` | Le indica al compilador que debe compilar con soporte de SystemVerilog. Usar siempre que el código sea `.sv` |
| `-full64` | Ejecuta todo en modo 64 bits. Requerido en la mayoría de setups; omitirlo puede fallar en sistemas de 64 bits |
| `-debug_access+all` | Hace que VCS guarde toda la información de debug. El `+all` da acceso a todos los componentes del diseño: registros, logic, wire, arrays, etc. Necesario para Verdi |
| `-kdb` | Genera la Kernel Debug Base (también llamada Knowledge Database), la base de datos que usa Verdi para navegación y debug. **Es `-kdb`, no `-kbd`** — error de tipeo común: "Keyboard Debug" no existe en VCS |
| `tb/ALU_tb.sv` | Primero se coloca el archivo de **testbench** |
| `rtl/ALU.sv` | Después el archivo de diseño |

Flags adicionales útiles:

| Flag | Explicación |
|---|---|
| `-o <nombre>` | Nombre personalizado para el ejecutable (default: `simv`) |
| `-l <logfile>` | Guarda la salida de compilación en un log (ej. `-l compile.log`) |
| `-timescale=1ns/1ps` | Define el timescale global |
| `-R` | Compila **y** ejecuta en un solo paso. Precaución: con `-R` la simulación corre pero Verdi no puede adjuntarse; omitir `-R` si se usará Verdi |

### Qué genera la compilación

Después de compilar, haz `ls` y observa:

| Archivo/Directorio | Generado por | Contiene | ¿Borrable? |
|---|---|---|---|
| `simv` | VCS (compilación) | El ejecutable de simulación | Sí, se regenera al recompilar |
| `csrc/` | VCS (compilación) | Archivos intermedios de compilación (código C generado, objetos) | Sí |
| `simv.daidir/` | VCS (compilación) | Base de datos de debug para Verdi (diseño elaborado, símbolos) | Sí, pero Verdi la necesita para abrir la sesión — bórrala solo junto con `simv` |
| `ucli.key` | simv (runtime) | Registro de comandos del intérprete UCLI de la sesión | Sí, es solo un log de sesión |
| `vcs.log` / `compile.log` | VCS si usaste `-l` | Mensajes de compilación | Consérvalo hasta verificar la corrida |
| `novas.fsdb` | simv (runtime) | Waveforms en formato FSDB (solo si el tb tiene `$fsdbDump*`) | Sí, se regenera en cada simulación |
| `novas.rc`, `novas_dump.log`, `verdiLog/` | Verdi (al abrirlo) | Configuración y logs de la sesión de Verdi | Sí |

Regla práctica: **lo único insustituible es tu código fuente (`rtl/`, `tb/`) y tus scripts.** Todo lo que empieza con `simv`, `csrc`, `novas` o `ucli` es regenerable.

### Ejecución

```bash
./simv
```

Ejecuta `simv` en el directorio actual: carga el diseño, corre el testbench, imprime las líneas de `$monitor` y termina con `$finish`. La salida esperada para la ALU es una línea por cada cambio de señal con el formato del `$monitor`:

```
op = 0011  in1 =  10  in2 =   0  invalid = 0  out = 65535  zero = 0  error = 1
```

### Cómo comprobar que todo salió correctamente

1. La compilación termina sin `Error-` y muestra al final la línea de resumen de VCS con el ejecutable generado.
2. `ls` muestra `simv`, `csrc/` y `simv.daidir/`.
3. `./simv` corre hasta `$finish` sin `Error` ni `Fatal`.
4. En la salida del `$monitor` se ven los corner cases: división por cero con `error = 1` y `out` saturado, e `invalid = 1` con `error = 1`.
5. Existe `novas.fsdb` en el directorio (porque el tb tiene los `$fsdbDump*`).

### Errores comunes en este paso

| Error | Causa | Solución |
|---|---|---|
| `-kbd: invalid option` | Tipeo de `-kdb` | Corregir a `-kdb` (Kernel Debug Base) |
| `Syntax error` apuntando a construcciones SV (`logic`, `always_comb`) | Falta `-sverilog` | Agregar `-sverilog` |
| Verdi abre pero "no database found" | Se compiló sin `-kdb` | Recompilar con `-kdb` |
| `./simv -verdivcs: invalid option` | El flag correcto es `-verdi` | Usar `./simv -verdi` |
| Resultados viejos tras editar código | Se ejecutó `simv` sin recompilar | Recompilar; opcionalmente limpiar antes |

### Limpieza antes de recompilar

```bash
rm -rf csrc/ simv* novas* ucli.key
```

| Elemento | Qué borra |
|---|---|
| `rm` | remove |
| `-r` | recursivo: borra directorios y su contenido |
| `-f` | forzado: sin confirmación, ignora archivos inexistentes |
| `csrc/` | Directorio de intermedios de compilación de VCS |
| `simv*` | El ejecutable `simv`, la base de coverage `simv.vdb` y el directorio `simv.daidir` |
| `novas*` | El `novas.fsdb` y archivos de configuración de Verdi |

## 2.2 El mismo flujo para el resto de módulos

El comando es idéntico; solo cambian los archivos. No se repite la explicación de flags.

**Memory** (RAM de 8 posiciones de 16 bits, escritura síncrona, lectura combinacional condicionada a `memoryRead`, usa solo `memoryAddress[2:0]`):

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -sverilog -full64 -debug_access+all -kdb tb/memory_tb.sv rtl/memory.sv
./simv
```

Verificar: el tb escribe 8 posiciones con `do_write` (datos `16'd40960 + addr`) y luego las lee con `do_read`; en el `$monitor` los `odata` leídos deben coincidir con lo escrito, y `odata = 0` cuando `read = 0`.

**Mux4** (combinacional, selector de 2 bits):

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -sverilog -full64 -debug_access+all -kdb tb/mux4_tb.sv rtl/mux4.sv
./simv
```

Verificar: para cada línea del `$monitor`, `dout` es igual a la entrada `din` que indica `select`.

**Register bank** (registro con reset asíncrono y write enable):

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -sverilog -full64 -debug_access+all -kdb tb/register_bank_tb.sv rtl/register_bank.sv
./simv
```

Verificar: tras `do_reset`, `out = 0`; en cada `do_write` la salida captura el dato en el flanco; en `do_hold` la salida retiene aunque `in` cambie.

**Control** (FSM RST→FETCH→EXECUTE→STORE). El testbench observa el estado interno con una referencia jerárquica `assign st_obs = dut.state;` — esto solo es posible porque compilamos con `-debug_access+all`:

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -sverilog -full64 -debug_access+all -kdb tb/control_tb.sv rtl/control.sv
./simv
```

Verificar en el `$monitor`: la secuencia de `state` es 0→1→2→3→1→2→3...; en FETCH se activan `aluin_en` y `cpu_rdy`; con opcode `101` (LOAD) aparecen `mem_rd = 1` y `selmux2 = 1` en EXECUTE; con `110` (STORE) aparece `mem_wr = 1` en STORE; con `cmd = 11_00_000` y `p_error = 1`, `nvalid = 1`.

**Top** (integración completa). Aquí la lista de archivos incluye todo el RTL; el testbench va primero:

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -sverilog -full64 -debug_access+all -kdb \
    tb/top_tb.sv \
    rtl/top.sv rtl/control.sv rtl/ALU.sv rtl/memory.sv rtl/mux4.sv rtl/register_bank.sv
./simv
```

El pipeline del top necesita 7 flancos de reloj para que el resultado se asiente, por eso la task `run_op` del testbench hace `repeat(7) @(posedge clk); #1;`. Verificar: la operación forzada de división por cero (`cmd = 00_01_011`, `din_2 = 0`) termina con `error = 1`; el par STORE (`00_01_110`) seguido de LOAD (`00_01_101`) devuelve en `dout` el dato que se escribió en memoria.

## Checklist final del capítulo 2

- [ ] Los 6 módulos compilan sin errores con el mismo patrón de comando
- [ ] Cada `./simv` corre hasta `$finish` y el `$monitor` muestra los corner cases esperados
- [ ] Sabes qué archivo genera cada etapa y cuáles son borrables
- [ ] Practicaste el ciclo limpiar → compilar → simular al menos una vez por módulo

---

# Capítulo 3 — Simulación con simv

## Objetivo

Dominar la ejecución del binario `simv`: sus opciones, sus logs, la semilla de aleatorización y la interpretación de sus mensajes.

## Explicación conceptual

`simv` es el producto de la compilación: contiene el diseño elaborado más el kernel de simulación de VCS. Todo lo que pasa "en tiempo de simulación" (valores de `$urandom_range`, `$monitor`, `$fsdbDumpvars`, coverage runtime) ocurre al ejecutar `simv`, no al compilar.

## Opciones de ejecución

| Comando | Explicación |
|---|---|
| `./simv` | Ejecuta la simulación compilada |
| `./simv -l sim.log` | Ejecuta y guarda toda la salida en `sim.log`. Recomendado siempre: el log es la evidencia de la corrida |
| `./simv -verdi` | Ejecuta la simulación y abre Verdi con la base de datos. **El flag es `-verdi`, no `-verdivcs`** |
| `./simv +define+SIM` | Pasa una macro de preprocesador a la simulación |
| `./simv +ntb_random_seed=<n>` | Fija la semilla de los generadores aleatorios (`$urandom_range`) para reproducir exactamente una corrida |
| `./simv +ntb_random_seed_automatic` | Semilla distinta en cada corrida, para explorar más estímulos |
| `./simv -cm <métricas>` | Recolecta coverage en runtime (capítulo 6) |

Sobre la semilla: los testbenches del CPU usan `$urandom_range(255, 0)`. Con la misma semilla, la secuencia de estímulos aleatorios es idéntica corrida tras corrida — así puedes reproducir un fallo. VCS imprime la semilla usada al inicio de la simulación; anótala cuando una corrida encuentre algo interesante.

## Interpretación de mensajes

| Mensaje | Significado |
|---|---|
| Cabecera con versión de VCS y semilla | Arranque normal |
| Líneas del `$monitor` | Tu salida funcional: una línea cada vez que cambia alguna señal monitoreada |
| `$finish called from file ... at time ...` | Fin ordenado de la simulación |
| `Warning-[...]` | No detiene la simulación, pero léelo: timescale implícito, señales sin conectar, etc. |
| `Error-[...]` / `Fatal` | La simulación no es válida; corrige antes de continuar |
| Simulación que no termina nunca | Falta `$finish`, o un `forever`/reloj sin condición de paro. Interrumpe con Ctrl+C |

## Cómo verificar resultados

Con el estilo de testbench del proyecto (salida `$monitor` pura, sin self-checking), la verificación es por inspección dirigida:

1. Guarda el log: `./simv -l sim.log`.
2. Filtra los corner cases con `grep`. Ejemplos reales:

```bash
grep "error = 1" sim.log        # divisiones por cero e invalid_data en ALU_tb
grep "mem_wr = 1" sim.log       # ciclos STORE en control_tb
grep "zero = 1" sim.log         # resultados cero
```

3. Compara contra el comportamiento esperado del RTL (tablas del capítulo 2).
4. Para análisis fino, abre las ondas en Verdi (capítulos 4 y 5).

## Checklist final del capítulo 3

- [ ] Toda corrida importante queda con `-l <log>`
- [ ] Sabes fijar y anotar la semilla para reproducir corridas
- [ ] Sabes distinguir Warning de Error y cuándo invalidar una corrida
- [ ] Usas `grep` sobre el log para localizar corner cases

---

# Capítulo 4 — Verdi: guía completa de la interfaz

## Objetivo

Abrir el CPU en Verdi y dominar sus ventanas principales: navegar la jerarquía, ver el fuente, trazar drivers y fanout, cruzar entre vistas y guardar/restaurar sesiones.

## Preparación

Verdi necesita dos cosas que se generan en el flujo del capítulo 2:

1. La base de datos de diseño `simv.daidir/`, producida por la compilación con `-debug_access+all -kdb`.
2. El archivo de ondas `novas.fsdb`, producido por `simv` porque los testbenches contienen `$fsdbDumpfile("novas.fsdb")` y `$fsdbDumpvars`.

Flujo previo (usando el top completo del CPU):

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -sverilog -full64 -debug_access+all -kdb \
    tb/top_tb.sv rtl/top.sv rtl/control.sv rtl/ALU.sv rtl/memory.sv rtl/mux4.sv rtl/register_bank.sv
./simv -l sim.log
```

## Formas de abrir Verdi

| Comando | Cuándo usarlo |
|---|---|
| `./simv -verdi` | Forma recomendada: ejecuta la simulación y lanza Verdi automáticamente con todo cargado |
| `verdi -dbdir simv.daidir -ssf novas.fsdb &` | Abrir una sesión post-simulación con diseño + ondas ya generados |
| `verdi -ssf novas.fsdb` | Solo cargar un archivo de ondas FSDB |
| `verdi -sv -f file.f` | Abrir Verdi con una lista de archivos fuente (`.f`) |
| `verdi` | Modo interactivo sin base de datos |

Flags de la forma post-simulación:

| Flag | Explicación |
|---|---|
| `-dbdir simv.daidir` | Indica dónde está guardada la base de datos de simulación (el diseño elaborado) |
| `-ssf novas.fsdb` | Signal Scan File: qué archivo de waveform va a abrir |
| `&` | Ejecuta en segundo plano para liberar la terminal |

## Recorrido por la interfaz

Verdi abre con tres marcos principales acoplados: el navegador de instancias a la izquierda, el código fuente al centro y (cuando se carga un FSDB) la ventana de ondas nWave abajo.

### Design Browser / Hierarchy

Panel izquierdo (pestaña "Instance"). Muestra el árbol de instancias tal como quedó elaborado:

```
top_tb
└── dut (top)
    ├── mux_a (mux4)
    ├── reg_a (register_bank)
    ├── mux_b (mux4)
    ├── reg_b (register_bank)
    ├── alu_i (ALU)
    ├── mem_i (memory)
    └── ctrl_i (control)
```

Acciones: un clic sobre una instancia carga su módulo en la ventana de fuente; expandir con la flecha muestra sub-instancias; la pestaña adyacente lista las señales (ports, regs, wires) del scope seleccionado. Desde ahí puedes arrastrar señales directamente a nWave.

### Source Window

Muestra el archivo fuente del scope activo con resaltado. Uso típico con el CPU:

- Clic en `ctrl_i` en la jerarquía → se abre `control.sv`.
- Doble clic sobre una señal en el fuente la selecciona en todas las vistas (cross probe).
- Los valores de simulación se pueden anotar sobre el fuente en el tiempo del cursor activo de nWave (menú Source → Active Annotation): verás por ejemplo el valor de `state` directamente junto a su declaración.

### Wave Window (nWave)

La ventana de ondas. Para poblarla: selecciona señales en la jerarquía o en el fuente y usa el menú contextual "Add to Waveform" (o arrástralas). Con el top del CPU, un conjunto útil es `clk`, `rst`, `cmd_reg`, `ctrl_i.state`, `alu_out`, `dout_high`, `dout_low`, `zero`, `error`.

Controles esenciales: zoom con la rueda o los botones de lupa, ajuste completo con "Zoom All", búsqueda de flancos con los botones de transición (siguiente cambio de la señal seleccionada), cambio de radix con clic derecho sobre la señal (binario/decimal/hex — útil para ver `cmd_reg` en binario y `dout` en decimal).

### Signal Window / Search

El buscador (icono de lupa o Ctrl+F según ventana) permite localizar señales por nombre en toda la jerarquía. Con nombres largos del CPU como `memoryWriteData`, escribir `memory*` filtra rápido. También puedes buscar texto dentro del Source Window.

### Data Flow / Driver Trace / Fanout Trace

Estas son las herramientas de debug estructural, y son la razón principal de compilar con `-kdb`:

- **Driver Trace**: con una señal seleccionada (ej. `error` en el top), el menú contextual "Trace Drivers" muestra qué proceso o instancia genera esa señal. En el CPU te lleva del puerto `error` del top al `always_ff` de banderas, y de ahí a `alu_error` que produce `alu_i`.
- **Fanout Trace** ("Trace Loads"): lo inverso — a dónde llega una señal. Trazar el fanout de `aluin_reg_en` muestra que llega a `reg_a.wr_en` y `reg_b.wr_en`.
- **Data Flow (Temporal Flow View)**: abre una vista de esquema donde puedes seguir cadena arriba la causa de un valor en un tiempo dado. Es la herramienta para responder "¿por qué `dout_low` vale 255 en t=815ns?": partes de `dout_low`, retrocedes por `sel2_out`, `alu_out`, y llegas a la división por cero en la ALU.

### Bookmarks

En nWave, Marker → Add Bookmark guarda una posición de tiempo+zoom con nombre (ej. "div_por_cero", "primer STORE"). Menú Bookmark para saltar entre ellas. Úsalos para marcar los corner cases del testbench.

### Cross Probe

Comportamiento por defecto de Verdi: seleccionar un objeto en una vista lo selecciona en todas. Doble clic en una transición sospechosa en nWave → el fuente salta a la línea que la genera; clic en una instancia de la jerarquía → fuente y esquema se sincronizan. Trabaja siempre con las tres vistas visibles para aprovecharlo.

### Guardar y restaurar sesión

- Guardar: File → Save Session. Genera un archivo de sesión (`.ses`) que recuerda ventanas abiertas, señales cargadas en nWave, radixes, bookmarks y zoom.
- Restaurar: File → Restore Session y seleccionar el `.ses`, o abrir Verdi apuntando a la misma `-dbdir`/`-ssf` y restaurar.

Guarda la sesión al terminar cada análisis largo del top: recargar 15 señales con sus radixes a mano es tiempo perdido.

## Errores comunes

| Problema | Causa | Solución |
|---|---|---|
| Verdi abre sin diseño en la jerarquía | Falta `-dbdir simv.daidir` o no se compiló con `-kdb` | Recompilar con `-kdb` y abrir con `-dbdir` |
| nWave vacío | No se cargó FSDB o el tb no lo generó | Verificar `novas.fsdb` existe; verificar los `$fsdbDump*` en el tb |
| Driver/Fanout trace no disponible | Compilación sin `-debug_access+all` | Recompilar con el flag |
| Cambios del RTL no se reflejan | Sesión sobre bases de datos viejas | Limpiar, recompilar, resimular, reabrir |

## Checklist final del capítulo 4

- [ ] Abriste el CPU con `verdi -dbdir simv.daidir -ssf novas.fsdb &`
- [ ] Navegaste la jerarquía completa hasta `ctrl_i` y `alu_i`
- [ ] Cargaste en nWave las señales clave y cambiaste radixes
- [ ] Hiciste un Driver Trace de `error` y un Fanout Trace de `aluin_reg_en`
- [ ] Creaste al menos un bookmark y guardaste la sesión

---

# Capítulo 5 — FSDB y Novas

## Objetivo

Entender exactamente qué cambia en el flujo para generar FSDB, qué archivos nuevos aparecen, y explotar nWave (el visor de ondas de Verdi/Novas) a fondo: cursores, mediciones, markers, snapshots y debug de RTL sobre las ondas.

## Explicación conceptual

FSDB (Fast Signal DataBase) es el formato de ondas de Verdi/Novas, equivalente funcional del VCD pero comprimido e indexado. Lo genera la simulación (no la compilación) cuando el testbench llama a las tareas del sistema de Novas:

```systemverilog
initial begin
    $fsdbDumpfile("novas.fsdb");
    $fsdbDumpvars;
    ...
end
```

- `$fsdbDumpfile("novas.fsdb")`: define el nombre del archivo FSDB de salida.
- `$fsdbDumpvars;`: sin argumentos, vuelca **todas** las señales de la jerarquía desde el tb hacia abajo.

Sin estas dos líneas, `novas.fsdb` no se genera y Verdi no puede mostrar gráficas. Todos los testbenches del CPU ya las incluyen al inicio de su bloque `initial`.

## Cambios respecto al flujo del capítulo 2

Ninguno en los comandos: la compilación con `-debug_access+all -kdb` ya deja los PLI de FSDB enlazados, y el testbench ya llama a los `$fsdbDump*`. La diferencia es únicamente qué observar después de `./simv`:

| Archivo nuevo | Aparece al | Contenido |
|---|---|---|
| `novas.fsdb` | Ejecutar `simv` | Todas las señales grabadas: clk, reset, entradas, salidas, registros internos, estados, buses |
| `novas.rc` | Abrir Verdi | Preferencias de la sesión |
| `novas_dump.log` | Ejecutar `simv` | Log del volcado FSDB |

Abrir la sesión de ondas:

```bash
verdi -dbdir simv.daidir -ssf novas.fsdb &
```

## nWave a fondo (con el top del CPU)

Carga en nWave: `clk`, `rst`, `cmdin`, `cmd_reg`, `ctrl_i.state`, `aluin_reg_en`, `aluout_reg_en`, `memoryRead`, `memoryWrite`, `selmux2`, `alu_out`, `dout_high`, `dout_low`, `zero`, `error`.

### Cursores

nWave tiene dos cursores: el **cursor activo** (clic izquierdo, línea principal) y el **marker de referencia** (clic con el botón central o arrastre del segundo cursor). La barra de estado muestra el tiempo de cada uno y el delta entre ambos.

### Zoom

| Acción | Cómo |
|---|---|
| Zoom in/out | Rueda del mouse o botones de lupa |
| Zoom a un intervalo | Arrastrar sobre la regla de tiempo |
| Ver todo | Zoom All |
| Centrar en el cursor | Zoom → Zoom Cursor |

### Mediciones

La medición fundamental es el delta entre cursor y marker. Ejercicio con el CPU: coloca el cursor en el flanco donde `cmdin` toma la instrucción de división por cero y el marker en el flanco donde `error` sube. El delta debe ser coherente con la latencia del pipeline (los 7 flancos del `run_op`, con periodo de 10ns del `always #5 clk = ~clk`). Los botones de "buscar siguiente transición" sobre una señal seleccionada permiten saltar de flanco en flanco con precisión en lugar de apuntar a mano.

### Markers

Además del marker de referencia, Marker → Add Marker crea marcadores nombrados fijos en tiempos concretos, con etiqueta visible. Marca los tres eventos forzados del `top_tb`: la división por cero, el STORE y el LOAD.

### Snapshots

nWave permite exportar la vista actual como imagen (File → Capture/Print de la ventana de ondas) para documentación. Úsalo para capturar la evidencia de cada corner case en tus reportes del curso.

### Buscar señales

En nWave: Signal → Get Signals abre el árbol completo de scopes del FSDB para añadir señales con filtro por nombre. Acepta comodines (`mem*`, `*out*`).

### Debug del RTL desde las ondas

El flujo de debug completo, con el ejemplo real de la división por cero del `top_tb`:

1. En nWave localiza el pulso de `error` (usa el marker "div_por_cero").
2. Doble clic sobre la transición → cross probe al Source Window: aterrizas en el `always_ff` de banderas de `top.sv`.
3. Clic derecho sobre `alu_error` → Trace Drivers → llegas al `always_comb` de `ALU.sv`, rama `3'b011` con `in2 == 0`.
4. Activa Active Annotation en el fuente para ver los valores de `in1`, `in2`, `op` en el tiempo del cursor: confirmas `in2 = 0`.
5. Si necesitas la causa aguas arriba (¿por qué `reg_b_q` es 0?), abre el Temporal Flow View desde la señal y retrocede: `reg_b_q` ← `mux_b_out` ← `din_2` del testbench.

Ese ciclo onda → fuente → driver → anotación → flow view es el patrón general para cualquier bug.

### Guardar sesión

Igual que en el capítulo 4: File → Save Session. La sesión de nWave conserva señales, grupos, radixes, markers y bookmarks.

## Checklist final del capítulo 5

- [ ] `novas.fsdb` se genera en cada `./simv`
- [ ] Mediste con cursor+marker la latencia de una operación del CPU
- [ ] Creaste markers nombrados en los tres corner cases del top_tb
- [ ] Completaste el ciclo de debug onda→fuente→driver con la división por cero
- [ ] Guardaste la sesión de nWave

---

# Capítulo 6 — Coverage

## Objetivo

Modificar el flujo de VCS para recolectar cobertura de código (line, condition, toggle, FSM, branch y assertion cuando exista), visualizarla en Verdi Coverage, interpretarla y usarla para mejorar los testbenches del CPU.

## Explicación conceptual

La cobertura de código mide qué tanto del RTL fue ejercitado por la simulación. Se activa en **dos** momentos: al compilar (instrumentación) y al simular (recolección). Si activas coverage solo en uno de los dos, no obtienes datos.

Flujo completo:

```
Paso 1 → Compilar (con coverage)
Paso 2 → Simular (con coverage)  → genera simv.vdb
Paso 3 → Visualizar coverage en Verdi
Paso 4 → Aumentar cobertura y repetir
```

## Paso 1 — Compilar con coverage

Ejemplo con el módulo control (el que tiene FSM, ideal para ver todas las métricas):

```bash
rm -rf csrc/ simv* novas* ucli.key
vcs -cm line+cond+fsm+tgl+branch -sverilog -full64 -kdb -debug_access+all \
    tb/control_tb.sv rtl/control.sv
```

El flag `-cm` (coverage metrics) activa las métricas en compilación. Se combinan con `+`:

| Métrica | Flag | Qué mide |
|---|---|---|
| Line | `line` | Qué líneas del código fueron ejecutadas |
| Condition | `cond` | Valores verdadero/falso de estructuras condicionales (if, case) |
| FSM | `fsm` | Estados, transiciones y secuencias de máquinas de estado |
| Toggle | `tgl` | Cambios de valor en señales: 0→1 y 1→0 |
| Branch | `branch` | Ramas tomadas en if-else, case y operadores ternarios |
| Assertion | `assert` | Ejecución y resultado de sentencias assert (solo si el código tiene assertions) |

Nota sobre assertion coverage: el sufijo `+assert` debe añadirse a los flags `-cm` (compilación **y** simulación) para que las aserciones aparezcan en la base de coverage. Si el proyecto no tiene SVA, omite `assert`.

## Paso 2 — Simular con coverage

```bash
./simv -cm line+cond+fsm+tgl+branch
```

Ejecuta la simulación y recolecta los datos según las métricas indicadas. Los resultados se guardan automáticamente en `simv.vdb`.

**Advertencia importante sobre bases viejas:** si ejecutas `./simv -cm ...` sin haber borrado el `simv.vdb` de una corrida anterior, los datos nuevos se **acumulan** sobre los viejos, y tras cambiar el testbench el reporte resulta engañoso. Borrar `simv.vdb` (está incluido en el patrón `simv*` de la limpieza) antes de recompilar garantiza resultados limpios.

## Paso 3 — Ver coverage en Verdi

```bash
verdi -cov -covdir ./simv.vdb/ &
```

| Flag | Explicación |
|---|---|
| `verdi` | Abre el visor gráfico de Synopsys |
| `-cov` | Activa la interfaz de visualización de coverage |
| `-covdir ./simv.vdb/` | Ruta a la base de datos de cobertura generada en el paso 2 |
| `&` | Ejecuta en segundo plano para liberar la terminal |

### La GUI de Verdi Coverage

Vistas principales:

- **Summary Frame (Coverage Browser)**: tabla jerárquica con el porcentaje de cobertura por métrica y por módulo. Cada fila es un scope (control_tb, dut); cada columna una métrica (Line, Cond, Toggle, FSM, Branch, Score total). Doble clic sobre un scope baja de nivel.
- **CovSrc / Hvp**: código fuente anotado con colores — **verde = cubierto, rojo = no cubierto**. Es la vista de trabajo: aquí ves exactamente qué línea, condición o rama faltó.
- **CovDetail**: detalle por métrica del objeto seleccionado: para una línea, cuántas veces se ejecutó; para toggle, si cada bit hizo 0→1 y 1→0; para la FSM, la tabla de estados y transiciones alcanzadas; para condition, la tabla de combinaciones verdadero/falso.

Filtros y navegación: en el Coverage Browser puedes filtrar por métrica (mostrar solo FSM), ordenar por score para atacar primero lo peor, y excluir scopes que no interesan (el propio testbench). Desde cualquier ítem rojo en CovDetail o CovSrc, el cross probe te lleva a la línea exacta del RTL — esa es la ruta "de Coverage hasta el RTL".

Reportes en texto: el menú de la GUI permite exportar el resumen; para automatizar existe la herramienta de línea de comandos de coverage de VCS (urg), pero el flujo del curso es la GUI.

## Paso 4 — Cómo aumentar el porcentaje de cobertura (con el proyecto real)

Método: abre CovSrc, mira lo rojo, pregunta "¿qué estímulo del testbench provocaría esto?" y agrega ese estímulo a la task o a un caso forzado. Casos concretos del CPU:

| Hueco típico (rojo) | Módulo | Estímulo que lo cubre |
|---|---|---|
| Rama `3'b011` interna con `in2 == 0` | ALU | Ya cubierta por `repeat(4) apply(4'b0011, $urandom_range(255,0), 0, 0);` — patrón a imitar |
| Rama `default` del case de `op` | ALU | Aplicar opcodes `100` o `111` (con `$urandom_range(255,0)` sobre 4 bits ya caen a veces; para garantía, forzar `apply(4'b0111, ...)`) |
| Transiciones de FSM que pasan por RST | control | El tb solo resetea al inicio; añadir un `do_reset()` a mitad de secuencia cubre STORE→RST/EXECUTE→RST vía reset asíncrono |
| Toggle de bits altos de `dout_high` | top | Forzar multiplicaciones con operandos grandes (`op = 010`, `din` cercanos a 255) para que los 8 bits altos conmuten |
| `nvalid_data = 1` | control/top | Ya forzado con `run_instruction(7'b11_00_000, 1)`; verificar que también ocurre con `cmd_in[4:3] == 2'b11` |
| Condición `memoryRead = 0` con dirección variada | memory | El tb ya alterna read/write en 8 direcciones; el toggle de bits altos de `memoryAddress` no es alcanzable en top (solo se usan [2:0]) — candidato a exclusión justificada |

Regla del curso: primero estímulo aleatorio en volumen (`repeat(n)` con `$urandom_range`), después casos forzados quirúrgicos para lo que el azar no alcanza. Tras cada cambio de testbench: limpiar (`rm -rf csrc/ simv* novas*`), recompilar, resimular, reabrir coverage.

## Errores comunes

| Problema | Causa | Solución |
|---|---|---|
| Porcentajes no cambian tras editar el tb | `simv.vdb` viejo acumulando datos | Borrar `simv.vdb` y regenerar |
| Columna FSM vacía | El módulo no tiene FSM o no se pidió `fsm` en `-cm` | Es normal en módulos combinacionales |
| Assertion coverage en 0 o ausente | Falta `+assert` en ambos `-cm`, o no hay SVA | Añadir `+assert` a compilación y simulación |
| Coverage con métricas distintas entre compilar y simular | Flags `-cm` diferentes en cada fase | Usar exactamente la misma lista |

## Checklist final del capítulo 6

- [ ] Compilaste y simulaste con la misma lista `-cm line+cond+fsm+tgl+branch`
- [ ] `simv.vdb` se borra antes de cada corrida limpia
- [ ] Navegaste Summary → CovSrc → CovDetail → línea RTL
- [ ] Identificaste al menos un hueco rojo y lo cerraste con un estímulo nuevo
- [ ] La FSM de control reporta todos sus estados y transiciones alcanzables

---

# Capítulo 7 — Proyecto UVM (ALU)

## Objetivo

Compilar, ejecutar y entender el ambiente UVM real del proyecto `uvm_alu`, que verifica la misma ALU del CPU con la metodología UVM.

## Explicación conceptual

UVM es una metodología estandarizada para crear ambientes de verificación reutilizables usando SystemVerilog orientado a objetos. Es una librería (clases, reglas y una arquitectura) incluida en VCS. La diferencia esencial con el testbench clásico:

| Aspecto | Sin UVM | Con UVM |
|---|---|---|
| Conexión de señales | Directa en el testbench | A través de `interface` y `virtual interface` |
| Organización del código | Un archivo `.sv` | Package con clases separadas por responsabilidad |
| Selección del test | Recompilar para cambiar test | `+UVM_TESTNAME=<nombre>` en runtime |
| Reutilización | Baja | Alta (factory, overrides) |
| Mensajes de debug | `$display`/`$monitor` | Macros UVM con niveles de verbosidad |

## Estructura del proyecto real

```
uvm_alu/
├── rtl/
│   └── ALU.v
└── tb/
    ├── alu_if.sv
    ├── alu_pkg.sv        (incluye con `include los archivos de clases)
    ├── alu_seq_item.sv
    ├── alu_sequence.sv
    ├── alu_driver.sv
    ├── alu_monitor.sv
    ├── alu_agent.sv
    ├── alu_env.sv
    ├── alu_test.sv
    └── tb_top.sv
```

Orden de definición dentro del package: `alu_seq_item → alu_sequence → alu_driver → alu_monitor → alu_agent → alu_env → alu_test`.

## Componentes, uno por uno

### Interface — tb/alu_if.sv

Reemplaza las conexiones directas entre testbench y DUT; agrupa todas las señales:

```systemverilog
interface alu_if #(parameter WIDTH = 8);
    logic [WIDTH-1:0]   in1, in2;
    logic [3:0]         op;
    logic               invalid_data;
    logic [2*WIDTH-1:0] out;
    logic               zero;
    logic               error;
endinterface
```

### Package — tb/alu_pkg.sv

Contiene e incluye todos los archivos de clases UVM:

```systemverilog
package alu_pkg;
    import uvm_pkg::*;
    `include "uvm_macros.svh"
    `include "alu_seq_item.sv"
    `include "alu_sequence.sv"
    `include "alu_driver.sv"
    `include "alu_monitor.sv"
    `include "alu_agent.sv"
    `include "alu_env.sv"
    `include "alu_test.sv"
endpackage
```

Como los `include` son relativos al directorio `tb/`, la compilación necesita `+incdir+tb/` (ver más abajo). Omitirlo produce errores SFCOR de archivos no encontrados.

### Transaction (sequence item) — tb/alu_seq_item.sv

Representa una transacción: los valores que el driver va a aplicar al DUT en un ciclo:

```systemverilog
class alu_seq_item extends uvm_sequence_item;
    `uvm_object_utils(alu_seq_item)
    rand logic [7:0] in1, in2;
    rand logic [3:0] op;
    rand logic       invalid_data;
    logic [15:0] out;
    logic        zero, error;
    function new(string name = "alu_seq_item");
        super.new(name);
    endfunction
endclass
```

`rand` marca los campos randomizables con `randomize()`. Los campos de salida (`out`, `zero`, `error`) no son `rand` porque los escribe el DUT, no el driver; el monitor los captura.

### Sequences — tb/alu_sequence.sv

Generan y envían transacciones al sequencer. Equivalen a las tasks del testbench clásico:

- `alu_rand_seq`: `repeat(40)` de items totalmente aleatorios (equivalente de la task de test aleatorio).
- `alu_div0_seq`: `repeat(5)` con constraint `op == 4'b0011; in2 == 8'h00; invalid_data == 1'b0;` (división por cero).
- `alu_invalid_seq`: `repeat(5)` con constraint `invalid_data == 1'b1;`.

Patrón de cada body: `create` → `start_item` → `randomize() with {...}` (con `uvm_fatal` si falla) → `finish_item`.

### Driver — tb/alu_driver.sv

Toma transacciones del sequencer y las aplica al DUT a través de la virtual interface:

- En `build_phase` recupera la interface: `uvm_config_db#(virtual alu_if)::get(this, "", "vif", vif)`; si no existe falla con `uvm_fatal`.
- En `run_phase`, en un `forever`: `seq_item_port.get_next_item(item)` → volcar campos a `vif` → `#10` (tiempo para que el DUT combinacional procese) → `seq_item_port.item_done()`. `get_next_item`/`item_done` es el protocolo TLM del driver: siempre en par.

### Monitor — tb/alu_monitor.sv

Observa pasivamente las señales del DUT cada `#10`, arma un item con entradas y salidas, lo publica por `uvm_analysis_port` (`ap.write(item)`) hacia scoreboard/coverage, e imprime con `` `uvm_info("MON", $sformatf(...), UVM_MEDIUM) ``.

### Agent — tb/alu_agent.sv

Encapsula sequencer, driver y monitor. En `build_phase` los crea con la factory (`type_id::create`); en `connect_phase` conecta `drv.seq_item_port.connect(sqr.seq_item_export)` — sin esto el driver nunca recibe transacciones.

### Environment — tb/alu_env.sv

Encapsula el agente. En proyectos más grandes también contiene scoreboard y coverage.

### Tests — tb/alu_test.sv

- `alu_base_test`: crea el env y en `run_phase` lanza `alu_rand_seq` sobre `env.agt.sqr`, entre `phase.raise_objection(this)` y `phase.drop_objection(this)`.
- `alu_div0_test` y `alu_invalid_test`: extienden el base y lanzan su secuencia específica.

Sin `raise_objection` la simulación termina inmediatamente; sin `drop_objection` nunca termina.

### Top — tb/tb_top.sv

Instancia el DUT y la interface, registra la interface en la config_db y arranca UVM:

```systemverilog
uvm_config_db #(virtual alu_if)::set(null, "uvm_test_top.*", "vif", dut_if);
run_test();
```

`"uvm_test_top.*"` es el path de componentes que tendrán acceso a la interface. `run_test()` sin argumento hace que UVM use el valor de `+UVM_TESTNAME` del runtime.

Jerarquía UVM resultante:

```
uvm_root
  └── uvm_test_top  (alu_base_test)
        └── env     (alu_env)
              └── agt  (alu_agent)
                    ├── sqr  (uvm_sequencer)
                    ├── drv  (alu_driver)
                    └── mon  (alu_monitor)
```

## Compilación

```bash
vcs -sverilog -ntb_opts uvm-1.2 -full64 -kdb -debug_access+all \
    +incdir+tb/ \
    tb/alu_if.sv \
    tb/alu_pkg.sv \
    rtl/ALU.v \
    tb/tb_top.sv \
    -l compile.log
```

| Elemento nuevo | Explicación |
|---|---|
| `-ntb_opts uvm-1.2` | Native TestBench Options: habilita la librería UVM versión 1.2 incluida en VCS |
| `+incdir+tb/` | Directorio de búsqueda para los `` `include `` del package. Necesario para que VCS resuelva los include hacia `tb/`; omitirlo causa errores SFCOR |
| Orden de archivos | Importa: la interface primero, luego el package, luego el RTL, finalmente el módulo top |
| `-l compile.log` | Guarda todos los mensajes de compilación |

## Simulación

```bash
./simv +UVM_TESTNAME=alu_base_test -l run.log
```

`+UVM_TESTNAME=<nombre>` es un argumento de runtime que indica qué clase de test ejecutar; UVM la busca en la factory por ese nombre. Cambiar de test **sin recompilar**:

```bash
./simv +UVM_TESTNAME=alu_div0_test -l run.log
./simv +UVM_TESTNAME=alu_invalid_test -l run.log
```

Verbosidad de mensajes:

```bash
./simv +UVM_TESTNAME=alu_base_test +UVM_VERBOSITY=UVM_HIGH -l run.log
```

| Nivel | Qué muestra |
|---|---|
| `UVM_LOW` | Solo mensajes críticos de regresión |
| `UVM_MEDIUM` | Nivel por defecto |
| `UVM_HIGH` | Mensajes de tracing de ejecución |
| `UVM_FULL` | Debug completo |

Macros de reporte que verás en los logs:

| Macro | Uso |
|---|---|
| `` `uvm_fatal("ID","msg") `` | Error fatal, detiene la simulación |
| `` `uvm_error("ID","msg") `` | Error, continúa pero incrementa error count |
| `` `uvm_warning("ID","msg") `` | Advertencia |
| `` `uvm_info("ID","msg",UVM_MEDIUM) `` | Información, filtrada por verbosidad |

## Con coverage

```bash
vcs -sverilog -ntb_opts uvm-1.2 -full64 -kdb -debug_access+all \
    -cm line+cond+fsm+tgl+branch \
    +incdir+tb/ \
    tb/alu_if.sv tb/alu_pkg.sv rtl/ALU.v tb/tb_top.sv \
    -l compile.log

./simv +UVM_TESTNAME=alu_base_test -cm line+cond+fsm+tgl+branch -l run.log
verdi -cov -covdir ./simv.vdb/ &
```

## Debug y verificación de la corrida

1. `compile.log` sin errores; la cabecera confirma UVM-1.2 cargada.
2. `run.log` empieza con el banner UVM, muestra los `uvm_info("MON", ...)` con `op/in1/in2/out/zero/error` por transacción.
3. El final del log trae el **UVM Report Summary**: `UVM_ERROR : 0` y `UVM_FATAL : 0` para una corrida limpia.
4. Con `alu_div0_test`, cada línea del monitor debe mostrar `error=1` y `out=65535`.
5. Si la simulación termina al instante sin transacciones: falta `raise_objection`. Si nunca termina: falta `drop_objection`. Si `uvm_fatal CFG "No se encontro virtual interface"`: el `set` de la config_db del top no coincide en path/clave con el `get`.

## Limpiar

```bash
rm -rf csrc/ simv* novas* *.log
```

## Buenas prácticas del proyecto

- Un archivo por clase, incluidos vía package con `+incdir+tb/`.
- Las secuencias dirigidas (`div0`, `invalid`) replican los corner cases del testbench clásico: la metodología cambia, los estímulos críticos no.
- Correr siempre con `-l run.log` y revisar el UVM Report Summary antes de dar por buena una corrida.
- Usar `+UVM_TESTNAME` para regresiones (varios tests con un solo binario).

## Checklist final del capítulo 7

- [ ] El proyecto compila con `-ntb_opts uvm-1.2` y `+incdir+tb/`
- [ ] Los tres tests corren cambiando solo `+UVM_TESTNAME`
- [ ] Los UVM Report Summary terminan con 0 errores/fatales
- [ ] Sabes trazar el camino de una transacción: sequence → sequencer → driver → DUT → monitor → analysis_port

---

# Capítulo 8 — Design Compiler y Design Vision

## Objetivo

Construir el flujo de síntesis progresivamente en `dc_shell`, comando por comando, hasta llegar a un `dc.tcl` completo y reproducible, y después recorrer a fondo la GUI (Design Vision).

## Explicación conceptual

Design Compiler transforma el RTL (descripción de comportamiento) en un netlist de compuertas (descripción estructural) mapeado a una librería de tecnología — en el curso, celdas estándar Sky130. El flujo tiene siete etapas fijas:

1. Search Path and Logic Library Setup
2. RTL Reading and Link
3. Constraints Setup
4. Pre-compile Reports
5. Compile/Synthesis
6. Post-compile Reports
7. Save Design

El diseño base de los labs es el `top` con el mux4 registrado (`top.v`, `mux4_registered.v`, `mux4.v`, `register_bank.v`); el mismo flujo se recicla para el CPU cambiando la lista de RTL en `analyze` y el nombre en `elaborate` (por ejemplo `analyze -format sverilog {top.sv control.sv ALU.sv memory.sv mux4.sv register_bank.sv}` y `elaborate top -parameters "WIDTH=8"`).

Formas de arrancar DC:

```bash
dc_shell                                        # shell interactivo
dc_shell -f scripts/dc.tcl                      # batch: fuente el script al arrancar
dcnxt_shell -f scripts/dc.tcl                   # variante NXT usada en el curso
dc_shell -f ./scripts/dc.tcl | tee -i ./logs/dc.log   # batch guardando el log
```

Dentro del shell interactivo también puedes fuentear el script:

```
dc_shell> source -echo -verbose scripts/dc.tcl
```

`-echo -verbose` imprime cada comando del script y su resultado — lee ese log: cada comando se "ecoa" y muestra su efecto. Truco de desarrollo: un `return` en cualquier punto del `dc.tcl` detiene el batch ahí y te devuelve al shell interactivo.

Ayuda dentro del shell:

```
dc_shell> help report*
dc_shell> help get*
dc_shell> man report_timing
dc_shell> man compile_ultra
```

## 8.1 Setup de librerías (construcción progresiva)

```tcl
set_app_var search_path "$search_path . ./rtl ./libs"
set_app_var target_library "sky130_fd_sc_hd__ff_100C_1v95.db sky130_fd_sc_hd__ss_100C_1v40.db"
set_app_var link_library "* $target_library"
get_libs
```

Línea por línea:

- `search_path`: lista de directorios donde DC busca archivos (RTL, librerías). Se **añade** `. ./rtl ./libs` a lo que ya había (`$search_path`), no se reemplaza.
- `target_library`: las librerías de tecnología a las que se mapeará el diseño. Aquí hay dos corners de Sky130: `ff_100C_1v95` (fast-fast, 1.95V) y `ss_100C_1v40` (slow-slow, 1.40V). Físicamente: DC evalúa las celdas en ambos extremos de proceso/voltaje.
- `link_library`: dónde resolver referencias al enlazar. El `*` significa "primero los diseños ya en memoria", después las target libraries.
- `get_libs`: verifica que las `.db` se cargaron; debe listar ambas librerías.

## 8.2 analyze

```tcl
analyze -format verilog {top.v mux4_registered.v mux4.v register_bank.v}
```

`analyze` lee y chequea sintácticamente el RTL, dejándolo en una forma intermedia. `-format verilog` indica el lenguaje (para el CPU en SystemVerilog: `-format sverilog`). La lista va entre llaves. Verificación: cada archivo debe reportar "successfully" y ningún `Error`.

## 8.3 elaborate

```tcl
elaborate top -parameters "WIDTH=3"
```

`elaborate` construye el diseño a partir de lo analizado: resuelve parámetros, expande instancias y genera un netlist **genérico sin mapear** (GTECH). `-parameters "WIDTH=3"` fija el valor del parámetro del top (los labs usan WIDTH=3; la variante final usa WIDTH=16). Verificación: DC reporta el diseño `top` elaborado y los templates instanciados; `get_designs` lo lista.

## 8.4 link

```tcl
current_design top
link
```

`current_design` fija el diseño de trabajo; `link` resuelve todas las referencias (celdas, subdiseños) contra la `link_library`. Verificación: `link` devuelve 1 y no reporta unresolved references.

Exploración útil en este punto (LAB0):

```
dc_shell> get_designs
dc_shell> get_cells
dc_shell> report_cell [get_cells -hier]
dc_shell> report_collection [get_cells] -columns {full_name ref_name is_sequential area}
dc_shell> get_ports
dc_shell> all_inputs
dc_shell> report_lib <lib_name>
```

Guardar un reporte y abrirlo sin salir del shell:

```
dc_shell> report_cell > reports/cells.rpt
dc_shell> sh gedit reports/cells.rpt
```

El netlist genérico puede escribirse para compararlo luego con el mapeado:

```
dc_shell> write_file -format verilog -hier -out unmapped.v
```

## 8.5 Constraints

Valores del curso (LAB1), todos como función del periodo de reloj:

| Constraint | Valor |
|---|---|
| Clock period | 10 ns |
| Clock setup uncertainty | 10% |
| Clock transition | 10% |
| Clock source latency | 5% |
| Clock network latency | 3% |
| Input delay | 40% |
| Output delay | 50% |
| Output load | 0.04 pF |
| Input min transition | 1% |
| Input max transition | 10% |

No se aplica input delay ni transition a los puertos de reloj (de ahí el `remove_from_collection`).

```tcl
set clk_val 10

create_clock -period $clk_val [get_ports clk] -name clk
set_clock_uncertainty -setup [expr $clk_val*0.1] [get_clocks clk]
set_clock_transition  -max   [expr $clk_val*0.1] [get_clocks clk]
set_clock_latency -source -max [expr $clk_val*0.05] [get_clocks clk]
set_clock_latency         -max [expr $clk_val*0.03] [get_clocks clk]

set_input_delay  -max [expr $clk_val*0.4] -clock clk [get_ports [remove_from_collection [all_inputs] clk]]
set_output_delay -max [expr $clk_val*0.5] -clock clk [get_ports [all_outputs]]

set_load -max 0.04 [all_outputs]
set_input_transition -min [expr $clk_val*0.01] [remove_from_collection [all_inputs] clk]
set_input_transition -max [expr $clk_val*0.1]  [remove_from_collection [all_inputs] clk]
```

Qué representa físicamente cada restricción:

- `create_clock`: define el reloj ideal de periodo `clk_val` sobre el puerto `clk`. Todo el análisis de timing se mide contra este periodo.
- `set_clock_uncertainty -setup`: margen de incertidumbre (jitter + skew estimado) que se resta del periodo disponible en los chequeos de setup. 10% del periodo = 1 ns de colchón.
- `set_clock_transition`: pendiente (slew) asumida en los flancos del reloj: relojes lentos degradan el timing de los flops.
- `set_clock_latency -source`: retardo desde la fuente real del reloj (PLL/oscilador) hasta el pin de entrada del chip.
- `set_clock_latency` (sin `-source`): latencia estimada de la red de reloj interna (el clock tree que aún no existe en síntesis).
- `set_input_delay`: cuánto tarda la lógica externa en entregar los datos después del flanco; deja a DC solo el 60% del periodo para la lógica de entrada.
- `set_output_delay`: cuánto tiempo necesita el receptor externo; DC debe entregar las salidas dejando libre el 50% del periodo.
- `set_load`: capacitancia (pF) que cada salida debe manejar; cargas mayores fuerzan celdas de salida más potentes.
- `set_input_transition`: qué tan limpios llegan los flancos de las entradas.

Verificación: cada comando de constraint debe devolver `1`. Un `0` o un error significa que la constraint no se aplicó.

## 8.6 Reportes pre-compile

Reportan sobre el netlist GTECH ya restringido. Se guardan con `>`:

```tcl
report_clock                 > ./reports/pre_syn_report_clock.rpt
report_clock -skew           > ./reports/pre_syn_report_clock_skew.rpt
report_port -verbose         > ./reports/pre_syn_report_port_constraints.rpt
check_timing                 > ./reports/pre_syn_check_timing.rpt
check_design                 > ./reports/pre_syn_check_design.rpt
```

- `report_clock`: periodo y fuente del reloj. `-skew`: uncertainty, transition, latency.
- `report_port -verbose`: input/output delays, output load, input transitions — confirma que las constraints de puertos quedaron.
- `check_timing`: problemas potenciales de timing (endpoints sin constraint, etc.).
- `check_design`: problemas del diseño (puertos sin conectar, celdas sin driver...).
- `write_sdc constraints.sdc` exporta las constraints en formato estándar SDC.

## 8.7 compile / compile_ultra

```tcl
compile_ultra -no_autoungroup
```

`compile_ultra` es el motor de síntesis y optimización de alto esfuerzo de DC (incluye las optimizaciones de `compile` clásico más técnicas avanzadas de timing, área y estructura, de forma automática). Lee el log de compilación en pantalla: muestra el proceso de minimización de costos y las pasadas de optimización.

Variantes trabajadas en el curso y cuándo usar cada una:

| Comando | Qué hace | Cuándo usarla |
|---|---|---|
| `compile` | Síntesis clásica, esfuerzo estándar | Referencia histórica; hoy se prefiere `compile_ultra` |
| `compile_ultra` | Alto esfuerzo, con auto-ungroup (aplana jerarquías para optimizar cruzando fronteras) | Resultado por defecto para cerrar timing/área |
| `compile_ultra -no_autoungroup` | Desactiva el auto-ungrouping, preservando las jerarquías del RTL | Cuando necesitas conservar la jerarquía para debug, ECOs, reportes por módulo o flujos posteriores |
| `compile_ultra -no_boundary_optimization` | No optimiza a través de las fronteras de los módulos (no propaga constantes ni simplifica puertos entre bloques) | Cuando los bloques deben mantenerse intactos (IPs, bloques que se reutilizan por separado) |
| `compile_ultra -gate_clock` | Inserta clock gating: agrega celdas que apagan el reloj de los registros cuando no cargan dato nuevo | Reducir potencia dinámica del clock tree en diseños con enables |
| `compile_ultra -retime` | Retiming: mueve registros a través de la lógica combinacional para balancear los caminos | Cerrar timing en pipelines desbalanceados |
| `compile_ultra -incremental` | No resintetiza desde cero: optimiza sobre el netlist ya mapeado | Después de cambios de constraints o de pasos como `insert_dft`, para recuperar QoR sin perder lo hecho |

Optimizaciones posteriores al compile (LAB2):

```tcl
# Contar flip-flops antes y después
set ff_before [sizeof_collection [all_registers]]
optimize_registers
set ff_after  [sizeof_collection [all_registers]]
puts "Antes: $ff_before Despues: $ff_after"
```

`optimize_registers` reubica/optimiza registros sobre el diseño ya compilado. En el diseño del lab el número de FFs no cambió: el diseño es simple y ya estaba optimizado — resultado real observado en el curso.

```tcl
optimize_netlist -area
```

Optimización de área sobre el netlist mapeado. En las corridas reales del curso produjo mejora de power (ver capítulo 9, diffs "before/after areaopt").

```tcl
change_names -rule verilog
```

Renombra objetos internos a nombres legales de Verilog antes de escribir el netlist final — evita netlists con nombres ilegales que rompen herramientas aguas abajo.

## 8.8 Reportes post-compile

```tcl
report_timing                     > $report_dir/report_timing.rpt
report_qor                        > $report_dir/report_qor.rpt
report_constraints -all_violators > $report_dir/report_constraints.rpt
report_power                      > $report_dir/report_power.rpt
check_design                      > $report_dir/check_design.rpt
report_area                       > $report_dir/report_area.rpt
```

Otros del curso: `report_area -hierarchy`, `report_timing -max_paths 10`, `report_cell`, `report_resources`, `report_constraint -all`.

Qué mirar en cada uno:

- `report_timing`: el camino crítico, con su slack. Slack positivo = cumple; `(VIOLATED)` = no.
- `report_qor`: resumen de calidad — critical path length/slack, cell counts, áreas, tiempos de compilación.
- `report_constraints -all_violators`: solo las violaciones (timing y design rules). Ideal para el veredicto rápido.
- `report_power`: potencia dinámica (internal + switching) y leakage, desglosada en clock_network, register, combinational.
- `report_area`: áreas combinacional, no-combinacional, buf/inv y total.
- `check_design`: warnings estructurales (ej. LINT-28 puertos sin conectar).

## 8.9 Guardar el diseño

```tcl
write_file -format ddc     -hierarchy -out $output_dir/mapped.ddc
write_file -format verilog -hierarchy -out $output_dir/mapped.v
write_sdc $output_dir/mapped.sdc
```

DDC es el formato binario nativo de DC (diseño + constraints, recargable con `read_ddc`); Verilog+SDC es el par portable para otras herramientas. Compara en Linux `unmapped.v` contra `mapped.v`: el primero es genérico GTECH, el segundo instancia celdas reales `sky130_fd_sc_hd__*`.

## 8.10 Organización por run_tag (script real del curso)

Para no sobrescribir corridas, el `dc.tcl` del curso parametriza los directorios con un tag al inicio:

```tcl
set run_tag "Sin_no_autoungroup"
;# ejemplos: no_autoungroup / no_boundary_opt / gate_clock / retime

set report_dir "./reports/$run_tag"
set output_dir "./outputs/$run_tag"
file mkdir $report_dir
file mkdir $output_dir
```

Todos los `report_* > $report_dir/...` y `write_file ... $output_dir/...` caen entonces en:

```
reports/compile_ultra/...   outputs/compile_ultra/...
reports/retime/...          outputs/retime/...
reports/gate_clock/...      outputs/gate_clock/...
```

Cambias `run_tag`, cambias la variante de `compile_ultra`, relanzas, y las corridas anteriores quedan intactas. Cierre del script:

```tcl
puts "Corrida '$run_tag' terminada. Reportes en $report_dir, outputs en $output_dir"
gui_start
```

## 8.11 Design Vision — la GUI

Se abre desde el shell con:

```
dc_shell> start_gui
```

o desde el script con `gui_start` (ambos lanzan Design Vision sobre el diseño en memoria; el `dc.tcl` del curso usa `gui_start` al final).

### Logical Hierarchy / Design Browser

Panel izquierdo "Logical Hierarchy": muestra el diseño `top` y sus subdiseños. Clic derecho sobre `top` → **Schematic View** abre el esquemático del nivel; doble clic sobre un símbolo baja al interior. Antes de compilar verás el netlist GTECH genérico; después de `compile_ultra`, celdas Sky130 reales — compara ambos, es el ejercicio del LAB0/LAB1.

### Schematic

El esquemático es navegable: doble clic desciende jerarquía, clic derecho sobre una celda muestra sus propiedades (ref_name, área, si es secuencial). Con `-no_autoungroup` verás las jerarquías del RTL preservadas (mux4, register_bank); sin él, un mar plano de compuertas.

### Timing Paths / Critical Path / Slack

Menú Timing → los análisis de caminos. El análisis de paths muestra los caminos ordenados por slack; el peor es el **critical path**. Seleccionar un path lo resalta en el esquemático (highlight), mostrando físicamente por qué compuertas pasa. El slack que ves aquí es el mismo `Critical Path Slack` de `report_qor` (ej. 3.55 en la corrida retime real del curso). Slack negativo aparece como VIOLATED.

### Constraint Browser / Clock Browser

- Constraint Browser: lista las constraints aplicadas y su estado de violación — la versión gráfica de `report_constraints -all_violators`.
- Clock Browser: los relojes definidos con periodo, uncertainty, latencias — la versión gráfica de `report_clock`/`-skew`.

### Net / Cell Browser, fanout y loads

Los browsers de nets y cells listan todos los objetos con columnas ordenables (área, fanout). Para inspeccionar **fanout** y **loads**: selecciona una net o pin (por ejemplo la salida del registro de comando), clic derecho → las opciones de conectividad muestran drivers y loads; en el esquemático puedes expandir la net para ver a qué entradas llega. Esto responde gráficamente lo mismo que los trace de Verdi pero sobre el netlist de compuertas.

### Área, power y reportes desde la GUI

Los menús Design/Timing de Design Vision exponen los mismos reportes del shell (`report_area`, `report_power`, `report_qor`, `report_timing`, `report_constraints`): cada uno abre en una ventana de texto dentro de la GUI, con los mismos números que las versiones `>` a archivo. Regla práctica del curso: los números oficiales se guardan por script en `reports/$run_tag/`; la GUI se usa para **entender** (ver el critical path resaltado, inspeccionar celdas, comparar esquemáticos entre corridas).

Recorrido mínimo obligatorio tras cada `compile_ultra`:

1. `start_gui`.
2. Schematic View del top: confirmar mapeo a celdas Sky130 y estado de la jerarquía.
3. Timing paths: localizar y resaltar el critical path, leer su slack.
4. Constraint Browser: confirmar qué está VIOLATED (en las corridas reales del curso: el max_leakage_power del diseño).
5. Abrir report_area y report_qor desde la GUI y contrastar con los `.rpt` guardados.

## Errores comunes del capítulo

| Problema | Causa | Solución |
|---|---|---|
| `unresolved references` en link | `.db` no encontradas o `search_path` sin `./libs` | Verificar `get_libs` y `search_path` |
| Constraint devuelve 0 | Puerto/reloj inexistente (typo) o reloj no creado aún | `create_clock` primero; verificar con `get_ports` |
| Reportes de dos corridas mezclados | Sin `run_tag`, se sobrescriben | Usar la estructura `reports/$run_tag` |
| El esquemático no muestra jerarquía | `compile_ultra` sin `-no_autoungroup` la aplanó | Decidir según el objetivo; recompilar si se necesita jerarquía |
| GUI no abre | Sesión sin X11/forwarding | Verificar el display; usar reportes de texto mientras |

## Checklist final del capítulo 8

- [ ] Construiste el flujo interactivo: setup → analyze → elaborate → link → constraints → reportes → compile_ultra → reportes → save
- [ ] Cada constraint devolvió 1 y `report_port -verbose` las confirma
- [ ] Tienes `mapped.v`, `mapped.ddc` y `mapped.sdc` bajo `outputs/$run_tag/`
- [ ] Recorriste Design Vision: jerarquía, esquemático, critical path resaltado, constraint browser, reportes
- [ ] Sabes explicar cuándo usar cada variante de `compile_ultra`

---

# Capítulo 9 — Comparación de síntesis: compare_reports.sh

## Objetivo

Entender línea por línea el script `comparador.sh` (compare_reports.sh), usarlo para comparar las corridas de `compile_ultra` y aprender a decidir cuál corrida es mejor a partir de los reportes reales del curso.

## Prerrequisito

Cada corrida de síntesis debe haberse ejecutado con su `run_tag` (capítulo 8.10), dejando la estructura:

```
reports/
├── incremental/
├── no_autoungroup/
├── no_autoungroup-gate_clock/
├── retime/
├── Sin_no_autoungroup/
└── width16_clk45/
```

El script se ejecuta **dentro** del directorio `reports/`:

```bash
chmod +x comparador.sh
./comparador.sh
```

`chmod +x` da permiso de ejecución; solo hace falta la primera vez.

## Análisis línea por línea del script

### Cabecera y setup

```bash
#!/bin/bash
OUT="comparador.md"
EASTER_EGG_IMG="reze.jpg"

> "$OUT"
```

- `#!/bin/bash`: shebang; el script corre con bash.
- `OUT`: nombre del markdown de salida.
- `> "$OUT"`: trunca (vacía) el archivo de salida antes de empezar. Todo lo demás se escribe con `>>` (append).

```bash
echo "# Comparacion de reportes de sintesis" >> "$OUT"
echo "" >> "$OUT"
echo "Generado: $(date)" >> "$OUT"
```

Escribe el título y la fecha de generación (`$(date)` es sustitución de comando).

```bash
TAGS=$(find . -maxdepth 1 -mindepth 1 -type d | sed 's|./||' | sort)
```

Descubre automáticamente las corridas: `find` lista solo los directorios inmediatos (`-maxdepth 1 -mindepth 1 -type d`), `sed 's|./||'` quita el prefijo `./` y `sort` los ordena. Cada directorio es un `run_tag`. Ventaja: agregar una corrida nueva no requiere tocar el script.

### Funciones de extracción

Cada función recibe la ruta de un `.rpt` como `$1` y extrae una métrica con el patrón `grep | head/tail | awk | xargs`:

```bash
get_total_area() {
    grep -i "Total cell area" "$1" 2>/dev/null | head -1 | awk -F: '{print $2}' | xargs
}
```

- `grep -i`: busca la línea sin distinguir mayúsculas. `2>/dev/null` silencia el error si el archivo no existe.
- `head -1`: se queda con la primera coincidencia.
- `awk -F: '{print $2}'`: divide por `:` y toma el valor a la derecha.
- `xargs`: truco para recortar espacios en blanco sobrantes.

Las demás siguen el mismo patrón con variantes:

- `get_leakage_power`: busca "Cell Leakage Power", usa `tail -1` (última coincidencia, la del total) y `awk '{print $NF, $(NF-1)}'` toma los dos últimos campos — el valor y su unidad. Nota: al imprimir `$NF` antes de `$(NF-1)` la unidad sale antes del número ("nW 263.7173"); es cosmético, no afecta la comparación (mejora propuesta abajo).
- `get_critical_slack`: "critical path slack" de `report_qor.rpt`.
- `get_cell_count`: "leaf cell count" de `report_qor.rpt`.
- `get_design_rules`: `grep -i -A5 "^Design Rules"` toma la sección Design Rules y 5 líneas después, y filtra las que contienen "violat".
- `get_constraints_violations`: si el reporte contiene "No timing violations" devuelve "0 (sin violaciones)"; si no, `grep -ci "VIOLATED"` cuenta las líneas con VIOLATED.
- `get_check_design_status`: cuenta líneas que empiezan con `Error` y con `Warning` (`grep -ci "^Error"` / `^Warning`) y las imprime como "Errors: N, Warnings: M".

### Sección 1: tabla resumen

```bash
for tag in $TAGS; do
    area=$(get_total_area "$tag/report_area.rpt")
    leak=$(get_leakage_power "$tag/report_power.rpt")
    slack=$(get_critical_slack "$tag/report_qor.rpt")
    cells=$(get_cell_count "$tag/report_qor.rpt")
    cviol=$(get_constraints_violations "$tag/report_constraints.rpt")
    echo "| $tag | ${area:-N/A} | ${leak:-N/A} | ${slack:-N/A} | ${cells:-N/A} | ${cviol:-N/A} |" >> "$OUT"
done
```

Recorre cada tag, extrae las cinco métricas de sus reportes y emite una fila de tabla Markdown. `${var:-N/A}` imprime "N/A" si la métrica no se pudo extraer (reporte ausente) — el script nunca revienta por reportes faltantes.

### Sección 2: Design Rules / Check Design

Por cada tag emite un subtítulo `### $tag` con las violaciones de design rules del QoR y el conteo de errores/warnings del `check_design.rpt`.

### Sección 3: diffs internos (before/after de la misma corrida)

```bash
diff_block() {
    local title="$1" f1="$2" f2="$3"
    if [[ -f "$f1" && -f "$f2" ]]; then
        echo "#### $title" >> "$OUT"
        echo '```diff' >> "$OUT"
        diff "$f1" "$f2" >> "$OUT"
        echo '```' >> "$OUT"
        echo "" >> "$OUT"
    fi
}
```

`diff_block` es la función central: si **ambos** archivos existen (`[[ -f ... && -f ... ]]`), escribe un bloque ```` ```diff ```` con la salida de `diff` (las líneas `<` son del primer archivo, `>` del segundo) — Markdown lo colorea. Si falta alguno, no escribe nada: por eso solo aparecen los diffs de corridas que realmente generaron esos reportes.

El bucle compara, dentro de cada tag: area/timing/qor/constraints/check_design **before vs after areaopt** (efecto de `optimize_netlist -area`) y area/timing **base vs postopt** (efecto de `optimize_registers`).

### Sección 4: diffs cruzados vs baseline

```bash
BASELINE="retime"
for tag in $TAGS; do
    [[ "$tag" == "$BASELINE" ]] && continue
    for repfile in report_area.rpt report_power.rpt report_qor.rpt report_constraints.rpt check_design.rpt; do
        diff_block "$repfile : $BASELINE vs $tag" "$BASELINE/$repfile" "$tag/$repfile"
    done
done
```

Fija `retime` como corrida de referencia y hace diff de cada reporte principal de las demás corridas contra ella. `continue` salta el propio baseline.

### Sección final

Inserta la imagen `reze.jpg` si existe en `reports/`; si no, deja el recordatorio. Termina imprimiendo dónde quedó el resultado.

## Resultados reales del curso (1 de julio de 2026)

Tabla resumen real generada por el script (diseño top del mux registrado, WIDTH=3, clk=10ns; las dos últimas filas son WIDTH=16, clk=45ns):

| Tag | Area | Leakage Power | Critical Slack | Cells | Constr. Viol. |
|---|---|---|---|---|---|
| incremental | 377.86 | 420.76 nW | 3.48 | 42 | 1 |
| no_autoungroup | 315.30 | 334.14 nW | 3.55 | 34 | 1 |
| no_autoungroup-gate_clock | 324.06 | 363.69 nW | 3.56 | 35 | 1 |
| retime | 276.52 | 263.72 nW | 3.55 | 43 | 1 |
| Sin_no_autoungroup (W=16, 45ns) | 7714.90 | 8.63 uW | 6.86 | 1206 | 1 |
| width16_clk45 | 7714.90 | 8.63 uW | 6.86 | 1206 | 1 |

## Cómo interpretar cada métrica

- **Area (Total cell area)**: suma de área de celdas (combinacional + secuencial + buf/inv). Menor es mejor a igualdad de lo demás. En los datos reales, retime logró la menor área (276.52) aplanando todo (0 celdas jerárquicas en su QoR) y con solo 3 celdas secuenciales.
- **Leakage Power**: potencia estática. Correlaciona fuerte con el área/tamaño de celdas: retime también gana aquí (263.7 nW). Ojo con las unidades al comparar: nW vs uW (las corridas W=16 están en uW — mil veces más).
- **Critical Slack**: margen del peor camino contra el periodo. Positivo = timing cumplido; cuanto más positivo, más margen (o señal de que el reloj es holgado). Todas las corridas W=3 rondan 3.5 ns de slack sobre 10 ns: timing cómodo, así que el criterio de decisión pasa a área/power.
- **QoR**: el `report_qor` completo — además del slack trae levels of logic (retime: 0.00 con path de 0.45 ns; incremental: 12 niveles y 1.51 ns), cell counts y tiempos de compilación.
- **Timing (report_timing)**: el detalle del camino crítico celda a celda. Úsalo cuando el slack sorprenda.
- **Constraints**: en todas las corridas reales hay exactamente 1 violación: `max_leakage_power` del diseño (`top_WIDTH3 ... (VIOLATED)`) — una constraint de potencia de la librería/setup, no de timing. Los diffs muestran cómo varía la magnitud: retime -249.34 vs incremental -417.17 (retime viola menos).
- **Check Design**: retime, no_autoungroup y gate_clock salen limpios (0 warnings); incremental trae 4 warnings LINT-28 — puertos del divisor DesignWare (`my_div_..._DW_div_uns_0`: `remainder[2:0]` y `divide_by_0`) sin conectar. Inofensivos aquí (la lógica no usa el resto de la división), pero hay que saber explicarlos.
- **Design Rules**: violaciones de max_transition/max_fanout/max_capacitance dentro del QoR; vacías en las corridas reales.

## Cómo decidir cuál corrida es mejor

Orden de evaluación:

1. **Corrección primero**: `check_design` sin errores y sin warnings inexplicables, design rules limpias.
2. **Timing**: slack ≥ 0 en todas. Si alguna viola timing, queda descartada salvo que sea la única y haya que renegociar el reloj.
3. **Con timing cumplido, optimizar área y leakage**: la de menor área/leakage gana.
4. **Criterios de flujo**: si necesitas jerarquía preservada (debug, DFT por bloque, reuso), `-no_autoungroup` puede ganar aunque pague área; si la meta es potencia dinámica, `-gate_clock` (su beneficio se ve en dynamic power del `report_power`, no en leakage).

Aplicado a los datos reales: con timing holgado en todas y 1 misma violación de leakage-constraint en todas, **retime es la mejor corrida en área (276.52) y leakage (263.7 nW)** — a costa de aplanar la jerarquía por completo. Si se exige jerarquía, la mejor es `no_autoungroup` (315.30 / 334.1 nW, check_design limpio). `incremental` quedó peor en área y leakage y con warnings: en este diseño el incremental partió de una base distinta y no aporta.

## Versión mejorada del script

Mejoras que aportan valor real (manteniendo la estructura y el estilo del original):

1. `get_leakage_power` con orden valor-unidad correcto: `awk '{print $(NF-1), $NF}'` (imprime "263.7173 nW" en vez de "nW 263.7173").
2. Diffs sin ruido de fechas: los diffs reales están dominados por líneas `Date :`. Filtrarlas deja solo los cambios técnicos.
3. Baseline configurable por argumento, con el actual por defecto.

```bash
#!/bin/bash
# compare_reports.sh (v2)
# Uso: ./compare_reports.sh [baseline]   (default: retime)
OUT="comparador.md"
EASTER_EGG_IMG="reze.jpg"
BASELINE="${1:-retime}"

> "$OUT"

echo "# Comparacion de reportes de sintesis"  >> "$OUT"
echo ""                                        >> "$OUT"
echo "Generado: $(date)"                       >> "$OUT"
echo ""                                        >> "$OUT"

TAGS=$(find . -maxdepth 1 -mindepth 1 -type d | sed 's|./||' | sort)

get_total_area() {
    grep -i "Total cell area" "$1" 2>/dev/null | head -1 | awk -F: '{print $2}' | xargs
}

get_leakage_power() {
    grep -i "Cell Leakage Power" "$1" 2>/dev/null | tail -1 | awk '{print $(NF-1), $NF}' | xargs
}

get_critical_slack() {
    grep -i "critical path slack" "$1" 2>/dev/null | head -1 | awk -F: '{print $2}' | xargs
}

get_cell_count() {
    grep -i "leaf cell count" "$1" 2>/dev/null | head -1 | awk -F: '{print $2}' | xargs
}

get_design_rules() {
    grep -i -A5 "^Design Rules" "$1" 2>/dev/null | grep -i "violat" | xargs
}

get_constraints_violations() {
    if grep -qi "No timing violations" "$1" 2>/dev/null; then
        echo "0 (sin violaciones)"
    else
        grep -ci "VIOLATED" "$1" 2>/dev/null | xargs
    fi
}

get_check_design_status() {
    local errs=$(grep -ci "^Error" "$1" 2>/dev/null)
    local warns=$(grep -ci "^Warning" "$1" 2>/dev/null)
    echo "Errors: $errs, Warnings: $warns"
}

echo "## 1. Resumen por tipo"                                                    >> "$OUT"
echo ""                                                                           >> "$OUT"
echo "| Tag | Area | Leakage Power | Critical Slack | Cells | Constr. Viol. |"   >> "$OUT"
echo "|---|---|---|---|---|---|"                                                  >> "$OUT"

for tag in $TAGS; do
    area=$(get_total_area "$tag/report_area.rpt")
    leak=$(get_leakage_power "$tag/report_power.rpt")
    slack=$(get_critical_slack "$tag/report_qor.rpt")
    cells=$(get_cell_count "$tag/report_qor.rpt")
    cviol=$(get_constraints_violations "$tag/report_constraints.rpt")
    echo "| $tag | ${area:-N/A} | ${leak:-N/A} | ${slack:-N/A} | ${cells:-N/A} | ${cviol:-N/A} |" >> "$OUT"
done
echo "" >> "$OUT"

echo "## 2. Design Rules / Check Design"                                          >> "$OUT"
echo ""                                                                            >> "$OUT"
for tag in $TAGS; do
    echo "### $tag"                                                                >> "$OUT"
    echo "- **Design Rules:** $(get_design_rules "$tag/report_qor.rpt")"           >> "$OUT"
    echo "- **Check Design:** $(get_check_design_status "$tag/check_design.rpt")"  >> "$OUT"
    echo "" >> "$OUT"
done

echo "## 3. Diffs internos (before/after dentro de la misma corrida)"             >> "$OUT"
echo ""                                                                            >> "$OUT"

# diff sin las lineas de fecha, que son ruido en todos los reportes
diff_block() {
    local title="$1" f1="$2" f2="$3"
    if [[ -f "$f1" && -f "$f2" ]]; then
        local d
        d=$(diff <(grep -v "^Date" "$f1") <(grep -v "^Date" "$f2"))
        if [[ -n "$d" ]]; then
            echo "#### $title"   >> "$OUT"
            echo '```diff'       >> "$OUT"
            echo "$d"            >> "$OUT"
            echo '```'           >> "$OUT"
            echo ""              >> "$OUT"
        else
            echo "#### $title"                      >> "$OUT"
            echo "_(sin diferencias tecnicas)_"     >> "$OUT"
            echo ""                                 >> "$OUT"
        fi
    fi
}

for tag in $TAGS; do
    diff_block "[$tag] report_area: before vs after areaopt" \
        "$tag/report_area_before_areaopt.rpt" "$tag/report_area_after_areaopt.rpt"
    diff_block "[$tag] report_timing: before vs after areaopt" \
        "$tag/report_timing_before_areaopt.rpt" "$tag/report_timing_after_areaopt.rpt"
    diff_block "[$tag] report_area: base vs postopt (optimize_registers)" \
        "$tag/report_area.rpt" "$tag/report_area_postopt.rpt"
    diff_block "[$tag] report_timing: base vs postopt (optimize_registers)" \
        "$tag/report_timing.rpt" "$tag/report_timing_postopt.rpt"
    diff_block "[$tag] report_qor: before vs after areaopt" \
        "$tag/report_qor_before_areaopt.rpt" "$tag/report_qor_after_areaopt.rpt"
    diff_block "[$tag] report_constraints: base vs after areaopt" \
        "$tag/report_constraints.rpt" "$tag/report_constraints_after_areaopt.rpt"
    diff_block "[$tag] check_design: base vs after areaopt" \
        "$tag/check_design.rpt" "$tag/check_design_after_areaopt.rpt"
done

echo "## 4. Diffs cruzados vs baseline ($BASELINE)"                               >> "$OUT"
echo ""                                                                            >> "$OUT"

for tag in $TAGS; do
    [[ "$tag" == "$BASELINE" ]] && continue
    for repfile in report_area.rpt report_power.rpt report_qor.rpt report_constraints.rpt check_design.rpt; do
        diff_block "$repfile : $BASELINE vs $tag" "$BASELINE/$repfile" "$tag/$repfile"
    done
done

echo "---"                                                                         >> "$OUT"
echo ""                                                                            >> "$OUT"
echo "## Reze "                                                                    >> "$OUT"
echo ""                                                                            >> "$OUT"

if [[ -f "$EASTER_EGG_IMG" ]]; then
    echo "![easter egg]($EASTER_EGG_IMG)"                                          >> "$OUT"
else
    echo "_(coloca $EASTER_EGG_IMG en reports/ para verlo aqui)_"                  >> "$OUT"
fi

echo "" >> "$OUT"
echo "Listo. Resultado guardado en: $OUT"
```

## Checklist final del capítulo 9

- [ ] Sabes explicar cada función de extracción (grep/awk/xargs) del script
- [ ] Corriste el script dentro de `reports/` y el `comparador.md` tiene tabla + secciones
- [ ] Interpretaste la tabla real: retime mejor en área/leakage, no_autoungroup mejor con jerarquía
- [ ] Sabes justificar la violación única de constraints (max_leakage) y los LINT-28 del incremental
- [ ] Entiendes las tres mejoras de la v2 y por qué aportan

---

# Capítulo 10 — DFT

## Objetivo

Insertar scan en el CPU sintetizado con Design Compiler, entender cada comando del flujo DFT del curso y verificar las cadenas resultantes.

## Explicación conceptual

DFT (Design For Testability) resuelve un problema de fabricación: una vez producido el chip, hay que poder detectar defectos físicos (nodos pegados a 0/1, cortos). La técnica del curso es **scan**:

- **Scan Flip-Flops**: cada FF funcional se reemplaza por una celda scannable (tipo `sdf` en vez de `df`), que agrega un mux interno: dato funcional vs dato de shift.
- **Scan Chain**: los scan-FFs se conectan en serie (scan-in → FF1 → FF2 → ... → scan-out), formando un registro de desplazamiento gigante.
- **Scan Enable (test_se)**: la señal que selecciona el modo: 0 = operación normal, 1 = shift por la cadena.
- Operación de test: se **shiftea** un patrón por `test_si`, se aplica un ciclo funcional (captura), y se **shiftea** el resultado hacia afuera por `test_so`. Así cada FF se vuelve controlable y observable.
- **ATPG** (Automatic Test Pattern Generation): la herramienta posterior (TetraMAX/DFTMAX) genera automáticamente los patrones que maximizan el **Fault Coverage** — el porcentaje de fallas modeladas (stuck-at, etc.) que los patrones detectan.
- **Scan Insertion**: el paso, dentro de DC, que reconecta los FFs en cadenas y agrega los puertos de test. Es lo que hace este capítulo.

## El flujo completo (LAB3 del curso, sobre el top sintetizado)

Se inserta en el `dc.tcl` reemplazando/extendiendo la etapa de compile. Correrlo primero en modo interactivo para observar cada paso "en vivo":

```tcl
compile_ultra -scan

create_test_protocol -infer_clock -infer_asynch
dft_drc
preview_dft
insert_dft
dft_drc

compile_ultra -incr -scan

dft_drc

report_scan_path -view existing_dft -chain all
report_constraints -all_violators
write_scan_def -output dft.scandef
```

## Explicación de cada comando

### compile_ultra -scan

Compila el diseño igual que un `compile_ultra` normal, pero `-scan` le dice a DC que use celdas de **flip-flop con capacidad de scan** desde la librería, en vez de FFs funcionales simples. Desde el mapeo elige celdas con el mux interno de modo scan, aunque todavía **no** las conecta en cadena — solo asegura que la lógica secuencial esté hecha con el tipo de celda correcto para poder encadenarla después. En el CPU esto afecta a todos los registros: `cmd_reg`, `reg_a`, `reg_b`, los registros de salida `dout_high/low`, las banderas `zero/error` y el registro de estado de la FSM.

### create_test_protocol -infer_clock -infer_asynch

DFT necesita saber cómo manejar relojes y resets/sets asíncronos durante el modo de test (por ejemplo, para no disparar un reset accidentalmente mientras shiftea datos por la cadena). Este comando analiza el diseño y detecta automáticamente:

- `-infer_clock`: qué señales son relojes y cómo deben comportarse en modo test (en el CPU: `clk`).
- `-infer_asynch`: qué señales son sets/resets asíncronos, para generar las restricciones que los desactiven o controlen durante el shift (en el CPU: `rst`, que es reset asíncrono de todos los registros).

Sin esto, DC no sabría cómo generar el protocolo de test (la secuencia de señales de control) necesario para los pasos siguientes.

### dft_drc (primera vez)

DRC = Design Rule Check, pero específico para DFT. Revisa el diseño **antes** de insertar las cadenas para detectar problemas que impedirían un scan insertion correcto: FFs sin acceso controlable/observable, conflictos de reloj, lógica que bloquea la propagación de datos de test. Es preventivo: si hay errores aquí, `insert_dft` puede fallar o generar cadenas incompletas.

### preview_dft

Muestra una **previsualización** de lo que haría `insert_dft` sin ejecutarlo: cuántas cadenas de scan se crearían, cuántos flip-flops entrarían en cada una, y estimaciones de impacto (área, longitud de cadena). Revisa el plan antes de comprometerte.

### insert_dft

Aquí se insertan **realmente** las cadenas: DC reconecta los FFs scannables en cadena serie (scan-in → FF1 → FF2 → ... → scan-out), agregando las señales de control `test_si`, `test_se` y la salida `test_so`. Este es el paso que modifica físicamente el netlist.

### dft_drc (segunda vez)

Se repite el chequeo, ahora **después** de insertar, para confirmar que la inserción fue correcta y no quedaron violaciones (cadenas mal cerradas, FFs fuera de cadena).

### compile_ultra -incr -scan

Compile incremental (no repite todo desde cero: optimiza sobre lo que ya está) manteniendo la conciencia de scan. Re-optimiza la lógica combinacional alrededor de las cadenas recién insertadas — por ejemplo, arregla el timing degradado por el mux de scan agregado en cada FF — sin deshacer la estructura de scan ya insertada.

### dft_drc (tercera vez)

Chequeo final, después del compile incremental, para confirmar que la reoptimización no rompió ni degradó las cadenas ya validadas.

### report_scan_path -view existing_dft -chain all

Reporte de texto que lista las cadenas insertadas: qué flip-flops están en cada cadena, en qué orden (desde `test_si` hasta `test_so`) y cuántas cadenas hay. Es el equivalente textual de lo que se ve en la GUI con las líneas conectando `test_si`/`clk` a los registros.

### report_constraints -all_violators

Revisa si, tras todo el proceso DFT (inserción + compile incremental), quedaron violaciones de las constraints originales — porque el mux extra en cada FF introduce delay adicional que antes no existía.

### write_scan_def -output dft.scandef

Guarda el archivo **SCANDEF** (Scan Definition), formato estándar de la industria que describe la estructura de las cadenas (orden de FFs, conexiones scan-in/scan-out) para que las herramientas posteriores del flujo — ATPG, o place & route — sepan cómo están conectadas y puedan generar los patrones de prueba o respetar esa conectividad.

## Verificación en la GUI

Si abres la GUI (`start_gui`) después del proceso DFT, el esquemático ya muestra los componentes de scan insertion: los puertos nuevos `test_si`, `test_se`, `test_so` y las conexiones de la cadena atravesando los registros del diseño. Recorre la cadena en el esquemático y contrástala con `report_scan_path`.

## Cómo comprobar que todo salió correctamente

1. Los tres `dft_drc` terminan sin violaciones (el primero puede traer avisos que `create_test_protocol` resuelve).
2. `preview_dft` reportó un plan coherente (todas las FFs del diseño incluidas).
3. `report_scan_path` lista la(s) cadena(s) completa(s) con todos los registros del CPU.
4. `report_constraints -all_violators` no muestra violaciones de timing nuevas (o son absorbibles con el slack disponible — recuerda los ~3.5 ns de margen del capítulo 9).
5. Existe `dft.scandef`.

## Errores comunes

| Problema | Causa | Solución |
|---|---|---|
| `dft_drc` reporta FFs no scannables | Se compiló sin `-scan` (celdas df normales) | Repetir `compile_ultra -scan` |
| Violaciones de reloj/asíncronos en dft_drc | Falta el protocolo de test | Ejecutar `create_test_protocol -infer_clock -infer_asynch` antes |
| Timing violado tras insert_dft | Delay del mux de scan | `compile_ultra -incr -scan` y revisar constraints |
| Cadena incompleta en report_scan_path | Errores del primer dft_drc ignorados | Resolverlos antes de `insert_dft` |

## Checklist final del capítulo 10

- [ ] Puedes explicar scan, scan chain, scan enable, scan-FF, fault coverage y ATPG con el CPU como ejemplo
- [ ] Ejecutaste el flujo completo interactivamente observando cada paso
- [ ] Los tres dft_drc pasan limpios
- [ ] `report_scan_path` y el esquemático de la GUI cuentan la misma historia
- [ ] `dft.scandef` generado y guardado con la corrida

---

# Capítulo 11 — Scripts finales reutilizables

## Objetivo

Consolidar todo el flujo en un conjunto de scripts consistentes entre sí, completamente comentados. Convenciones comunes: todos asumen la estructura `rtl/` + `tb/` del proyecto CPU, todos usan las mismas flags verificadas del curso, y todos son inofensivos de releer meses después.

## vcs_compile.sh

```bash
#!/bin/bash
# vcs_compile.sh — Compila un testbench + su RTL con VCS listo para Verdi.
# Uso:
#   ./vcs_compile.sh tb/ALU_tb.sv rtl/ALU.sv
#   ./vcs_compile.sh tb/top_tb.sv rtl/top.sv rtl/control.sv rtl/ALU.sv rtl/memory.sv rtl/mux4.sv rtl/register_bank.sv
# El testbench SIEMPRE va primero (convención de VCS del curso).

if [[ $# -lt 2 ]]; then
    echo "Uso: $0 <testbench.sv> <rtl1.sv> [rtl2.sv ...]"
    exit 1
fi

# -sverilog        : soporte SystemVerilog
# -full64          : modo 64 bits (requerido en el setup del curso)
# -debug_access+all: acceso de debug completo (necesario para Verdi)
# -kdb             : Kernel Debug Base para Verdi (es -kdb, NO -kbd)
# -l compile.log   : log de compilacion
vcs -sverilog -full64 -debug_access+all -kdb "$@" -l compile.log

echo "Compilacion terminada. Ejecutable: ./simv  Log: compile.log"
```

## run_simv.sh

```bash
#!/bin/bash
# run_simv.sh — Ejecuta la simulacion compilada y guarda el log.
# Uso:
#   ./run_simv.sh                  (semilla por defecto)
#   ./run_simv.sh 12345            (semilla fija para reproducir una corrida)

SEED_ARG=""
if [[ -n "$1" ]]; then
    # +ntb_random_seed fija la semilla de $urandom_range para reproducibilidad
    SEED_ARG="+ntb_random_seed=$1"
fi

# -l sim.log : evidencia de la corrida; el fsdb lo genera el propio tb
./simv $SEED_ARG -l sim.log

echo "Simulacion terminada. Log: sim.log  Ondas: novas.fsdb (si el tb tiene \$fsdbDump*)"
```

## run_verdi.sh

```bash
#!/bin/bash
# run_verdi.sh — Abre Verdi con la base de datos de diseño y las ondas FSDB.
# Requiere: compilacion con -debug_access+all -kdb y simulacion ya ejecutada.

if [[ ! -d simv.daidir ]]; then
    echo "No existe simv.daidir. Compila primero (vcs_compile.sh)."
    exit 1
fi
if [[ ! -f novas.fsdb ]]; then
    echo "No existe novas.fsdb. Ejecuta la simulacion primero (run_simv.sh)."
    exit 1
fi

# -dbdir : base de datos de simulacion (diseño elaborado)
# -ssf   : Signal Scan File, el FSDB con las señales
# &      : en segundo plano para liberar la terminal
verdi -dbdir simv.daidir -ssf novas.fsdb &
```

## run_fsdb.sh

```bash
#!/bin/bash
# run_fsdb.sh — Flujo completo hasta las ondas: limpia, compila, simula y abre Verdi.
# Uso: ./run_fsdb.sh <testbench.sv> <rtl1.sv> [rtl2.sv ...]

if [[ $# -lt 2 ]]; then
    echo "Uso: $0 <testbench.sv> <rtl1.sv> [rtl2.sv ...]"
    exit 1
fi

# Limpieza previa: garantiza bases de datos frescas
rm -rf csrc/ simv* novas* ucli.key

vcs -sverilog -full64 -debug_access+all -kdb "$@" -l compile.log || exit 1
./simv -l sim.log || exit 1
verdi -dbdir simv.daidir -ssf novas.fsdb &
```

## run_verdi_cov.sh

```bash
#!/bin/bash
# run_verdi_cov.sh — Flujo completo de coverage: limpia, compila y simula con
# las mismas metricas, y abre Verdi Coverage.
# Uso: ./run_verdi_cov.sh <testbench.sv> <rtl1.sv> [rtl2.sv ...]

if [[ $# -lt 2 ]]; then
    echo "Uso: $0 <testbench.sv> <rtl1.sv> [rtl2.sv ...]"
    exit 1
fi

# Mismas metricas en compilacion y simulacion (obligatorio).
# Añadir +assert SOLO si el codigo tiene assertions.
CM="line+cond+fsm+tgl+branch"

# Borrar simv.vdb es CRITICO: las bases viejas acumulan datos y
# producen reportes engañosos tras cambiar el testbench.
rm -rf csrc/ simv* novas* ucli.key

vcs -cm $CM -sverilog -full64 -kdb -debug_access+all "$@" -l compile.log || exit 1
./simv -cm $CM -l sim.log || exit 1
verdi -cov -covdir ./simv.vdb/ &
```

## run_uvm.sh

```bash
#!/bin/bash
# run_uvm.sh — Compila y corre el proyecto UVM de la ALU.
# Uso:
#   ./run_uvm.sh                     (test por defecto: alu_base_test)
#   ./run_uvm.sh alu_div0_test
#   ./run_uvm.sh alu_invalid_test
# Ejecutar desde el directorio uvm_alu/ (que contiene rtl/ y tb/).

TEST="${1:-alu_base_test}"

rm -rf csrc/ simv* novas* *.log

# -ntb_opts uvm-1.2 : libreria UVM 1.2 incluida en VCS
# +incdir+tb/       : resuelve los `include del package (evita errores SFCOR)
# Orden de archivos : interface -> package -> RTL -> top
vcs -sverilog -ntb_opts uvm-1.2 -full64 -kdb -debug_access+all \
    +incdir+tb/ \
    tb/alu_if.sv \
    tb/alu_pkg.sv \
    rtl/ALU.v \
    tb/tb_top.sv \
    -l compile.log || exit 1

# +UVM_TESTNAME selecciona el test en runtime, sin recompilar
./simv +UVM_TESTNAME=$TEST -l run.log

echo "Test $TEST terminado. Revisa el UVM Report Summary al final de run.log"
```

## dc.tcl

```tcl
############################################################
# dc.tcl — Flujo de sintesis Design Compiler (FODISEGUA)
# Ejecutar:  dcnxt_shell -f scripts/dc.tcl | tee -i logs/dc.log
############################################################

############################################################
# Run tag: separa reportes y outputs de cada corrida.
# Cambiar el tag y la variante de compile_ultra en cada corrida.
############################################################
set run_tag "compile_ultra"
;# ejemplos: compile_ultra / no_autoungroup / no_boundary_optimization /
;#           gate_clock / retime / incremental

set report_dir "./reports/$run_tag"
set output_dir "./outputs/$run_tag"
file mkdir $report_dir
file mkdir $output_dir

############################################################
# 1. Search Path and Logic Library Setup
############################################################
set_app_var search_path "$search_path . ./rtl ./libs"
set_app_var target_library "sky130_fd_sc_hd__ff_100C_1v95.db sky130_fd_sc_hd__ss_100C_1v40.db"
set_app_var link_library "* $target_library"
get_libs

############################################################
# 2. RTL Reading and Link
# Para el lab:   {top.v mux4.v mux4_registered.v register_bank.v}, WIDTH=3
# Para el CPU:   -format sverilog {top.sv control.sv ALU.sv memory.sv mux4.sv register_bank.sv}, WIDTH=8
############################################################
analyze -format verilog {top.v mux4.v mux4_registered.v register_bank.v}
elaborate top -parameters "WIDTH=3"
link

############################################################
# 3. Constraints Setup (todas en funcion del periodo)
############################################################
set clk_val 10

create_clock -period $clk_val [get_ports clk] -name clk
set_clock_uncertainty -setup [expr $clk_val*0.1]  [get_clocks clk]
set_clock_transition  -max   [expr $clk_val*0.1]  [get_clocks clk]
set_clock_latency -source -max [expr $clk_val*0.05] [get_clocks clk]
set_clock_latency         -max [expr $clk_val*0.03] [get_clocks clk]

set_input_delay  -max [expr $clk_val*0.4] -clock clk [get_ports [remove_from_collection [all_inputs] clk]]
set_output_delay -max [expr $clk_val*0.5] -clock clk [get_ports [all_outputs]]

set_load -max 0.04 [all_outputs]
set_input_transition -min [expr $clk_val*0.01] [remove_from_collection [all_inputs] clk]
set_input_transition -max [expr $clk_val*0.1]  [remove_from_collection [all_inputs] clk]

############################################################
# 4. Pre-compile Reports (sobre el netlist GTECH restringido)
############################################################
report_clock                 > $report_dir/pre_syn_report_clock.rpt
report_clock -skew           > $report_dir/pre_syn_report_clock_skew.rpt
report_port -verbose         > $report_dir/pre_syn_report_port_constraints.rpt
check_timing                 > $report_dir/pre_syn_check_timing.rpt
check_design                 > $report_dir/pre_syn_check_design.rpt

############################################################
# 5. Compile/Synthesis — descomentar UNA variante por corrida
############################################################
compile_ultra
#compile_ultra -no_autoungroup
#compile_ultra -no_boundary_optimization
#compile_ultra -no_autoungroup -gate_clock
#compile_ultra -retime
#compile_ultra -no_autoungroup -gate_clock -incremental

############################################################
# 6. Post-compile Reports
############################################################
report_timing                     > $report_dir/report_timing.rpt
report_qor                        > $report_dir/report_qor.rpt
report_constraints -all_violators > $report_dir/report_constraints.rpt
report_power                      > $report_dir/report_power.rpt
check_design                      > $report_dir/check_design.rpt
report_area                       > $report_dir/report_area.rpt

############################################################
# Opcional: optimizaciones post-compile (LAB2)
############################################################
#set ff_before [sizeof_collection [all_registers]]
#optimize_registers
#set ff_after  [sizeof_collection [all_registers]]
#puts "FFs antes: $ff_before  despues: $ff_after"
#report_timing > $report_dir/report_timing_postopt.rpt
#report_area   > $report_dir/report_area_postopt.rpt

#report_area   > $report_dir/report_area_before_areaopt.rpt
#report_timing > $report_dir/report_timing_before_areaopt.rpt
#report_qor    > $report_dir/report_qor_before_areaopt.rpt
#optimize_netlist -area
#report_area        > $report_dir/report_area_after_areaopt.rpt
#report_timing      > $report_dir/report_timing_after_areaopt.rpt
#report_qor         > $report_dir/report_qor_after_areaopt.rpt
#report_constraints -all_violators > $report_dir/report_constraints_after_areaopt.rpt
#check_design       > $report_dir/check_design_after_areaopt.rpt

############################################################
# 7. Save Design
############################################################
change_names -rule verilog
write_file -format ddc     -hierarchy -out $output_dir/mapped.ddc
write_file -format verilog -hierarchy -out $output_dir/mapped.v
write_sdc $output_dir/mapped.sdc

puts "Corrida '$run_tag' terminada. Reportes en $report_dir, outputs en $output_dir"

############################################################
# GUI — comentar para corridas batch puras
############################################################
gui_start
```

## compare_reports.sh

La versión mejorada completa está en el capítulo 9. Se ejecuta dentro de `reports/`:

```bash
chmod +x compare_reports.sh
./compare_reports.sh            # baseline por defecto: retime
./compare_reports.sh gate_clock # baseline alternativo
```

## clean.sh

```bash
#!/bin/bash
# clean.sh — Limpia todos los artefactos regenerables de VCS/Verdi.
# NO toca rtl/, tb/, scripts/, reports/ ni outputs/.
# Uso:
#   ./clean.sh          (limpieza estandar)
#   ./clean.sh logs     (ademas borra *.log)

rm -rf csrc/ simv* novas* ucli.key verdiLog/ novas.rc novas_dump.log

if [[ "$1" == "logs" ]]; then
    rm -f *.log
fi

echo "Limpieza terminada."
```

## run_all.sh

```bash
#!/bin/bash
# run_all.sh — Regresion completa del CPU: limpia, compila y simula
# los 6 modulos en secuencia, guardando un log por modulo en logs/.
# Ejecutar desde el directorio CPU/ (que contiene rtl/ y tb/).

set -e   # aborta al primer error

mkdir -p logs

# Cada entrada: "<nombre> <testbench> <lista de rtl>"
RUNS=(
"mux4          tb/mux4_tb.sv          rtl/mux4.sv"
"register_bank tb/register_bank_tb.sv rtl/register_bank.sv"
"ALU           tb/ALU_tb.sv           rtl/ALU.sv"
"memory        tb/memory_tb.sv        rtl/memory.sv"
"control       tb/control_tb.sv       rtl/control.sv"
"top           tb/top_tb.sv           rtl/top.sv rtl/control.sv rtl/ALU.sv rtl/memory.sv rtl/mux4.sv rtl/register_bank.sv"
)

for run in "${RUNS[@]}"; do
    # separa el nombre del resto de la linea
    name=$(echo $run | awk '{print $1}')
    files=$(echo $run | cut -d' ' -f2-)

    echo "=== $name ==="
    rm -rf csrc/ simv* novas* ucli.key

    vcs -sverilog -full64 -debug_access+all -kdb $files -l logs/${name}_compile.log
    ./simv -l logs/${name}_sim.log

    echo "=== $name OK (logs/${name}_sim.log) ==="
done

echo "Regresion completa terminada. Logs en logs/."
```

## Consistencia entre scripts

- Flags de VCS idénticas en todos: `-sverilog -full64 -debug_access+all -kdb`.
- Limpieza idéntica en todos: `rm -rf csrc/ simv* novas* ucli.key`.
- Todo log queda en archivo (`-l ...`), nunca solo en pantalla.
- El `dc.tcl` y `compare_reports.sh` comparten la convención `reports/$run_tag` / `outputs/$run_tag`.
- Ningún script toca código fuente ni reportes: solo artefactos regenerables.

## Checklist final del manual

- [ ] Los 11 scripts existen, con permisos de ejecución (`chmod +x *.sh`)
- [ ] `run_all.sh` corre la regresión de los 6 módulos sin errores
- [ ] `run_verdi_cov.sh` reproduce el flujo de coverage con base limpia
- [ ] `run_uvm.sh` corre los tres tests UVM cambiando solo el argumento
- [ ] Una corrida completa de `dc.tcl` por variante llena `reports/$run_tag`
- [ ] `compare_reports.sh` genera el comparador y puedes defender la corrida ganadora
