# Protocolo de pruebas — FC_Fifo v1.1

Entorno: TIA Portal V19 + PLCSIM. FIFO de 10 casillas (`DB_Test.FifoPrueba`).
Órdenes aplicadas desde una tabla de observación, una variable por vez.
Cada prueba parte de una FIFO vacía (STOP → RUN o Clear) salvo que se indique.

| # | Caso | Acción | Resultado esperado | Obtenido |
|---|---|---|---|---|
| P1 | Push y flanco | PartID=1, Push=TRUE mantenido 5 s, luego FALSE | Count=1, Tail=1, Buffer[0]=1; sin más cambios durante los 5 s | OK |
| P2 | Llenado | 9 Push más (PartID 2..10) | Count=10, Tail=0 (vuelta), Full=TRUE, Buffer[9]=10 | OK |
| P3 | Push con FIFO llena | Push PartID=11 | Count=10, Buffer[0]=1 intacto, LastError=16#8001, ErrorCount=1 | OK |
| P4 | Pop normal | 3 Push (1,2,3) + 1 Pop | LastOut=1, Head=1, Count=2, Peek=2, Buffer[0]=0 | OK |
| P5 | Pop con FIFO vacía | 2 Pop + 1 Pop extra | Count=0, Head=3 sin cambio en el extra, LastError=16#8002, ErrorCount=1 | OK |
| P6 | Vuelta de Head | Desde Head=3: 8 Push + 8 Pop | Head=1, Tail=1, Count=0 | OK |
| P7 | Push + Pop simultáneos, FIFO llena | 10 Push; Push(11)+Pop en el mismo ciclo | LastOut=1, Count=10, Head=1, Tail=1, Buffer[0]=11 | OK |
| P8 | Clear | 3 Push, 3 Pop + 1 extra, 2 Push, Clear | Head=Tail=Count=0, buffer en 0, LastError=0; MaxCount=3 y ErrorCount=1 se conservan | OK |
| P9 | Clear + Push simultáneos | Clear y Push(9) en el mismo ciclo | Count=0, Buffer[0]=0: el Push se anula | OK |
| P10 | Corrupción fuera de rango | 2 Push, Head:=15, Pop | PLC en RUN; Error/Status=16#8003 fijos; Count=2; Peek vacío; Pop bloqueado | OK |
| P11 | Corrupción incoherente | Clear, 3 Push, Tail:=5, Pop | (0+3) MOD 10 = 3 ≠ 5 → Status=16#8003; Count=3, Head=0 sin cambios | OK |
| P12 | Recuperación | Clear sobre P11, Push(7) | Status=0, LastError=0, Count=1, Buffer[0]=7 | OK |
| P13 | ResetDiag | 2 Push, ResetDiag | ErrorCount=0, MaxCount=2 (ocupación actual), Count=2 | OK |

Notas:
- `PushDone`, `PopDone`, `Error` y `Status` de los casos P3 y P5 duran un ciclo
  y no se ven en la tabla de observación; se verifican mediante `LastError`
  y `ErrorCount`.
- En P10 y P11 la corrupción es un estado: `Error` y `Status` permanecen
  activos mientras los índices sean inválidos.