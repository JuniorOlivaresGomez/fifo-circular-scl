# FIFO circular en SCL para TIA Portal

Bloque `FC_Fifo` para S7-1200/1500 escrito en SCL, sin bloques de librería.
Implementa una FIFO como **buffer circular** (Head / Tail / Count) con
validación de índices, órdenes por flanco y diagnóstico propio.

Pensado para **seguimiento de piezas** (*material tracking*): un sensor
identifica cada pieza al entrar y, aguas abajo, un actuador recibe los datos
de esa misma pieza en el orden correcto.

## Características

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
| `docs/pruebas.md` | Protocolo de pruebas (13 casos, esperado / obtenido) |

## Instalación en TIA Portal

1. Árbol del proyecto → PLC → **Fuentes externas** → **Agregar nuevo archivo externo** → `FC_Fifo.scl`
2. Clic derecho sobre el archivo → **Generar bloques a partir de fuente**
3. Crear un DB global (no remanente) con una variable de tipo `UDT_Fifo` por cada FIFO

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
          Fifo      := "DB_Tracking".FifoTramo1);
```

### Contrato de uso

- Llamar **una vez por ciclo** para cada FIFO (las memorias de flanco están en el UDT)
- No llamar a la misma FIFO desde OB distintos
- `DataOut` solo es válido en el ciclo con `PopDone = TRUE`
- Revisar `Error` / `Status` antes de usar `Count`, `Empty` o `Full`
- `C_SIZE` (FC) debe coincidir con el tamaño del array de `UDT_Fifo`
- Clear solo con la línea físicamente vacía

### Códigos de estado

| Status | Significado |
|---|---|
| 16#0000 | OK |
| 16#8001 | Push rechazado: FIFO llena |
| 16#8002 | Pop rechazado: FIFO vacía |
| 16#8003 | Índices corruptos: FIFO bloqueada hasta Clear |

## Limitaciones conocidas

- El tamaño está definido en dos sitios (`Array[0..9]` y `C_SIZE`)
- Solo admite elementos de tipo `UDT_FifoItem`
- Detecta índices corruptos, no el desfase entre los datos y las piezas físicas

## Hoja de ruta

- **v2.0**: buffer como `Array[*]` con `LOWER_BOUND` / `UPPER_BOUND` (sin `C_SIZE`)
- `FC_Tracking`: FIFO encadenadas en una clasificadora con dos empujadores
- FB del empujador con máquina de estados y watchdog
- Validación en Factory IO

## Licencia

MIT