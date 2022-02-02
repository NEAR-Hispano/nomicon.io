# Economía

**Esto está bajo fuerte desarrollo.**

## Units

| Name | Value |
| - | - |
| yoctoNEAR | monto indivisible más chico de la moneda nativa *NEAR*. |
| NEAR | `10**24` yoctoNEAR |
| block | unidad de tiempo más pequeña en la cadena |
| gas | unidad para medir el uso de la blockchain |

## Parámetros generales

| Name | Value |
| - | - |
| `INITIAL_SUPPLY` | `10**33` yoctoNEAR |
| `MIN_GAS_PRICE` | `10**5` yoctoNEAR |
| `REWARD_PCT_PER_YEAR` | `0.05` |
| `EPOCH_LENGTH` | `43,200` bloques |
| `EPOCHS_A_YEAR` | `730` epochs |
| `INITIAL_MAX_STORAGE` | `10 * 2**40` bytes == `10` TB |
| `TREASURY_PCT` | `0.1` |
| `TREASURY_ACCOUNT_ID` | `treasury` |
| `CONTRACT_PCT` | `0.3` |
| `INVALID_STATE_SLASH_PCT` | `0.05` |
| `ADJ_FEE` | `0.001` |
| `TOTAL_SEATS` | `100` |
| `ONLINE_THRESHOLD_MIN` | `0.9` |
| `ONLINE_THRESHOLD_MAX` | `0.99` |
| `BLOCK_PRODUCER_KICKOUT_THRESHOLD` | `0.9` |
| `CHUNK_PRODUCER_KICKOUT_THRESHOLD` | `0.6` |

## Variables generales

| Name | Description | Initial value |
| - | - | - |
| `totalSupply[t]` | Suministro total de NEAR en un epoch[t] dado | `INITIAL_SUPPLY` |
| `gasPrice[t]` | El costo de 1 unidad de *gas* en tokens NEAR (vea la sección Tarifas de transacción a continuación) | `MIN_GAS_PRICE` |
| `storageAmountPerByte[t]` | keeping constant, `INITIAL_SUPPLY / INITIAL_MAX_STORAGE` | `~9.09 * 10**19` yoctoNEAR |

## Emisión

El protocolo establece un techo para la emisión máxima de tokens, y dinámicamente decrementa esta emisión dependiendo en el monto total de tarifas en el sistema.

| Name | Description |
| - | - |
| `reward[t]` | `totalSupply[t]` * ((`1 + REWARD_PCT_PER_YEAR`) ** (`1/EPOCHS_A_YEAR`) - `1`) |
| `epochFee[t]` | `sum([(1 - DEVELOPER_PCT_PER_YEAR) * block.txFee + block.stateFee for block in epoch[t]])` |
| `issuance[t]` | La cantidad de token emitido en un cierto epoch[t], `issuance[t] = reward[t] - epochFee[t]` |

Donde `totalSupply[t]` es el número total de tokens en el sistema en un tiempo *t* dado.
Si `epochFee[t] > reward[t]` la emisión es negativa, por lo tanto `totalSupply[t]` se decrementa en el epoch dado.

## Tarifas de transacción

Cada transacción debe comprar el gas suficiente para cubrir el costo de la banda ancha y ejecución antes de ser incluída.

El gas unifica la ejecución y los bytes del uso de la banda ancha de la blockchain. Cada instrucción WASM o función pre-compilada se le asigna una cantidad de gas basado en medidas de computadora de denominador común. Lo mismo ocurre con el peso del ancho de banda utilizado en función de los costos unificados generales. Para un mapeo específico de los números de gas vea [???]().

El gas se le asigna su precio dinámicamente en tokens `NEAR`. En cada bloque `t`, actualizamos `gasPrice[t] = gasPrice[t - 1] * (gasUsed[t - 1] / gasLimit[t - 1] - 0.5) * ADJ_FEE`.

Donde `gasUsed[t] = sum([sum([gas(tx) for tx in chunk]) for chunk in block[t]])`.
`gasLimit[t]` es definido como `gasLimit[t] = gasLimit[t - 1] + validatorGasDiff[t - 1]`, donde `validatorGasDiff` es un parámetro el que cada fragmento productor puede incrementar o decrementar el límite del gas basado en cuanto tarde en ejecutar el fragmento anterior. `validatorGasDiff[t]` solo puede estar dentro del `±0.1%` del `gasLimit[t]` y solo si `gasUsed[t - 1] > 0.9 * gasLimit[t - 1]`.

## Participación en el estado

El monto de `NEAR` en una cuenta representa el derecho de esta de tomar una porción del estado global de la blockchain. Las transacciones fallan si la cuenta no tiene el balance suficiente para cubrir el almacenamiento requerido para la cuenta dada.

```python
def check_storage_cost(account):
    # Calcula requiredAmount dado el tamaño de la cuent.
    requiredAmount = sizeOf(account) * storageAmountPerByte
    return Ok() if account.amount + account.locked >= requiredAmount else Error(requiredAmount)

# Revisa cuando una transacción es recibida y verifica que es válida.
def verify_transaction(tx, signer_account):
    # ...
    # Actualiza la cuenta que firma con el monto que tendrá después de ejecutar esta tx.
    update_post_amount(signer_account, tx)
    result = check_storage_cost(signer_account)
    # Si el balance es suficiente O si la cuenta ha sido borrada por el dueño.
    if not result.ok() or DeleteAccount(tx.signer_id) in tx.actions:
        assert LackBalanceForState(signer_id: tx.signer_id, amount: result.err())

# Después de tocar / cambiar la cuenta, verificamos si todavía tiene el balance suficienta para cubrir su almacenamiento.
def on_account_change(block_height, account):
    # ... ejecutar transacciones / cambios en recibos ...
    # Validar poscondición y revertir si falla.
    result = check_storage_cost(sender_account)
    if not result.ok():
        assert LackBalanceForState(signer_id: tx.signer_id, amount: result.err())
```

Donde `sizeOf(account)` inclute el tamaño de `account_id`, la estructura de `account` y el tamaño de todos los datos guardados bajo la cuenta.

La cuenta puede terminar sin saldo suficiente en caso de que se reduzca. La cuenta se volverá inusable dado que todas las transacciones que se puedan originar fallarán (incluyendo la eliminación de la cuenta).
La única manera de recuperarla en este caso es mandando fondos extra desde diferentes cuentas.

## Validadores

Los validadores NEAR proveen sus recursos a cambio de una recompensa `epochReward[t]`, donde [t] representa el epoch considerado

| Name | Description |
| - | - |
| `epochReward[t]` | `= coinbaseReward[t] + epochFee[t]` |
| `coinbaseReward[t]` | La inflación máxima por epoch[t], en función de `REWARD_PCT_PER_YEAR / EPOCHS_A_YEAR` |


### Selección de validador

| Name | Description |
| - | - |
| `proposals: Proposal[]` | El arreglo de todas las nuevas transacciones apiladas que han pasado durante el epoch (si una cuenta tiene varias, solo la última es usada) |
| `current_validators` | El arreglo de todos los validadores existentes durante el epoch |
| `epoch[T]` | El epoch cuando el validator[v] es seleccionado de el arreglo de subasta `proposals` |
| `seat_price` | La participación mínima necesaria para convertirse en validador en el epoch[T] |
| `stake[v]` | La cantidad en tokens NEAR stakeados por el validador[v] durante la subasta al final de epoch[T-2], menos `INCLUSION_FEE` |
| `shard[v]` | El fragmento es asignado aleatoriamente al validator[v] en el epoch[T-1], para que así su nodo puede descargar y sincronizarse con su estado |
| `num_allocated_seats[v]` | Número de asientos asignados al validator[v], calculados desde stake[v]/seatPrice |
| `validatorAssignments` | El arreglo ordenado resultante de todas las `proposals` con un stake mayor que el `seatPrice` |

```rust
struct Proposal {
    account_id: AccountId,
    stake: Balance,
    public_key: PublicKey,
}
```

Durante el epoch, la salida de stakear transacciones produce `proposals`, que son recolectadas en la forma de `Proposal`s.
Al final de cada epoch `T`, el siguiente algoritmo es ejecutado para determinar los validadores para el epoch `T + 2`:

1. Para cada validador en `current_validators` determina `num_blocks_produced`, `num_chunks_produced` basado en lo que produjeron durante el epoch.
2. Remueve validadores, para quienes `num_blocks_produced < num_blocks_expected * BLOCK_PRODUCER_KICKOUT_THRESHOLD` o `num_chunks_produced < num_chunks_expected * CHUNK_PRODUCER_KICKOUT_THRESHOLD`.
3. Agrega validadores de `proposals`, si el validador está también en `current_validators`, el stakeo considerado de la propuesta es `0 if proposal.stake == 0 else proposal.stake + reward[proposal.account_id]`.
4. Encuentra el precio asiento `seat_price = findSeatPrice(current_validators - kickedout_validators + proposals, num_seats)`, donde cada validador obtiene `floor(stake[v] / seat_price)` asientos y `seat_price` es el número entero más grande tal que el número total de asientos es al menos `num_seats`.
5. Filtra validadores y propuestas a solo esos con stake mayor o igual al precio asiento.
6. Para cada validador, los replica por número de asientos que obtienen `floor(stake[v] / seat_price)`.
7. Barajar aleatoriamente con la semilla de la aleatoriedad generada en el último bloque del epoch actual (a través de `VRF(block_producer.private_key, block_hash)`).
8. Quitar todos los asientso que están sobre el `num_seats` que se necesita.
10. Usa este conjunto para productores de bloques y cambia la ventana sobre él como productores de fragmentos

```python
def findSeatPrice(stakes, num_seats):
    """Encuentre el precio del asiento dado el conjunto de stakes y la cantidad de asientos requeridos.

    El precio asiento es el número entero más alto tal que si sumamos `floor(stakes[i] / seat_price)` sea al menos `num_seats`.
    """
    stakes = sorted(stakes)
    total_stakes = sum(stakes)
    assert total_stakes >= num_seats, "Total stakes should be above number of seats"
    left, right = 1, total_stakes + 1
    while True:
        if left == right - 1:
            return left
        mid = (left + right) // 2
        sum = 0
        for stake in stakes:
            sum += stake // mid
            if sum >= num_seats:
                left = mid
                break
        right = mid
```

### Cálculo de las recompensas del validador

Nota: todos lo cálculos son hechas con números Racionales.

La recompensa total en cada epoch `t` es igual a:
```python
total_reward[t] = floor(totalSupply * max_inflation_rate * num_blocks_per_year / epoch_length)
```

donde `max_inflation_rate`, `num_blocks_per_year`, `epoch_length` son parámetros génesis y `totalSupply` es
tomado del último bloque en el epoch.

Después de que una fracción de la recompensa va a la tesorería y el monto restante será usado para calcular las recompensas del validador:
```python
treasury_reward[t] = floor(reward[t] * protocol_reward_rate)
validator_reward[t] = total_reward[t] - treasury_reward[t]
```

Los validadores que no llegaron al umbral para los bloques o fragmentos se sacan y no obtienen ninguna recompensa, de lo contrario el tiempo de actividad
de un validador es calculado:

```python
pct_online[t][j] = (num_produced_blocks[t][j] / expected_produced_blocks[t][j] + num_produced_chunks[t][j] / expected_produced_chunks[t][j]) / 2
if pct_online > ONLINE_THRESHOLD:
    uptime[t][j] = min(1, (pct_online[t][j] - ONLINE_THRESHOLD_MIN) / (ONLINE_THRESHOLD_MAX - ONLINE_THRESHOLD_MIN))
else:
    uptime[t][j] = 0
```

Donde `expected_produced_blocks` y `expected_produced_chunks` es el número de bloques y fragmentos respectivos que se espera sean producidos por el validador `j` dado en el epoch `t`.

La recompensa específica del `validator[t][j]` para el epoch `t` es pues proporcional a la fracción de stake de este validador del stake total:

```python
validatorReward[t][j] = floor(uptime[t][j] * stake[t][j] * validator_reward[t] / total_stake[t])
```

### Slashing

#### ChunkProofs (pruebas de fragmento)

```python
# Verifica que el fragmento es inválido, porque las pruebas en el encabezado no coincide con el cuerpo.
def chunk_proofs_condition(chunk):
    # TODO

# Al final del epoch, ejecuta actualizar validadores y
# determina cuanto se recortarán los validadores.
def end_of_epoch_update_validators(validators):
    # ...
    for validator in validators:
        if validator.is_slashed:
            validator.stake -= INVALID_STATE_SLASH_PCT * validator.stake
```

#### ChunkState (estado del fragmento)

```python
# Verifica que la raíz del estado de la publicación del encabezado del fragmento sea inválida,
# porque la ejecución del fragmento anterior no lleva a eso.
def chunk_state_condition(prev_chunk, prev_state, chunk_header):
    # TODO
    
# Al final del epoch, ejecuta actualizar validadores y
# determina cuanto se recortarán los validadores.
def end_of_epoch(..., validators):
    # ...
    for validator in validators:
        if validator.is_slashed:
            validator.stake -= INVALID_STATE_SLASH_PCT * validator.stake
```

## Protocolo de tesorería

La cuenta de tesorería `TREASURY_ACCOUNT_ID` recibe una fracción de recompensa cada epoch `t`:

```python
# Al final del epoch, actualizar tesorería
def end_of_epoch(..., reward):
    # ...
    accounts[TREASURY_ACCOUNT_ID].amount = treasury_reward[t]
```

## Recompensas de contrato

La cuenta contrato es recompensada con el 30% del gas quemado durante la ejecución de sus funciones.
La recompensa se acredita a la cuenta contrato después de aplicar el recibo correspondiente con [`FunctionCallAction`](../RuntimeSpec/Actions.md#functioncallaction), el gas es convertido a tokens usando el precio del gas del bloque actual.

Puede leer más sobre:
- [ejecución de recibos](../RuntimeSpec/Receipts.md);
- [tarifas de tiempo de ejecución](../RuntimeSpec/Fees/Fees.md) con la descripción de [como se cobra el gas](../RuntimeSpec/Fees/Fees.md#gas-tracking).
