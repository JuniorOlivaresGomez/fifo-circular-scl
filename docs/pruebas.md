# Protocolo de pruebas — FC_Fifo v2.0

Entorno: TIA Portal V19 + PLCSIM, CPU 1512C-1 PN (FW V2.9).
Banco de pruebas: `test/DB_Test.db` + `test/FC_TestFifo.scl`, llamado desde OB1.
Órdenes aplicadas desde tablas de observación (modificar, no forzar), una variable por vez.
Inicio limpio de cada prueba: Clear y ResetDiag (salvo que se indique otra cosa).

## Tablas de observación

| Tabla | FIFO | Variables principales |
|---|---|---|
| TestFIFO | FifoPrueba + BufferPrueba[0..9] | Push, Pop, Clear, ResetDiag, DataIn.PartID, Head, Tail, Count, LastError, ErrorCount, LastOut.PartID, BufferPrueba[0]/[9].PartID, Peek.PartID, Full, Error, Status |
| TestFIFOChica | FifoChica + BufferChico[0..4] | Las mismas, con las señales de la 2.ª FIFO |
| TestFIFOUno | FifoUno + BufferUno[1..10] | Las mismas, con las señales de la 3.ª FIFO |

## Casos base (FifoPrueba, Array[0..9])

| # | Caso | Acción | Resultado esperado | Obtenido |
|---|---|---|---|---|
| P1 | Push y flanco | PartID=1, Push=TRUE mantenido 5 s, luego FALSE | Count=1, Tail=1, Buffer[0]=1; sin más cambios durante los 5 s | OK |
| P2 | Llenado | 10 Push (PartID 1..10) | Count=10, Tail=0 (vuelta), Full=TRUE, Buffer[9]=10 | OK |
| P3 | Push con FIFO llena | Push PartID=11 | Count=10, Buffer[0]=1 intacto, LastError=16#8001, ErrorCount=1 | OK |
| P4 | Pop normal | 3 Push (1,2,3) + 1 Pop | LastOut=1, Head=1, Count=2, Peek=2, Buffer[0]=0 | OK |
| P5 | Pop con FIFO vacía | 2 Pop + 1 Pop extra | Count=0, Head=3 sin cambio en el extra, LastError=16#8002, ErrorCount=1 | OK |
| P6 | Vuelta de Head | Desde Head=3: 8 Push + 8 Pop | Head=1, Tail=1, Count=0, LastOut=8 | OK |
| P7 | Push + Pop simultáneos, FIFO llena | Tras P2: Push(11)+Pop en el mismo ciclo | LastOut=1, Count=10, Head=1, Tail=1, Buffer[0]=11 | OK |
| P8 | Clear | 3 Push, 3 Pop + 1 extra, 2 Push, Clear | Head=Tail=Count=0, buffer en 0, LastError=0; MaxCount=3 y ErrorCount=1 se conservan | OK |
| P9 | Clear + Push simultáneos | Clear y Push(9) en el mismo ciclo | Count=0, Buffer[0]=0: el Push se anula | OK |
| P10 | Corrupción fuera de rango | 2 Push, Head:=15, Pop | PLC en RUN; Error/Status=16#8003 fijos; Count=2; Peek vacío; Pop bloqueado | OK |
| P11 | Corrupción incoherente | 3 Push, Tail:=5, Pop | (0+3) MOD 10 = 3 ≠ 5 → Status=16#8003; Count=3, Head=0 sin cambios | OK |
| P12 | Recuperación | Clear sobre P11, Push(7) | Status=0, LastError=0, Count=1, Buffer[0]=7, Peek=7 | OK |
| P13 | ResetDiag | 2 Push, ResetDiag | ErrorCount=0, MaxCount=2 (ocupación actual), Count=2 | OK |

## Casos de v2.0 (Array[*])

| # | Caso | FIFO | Acción | Resultado esperado | Obtenido |
|---|---|---|---|---|---|
| P14 | Tamaño automático | FifoChica + BufferChico[0..4] | 5 Push + 1 Push | Tras 5: Count=5, Tail=0, Head=0, Full=TRUE. Tras el 6.º: LastError=16#8001, ErrorCount=1, Count=5 | OK |
| P15 | Límite inferior ≠ 0 | FifoUno + BufferUno[1..10] | 1 Push (PartID=1) | BufferUno[1].PartID=1, Head=0, Tail=1, Peek=1 | OK |

Regresión v2.0: P2, P6, P7, P10 y P11 repetidas con la versión final. Resultado OK.

## No probado

| Caso | Motivo |
|---|---|
| Status 16#8004 (buffer > 32767 casillas) | Requiere un array de más de 32767 elementos |
| Coherencia con Head + Count > 32767 | Requiere un buffer de más de 16383 casillas. Cubierto por revisión de código (cálculo en DInt) |
| Fifo y Buffer de parejas distintas | Por diseño, la FC no puede detectarlo (ver contrato de uso) |

## Notas

- `PushDone`, `PopDone`, `Error` y `Status` de los casos P3 y P5 duran un ciclo y no
  se ven en la tabla de observación; se verifican mediante `LastError` y `ErrorCount`.
- En P10 y P11 la corrupción es un estado: `Error` y `Status` permanecen activos
  mientras los índices sean inválidos.