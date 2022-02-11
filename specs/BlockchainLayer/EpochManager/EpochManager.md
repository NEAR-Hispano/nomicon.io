# EpochManager

## Finalizando un epoch

En el último bloque del epoch `T`, `EpochManager` calcula `EpochInfo` para el epoch `T+2`, que
es definida por `EpochInfo` para `T+1` y la información agregada de los bloques del epoch `T`.

`EpochInfo` es toda la la información que `EpochManager` almacena para el epoch, que es:
- `epoch_height`: altura del epoch (`T+2`)
- `validators`: la lista de validadores seleccionados para el epoch `T+2`
- `validator_to_index`: Mapeo desde id de la cuenta al index en `validators`
- `block_producers_settlement`: define el mapeo desde la altura hasta el productor de bloques
- `chunk_producers_settlement`: define el mapeo desde la altura y el id del fragmento hasta el productor de fragmentos
- `hidden_validators_settlement`: TODO
- `fishermen`, `fishermen_to_index`: TODO. desactivado en mainnet a través de un `fishermen_threshold` grande en configuración
- `stake_change`: TODO
- `validator_reward`: recompensa de validador para el epoch `T`
- `validator_kickout`: vea [conjunto Kickout](#kickout-set)
- `minted_amount`: tokens minteados en el epoch `T`
- `seat_price`: precio asiento del epoch
- `protocol_version`: TODO

Agregando bloques de el epoch calcula los siguientes conjuntos:
- `block_stats`/`chunk_stats`: estadísticas de actividad en forma de bloques/fragmentos `produced` y `expected` para cada validador de `T`
- `proposals`: propuestas de stake hechas en el epoch `T`. Si una cuenta hizo múltiples propuestas, la última es usada.
- `slashes`: vea [conjunto Slash](#slash-set)

### Conjunto Slash
NOTA: el slashing actualmente está desactivado. Lo que sigue a continuación es el diseño actual, que puede cambiar.

Los conjuntos Slash se mantenidas en bloques. Si un validador se slashea en un epoch `T`, los bloques subsecuentes de los epochs `T` y
`T+1` lo mantienen en sus conjuntos slash. Al final del epoch `T`, el validador slasheado también se agrega a `kickout[T+2]`.
Las propuestas de los bloques en los conjuntos slash son ignoradas.

Se queda en el conjunto slash (y conjunto kickout) por dos o tres epochs dependiendo en si iba a ser un validador en `T+1`:
- Caso común: `v` está en `validators[T]` y en `validators[T+1]`
   - las propuestas de `v` en `T`, `T+1` y `T+2` son ignorados
   - `v` se agrega a `kickout[T+2]`, `kickout[T+3]` y `kickout[T+4]` como slasheados
   - `v` puede stakear otra vez empezando con el primer bloque de `T+3`.
- Si `v` está en `validators[T]` pero no en `validators[T+1]` (e.j. si no está stakeado en `T-1`)
   - las propuestas de `v` en `T` y `T+1` es ignorado
   - `v` es agregado a `kickout[T+2]` y `kickout[T+3]` como slasheado
   - `v` puede hacer una propuesta en `T+2` para volverse un validador en `T+4`
- Si `v` está en `validators[T-1]` pero no en `validators[T]` (e.j. hizo comportamiento slashable justo antes de rotar)
    - las propuestas de `v` en `T` y `T+1` es ignorado
    - `v` es agregado a `kickout[T+2]` y `kickout[T+3]` como slasheado
    - `v` puede hacer una propuesta en `T+2` para volverse un validador en `T+4`
    
## Calculando EpochInfo

### Conjunto Kickout
`kickout[T+2]` contiene los validadores del epoch `T+1` que dejan de ser validadores en `T+2`, y también cuentas que no
son necesariamente validadores de `T+1`, pero se mantienen en los conjuntos de slashing por la regla descrita [anteriormente](#slash-set).

`kickout[T+2]` es calculado de la manera siguiente:
1. `Slashed`: cuentas en el conjunto slash del último bloque en `T`
2. `Unstaked`: cuentas que remueven su stake en el epoch `T`, si su stake es diferente de 0 para el epoxh `T+1`
3. `NotEnoughBlocks/NotEnoughChunks`: Para cada validador se calcula la relación entre los bloques producidos y los bloques esperados producidos (lo mismo con fragmentos producidos/esperados).
    - Excepción: Si todos los validadores de `T` están en `kickout[T+1]` o serán expulsados, no expulsamos el
      validador con el máximo número de bloques producidos. Si hay múltiples, escogemos el que tenga el id
      de validador mínimo en el epoch.
4. `NotEnoughStake`: calculado después de la selección de validador. Las cuentas que tienen stake en el epoch `T+1`, pero no alcanza el umbral del stake para el epoch `T+2`.
5. `DidNotGetASeat`: calculado después de la selección de validador. Las cuentas que tienen stake en el epoch `T+1`, alcanza el umbral para el epoch `T+2`, pero no obtuvo ningún asiento.

### Procesamiento de propuestas
El conjunto de propuestas es procesado por el algoritmo de selección de validador, pero antes de eso, el conjunto de propuestas es ajustado
de la siguiente manera:
1. Si una cuenta está en el conjunto slash a partir del final del `T`, o se expulsa para `NotEnoughBlocks/NotEnoughChunks` en el epoch `T`,
  su propuesta es ignorada.
2. Si un validador está en `validators[T+1]`, y no hizo una propuesta, agrega una propuesta implícita con su stake en `T+1`.
3. Si un validador está en `validators[T]` y `validators[T+1]`, e hizo una propuesta en `T` (incluyendo las implícitas),
  entonces sus recompensas por epoch `T` es automáticamente agregada a la propuesta.

El conjunto ajustado de propuestas es usado para calcular el precio asiento, y determinar `validators`,`block_producers_settlement` y 
los conjuntos `chunk_producers_settlement`. Este algoritmo está descrito en [Economía](../../Economics/README.md#validator-selection).

### Recompensa de validador
El cálculo de recompensas se describe en la sección [Economía](../../Economics/README.md#rewards-calculation).
