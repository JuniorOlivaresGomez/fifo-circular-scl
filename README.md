# FIFO circular en SCL para TIA Portal

Bloque `FC_Fifo` para S7-1500 escrito en SCL, sin bloques de librería.
Implementa una FIFO como **buffer circular** (Head / Tail / Count) con
tamaño automático, validación de índices, órdenes por flanco y diagnóstico propio.

Pensado para **seguimiento de piezas** (*material tracking*): un sensor
identifica cada pieza al entrar y, aguas abajo, un actuador recibe los datos
de esa misma pieza en el orden correcto.

## Características

- Tamaño automático: el buffer es un `Array[*]` y la FC lee sus límites con
  `LOWER_BOUND` / `UPPER_BOUND`. Admite cualquier tamaño (hasta 32767 casillas)
  y cualquier límite inferior
- Push y Pop en tiempo constante: los datos no se mueven, solo los índices
- `Count` distingue FIFO vacía y llena cuando `Head = Tail`
- Push, Pop y Clear por flanco positivo: una pieza = una operación
- Orden de ejecución Clear → Validación → ResetDiag → Pop → Push
- Validación de rangos y coherencia: un índice corrupto bloquea la FIFO
  en lugar de provocar un acceso fuera del array
- Diagnóstico: `LastError`, `ErrorCount` (saturado) y `MaxCount`

## Contenido

| Archivo | Descripción |
|---|---|
| `src/FC_Fifo.scl` | `UDT_FifoItem`, `UDT_Fifo` y `FC_Fifo` en un único archivo fuente |
| `test/DB_Test.db` | Datos del banco de pruebas: tres FIFO con buffers [0..9], [0..4] y [1..10] |
| `test/FC_TestFifo.scl` | Llamadas del banco de pruebas (una por FIFO) |
| `docs/pruebas.md` | Protocolo de pruebas (15 casos, esperado / obtenido) |

## Requisitos

S7-1500, o S7-1200 con soporte de `Array[*]` en parámetros de bloque.
Probado con TIA Portal V19 y PLCSIM (CPU 1512C-1 PN, FW V2.9).

## Instalación en TIA Portal

1. Árbol del proyecto → PLC → **Fuentes externas** → **Agregar nuevo archivo externo** → `src/FC_Fifo.scl`
2. Clic derecho sobre el archivo → **Generar bloques a partir de fuente**
3. En un DB global (no remanente), crear **por cada FIFO una pareja** de variables:
   - `FifoTramoN : UDT_Fifo` (control)
   - `BufferTramoN : Array[0..N] of UDT_FifoItem` (datos; el tamaño lo eliges tú)

Para reproducir las pruebas, importar también `test/DB_Test.db` y
`test/FC_TestFifo.scl`, y llamar a `FC_TestFifo` desde OB1.

> El campo `VERSION` de los UDT exportados no indica la versión del proyecto
> (TIA no permite editarlo fuera de una librería). La versión es la del tag
> de la release y la del historial en la cabecera de `FC_Fifo`.

## Uso

```
"FC_Fifo"(Push      := "Sensor_Entrada",
          Pop       := "Sensor_Desvio",
          Clear     := "PB_ResetTracking",
          ResetDiag := "PB_ResetDiag",
          DataIn    := #itemIn,
          DataOut   => #itemOut,
          Peek      => #proximo,
          PushDone  => #pushDone,
          PopDone   => #popDone,
          Empty     => #vacia,
          Full      => #llena,
          Count     => #cantidad,
          Error     => #error,
          Status    => #status,
          Fifo      := "DB_Tracking".FifoTramo1,
          Buffer    := "DB_Tracking".BufferTramo1);
```

### Contrato de uso

- Llamar **una vez por ciclo** para cada FIFO (las memorias de flanco están en el UDT)
- No llamar a la misma FIFO desde OB distintos
- `DataOut` solo es válido en el ciclo con `PopDone = TRUE`
- Revisar `Error` / `Status` antes de usar `Count`, `Empty` o `Full`
- `Fifo` y `Buffer` deben ser pareja (`FifoTramoN` con `BufferTramoN`): la FC no puede comprobarlo
- Clear solo con la línea físicamente vacía

### Códigos de estado

| Status | Significado |
|---|---|
| 16#0000 | OK |
| 16#8001 | Push rechazado: FIFO llena |
| 16#8002 | Pop rechazado: FIFO vacía |
| 16#8003 | Índices corruptos: FIFO bloqueada hasta Clear |
| 16#8004 | Buffer de más de 32767 casillas: FIFO bloqueada |

## Limitaciones conocidas

- La FC no puede verificar que `Fifo` y `Buffer` sean pareja; un error de conexión
  mezcla los datos de dos FIFO sin generar error
- Solo admite elementos de tipo `UDT_FifoItem`
- Tamaño máximo 32767 casillas (índices de tipo Int)
- Detecta índices corruptos, no el desfase entre los datos y las piezas físicas

## Hoja de ruta

- `FC_Tracking`: FIFO encadenadas en una clasificadora con dos empujadores
- FB del empujador con máquina de estados y watchdog
- Validación en Factory IO
- Comparación con `LGF_FIFO` (librería LGF de Siemens)

## Licencia

MIT