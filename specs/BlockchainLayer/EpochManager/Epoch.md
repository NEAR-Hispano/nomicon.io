# Epoch

Todos los bloques son divididos en epochs. Dentro de un epoch, el conjunto de validadores es ajustado, y la rotación de validadores
pasa dentro de los límites del epoch.

Un bloque génesis está en su propio epoch. Después de que un bloque está en el epoch de su padre
o empieza en un epoch nuevo si se logran ciertas condiciones.

Dentro de un epoch, la asignación de validador está basada en la altura del bloque: cada altura tiene un productor de bloque asignado, y
cada altura y fragmento tiene un productor de fragmento.

### Final de un epoch
Un bloque está definido para ser el último bloque en su epoch si es el bloque génesis o si las siguientes condiciones se cumplen:
- Sea `estimated_next_epoch_start = first_block_in_epoch.height + epoch_length`
- `block.height + 1 >= estimated_next_epoch_start`
- `block.last_finalized_height + 3 >= estimated_next_epoch_start`

`epoch_length` está definido en `genesis_config` y tiene un valor de altura delta `43200` en mainnet (12 horas a un bloque por segundo).

### EpochHeight
Los epochs en una cadena puede ser identificados por la altura, que está definida en la siguiente manera:
- Un epoch especial que contiene el solo bloque génesis: undefined
- El epoch empezando del bloque que va después de génesis: `0` (NOTA: en la implementación actual es `1` debido a un bug, para que no haya epochs con altura `1`)
- Siguientes epochs: altura del epoch padre más uno

### Epoch id
Cada bloque almacena el id de su epoch - `epoch_id`.

El id del epoch está definida como
- Para el bloque especial génesis epoch es `0`
- Para el epoch con altura `0` es `0` (NOTA: los primeros dos epochs usan el mismo id de epoch)
- Para el epoch con altura `1` es el hash del bloque génesis
- Para el epoch con altura `T+2` es el hash de el último bloque en el epoch `T`

### Final de un epoch
- Después de procesar el último bloque del epoch `T`, el `EpochManager` agrega información del bloque del epoch, y calcula
el validador establecido por el epoch `T+2`. Este proceso se describe en [EpochManager](EpochManager.md).
- Después de eso, el conjunto de validadores rota a la época `T+1`, y el siguiente bloque es producido por el validador del nuevo conjunto
- Aplicando el primer bloque del epoch `T+1`, en adición a una transición normal, también aplica las transiciones de estado por epoch:
  recompensas de validador, desbloqueos de stakes y slashing.
