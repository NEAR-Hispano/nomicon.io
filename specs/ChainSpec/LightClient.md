# Cliente ligero

El estado del cliente ligero está definido por:

1. `BlockHeaderInnerLiteView` para la cabeza actual (que contiene `height`, `epoch_id`, `next_epoch_id`, `prev_state_root`, `outcome_root`, `timestamp`, el hash de los productores de bloque establecido por el siguiente epoch `next_bp_hash`, y la raíz merkle para todos los hashes de bloque `block_merkle_root`);
2. El conjunto de productores de bloques para los actuales y siguientes epochs.

El `epoch_id` se refiere al epoch al que pertenece el bloque que es la cabeza conocida actual, y `next_epoch_id` es el epoch que seguirá después.

Los clientes ligeros operan trayendo periódicamente instancias de `LightClientBlockView` a través de un end-point RPC descrito a [continución](#rpc-end-point).

El cliente ligero no necesita recibir `LightClientBlockView` para todos los bloques. Tener el `LightClientBlockView` para el bloque `B` es suficiente para poder verificar cualquier declaración acerca del estado o las salidas en cualquier bloque en la ascendencia de `B` (incluyendo al mismo `B`). Particularmente, tener `LightClientBlockView` en la cabeza es suficiente para verificar localmente cualquier declaración acerca del estado o las salidas en cualquier bloque en la cadena canónica.

Sin embargo, para verificar la validez de un `LightClientBlockView` particular, el cliente ligero de de haber verificado un `LightClientBlockView` para al menos un bloque en el epoch anterior, por lo tanto para sincronizar para la cabeza el cliente ligero tendrá que traer y verificar un `LightClientBlockView` por cada epoch pasado.

## Validación de vistas de bloque de clientes ligeros

```rust
pub enum ApprovalInner {
    Endorsement(CryptoHash),
    Skip(BlockHeight)
}

pub struct ValidatorStakeView {
    pub account_id: AccountId,
    pub public_key: PublicKey,
    pub stake: Balance,
}

pub struct BlockHeaderInnerLiteView {
    pub height: BlockHeight,
    pub epoch_id: CryptoHash,
    pub next_epoch_id: CryptoHash,
    pub prev_state_root: CryptoHash,
    pub outcome_root: CryptoHash,
    pub timestamp: u64,
    pub next_bp_hash: CryptoHash,
    pub block_merkle_root: CryptoHash,
}

pub struct LightClientBlockLiteView {
    pub prev_block_hash: CryptoHash,
    pub inner_rest_hash: CryptoHash,
    pub inner_lite: BlockHeaderInnerLiteView,
}


pub struct LightClientBlockView {
    pub prev_block_hash: CryptoHash,
    pub next_block_inner_hash: CryptoHash,
    pub inner_lite: BlockHeaderInnerLiteView,
    pub inner_rest_hash: CryptoHash,
    pub next_bps: Option<Vec<ValidatorStakeView>>,
    pub approvals_after_next: Vec<Option<Signature>>,
}
```

Recuerda que el hash del bloque es

```rust
sha256(concat(
    sha256(concat(
        sha256(borsh(inner_lite)),
        sha256(borsh(inner_rest))
    )),
    prev_hash
))
```

Los campos `prev_block_hash`, `next_block_inner_hash` y `inner_rest_hash` son usados para reconstruir los hashes del actual bloque y el siguiente, y las aprobaciones que serán firmadas de la manera siguiente (donde `block_view` es una instancia de `LightClientBlockView`):

```python
def reconstruct_light_client_block_view_fields(block_view):
    current_block_hash = sha256(concat(
        sha256(concat(
            sha256(borsh(block_view.inner_lite)),
            block_view.inner_rest_hash,
        )),
        block_view.prev_block_hash
    ))

    next_block_hash = sha256(concat(
        block_view.next_block_inner_hash,
        current_block_hash
    ))

    approval_message = concat(
        borsh(ApprovalInner::Endorsement(next_block_hash)),
        little_endian(block_view.inner_lite.height + 2)
    )

    return (current_block_hash, next_block_hash, approval_message)
```

El cliente liger actualiza su cabeza con la información de `LightClientBlockView` si:

1. La altura del bloque es mayor que la altura de la cabeza actual;
2. El epoch del bloque es igual al `epoch_id` o `next_epoch_id` conocido para la cabeza actual;
3. Si el epoch del bloque es igual al `next_epoch_id` de la cabeza, entonces `next_bps` no es `None`;
4. `approvals_after_next` contiene firmas válidas en `approval_message` de los productores de bloques del epoch correspondiente (vea la siguiente sección):
5. Las firmas presentes en `approvals_after_next` corresponden a más de 2/3 del stake total (vea la siguiente sección).
6. Si `next_bps` no es none, `sha256(borsh(next_bps))` corresponde al `next_bp_hash` en `inner_lite`.

```python
def validate_and_update_head(block_view):
    global head
    global epoch_block_producers_map

    current_block_hash, next_block_hash, approval_message = reconstruct_light_client_block_view_fields(block_view)

    # (1)
    if block_view.inner_lite.height <= head.inner_lite.height:
        return False

    # (2)
    if block_view.inner_lite.epoch_id not in [head.inner_lite.epoch_id, head.inner_lite.next_epoch_id]:
        return False

    # (3)
    if block_view.inner_lite.epoch_id == head.inner_lite.next_epoch_id and block_view.next_bps is None:
        return False

    # (4) and (5)
    total_stake = 0
    approved_stake = 0

    epoch_block_producers = epoch_block_producers_map[block_view.inner_lite.epoch_id]
    for maybe_signature, block_producer in zip(block_view.approvals_after_next, epoch_block_producers):
        total_stake += block_producer.stake

        if maybe_signature is None:
            continue

        approved_stake += block_producer.stake
        if not verify_signature(
            public_key: block_producer.public_key,
            signature: maybe_signature,
            message: approval_message
        ):
            return False

    threshold = total_stake * 2 // 3
    if approved_stake <= threshold:
        return False

    # (6)
    if block_view.next_bps is not None:
        if sha256(borsh(block_view.next_bps)) != block_view.inner_lite.next_bp_hash:
            return False

        epoch_block_producers_map[block_view.inner_lite.next_epoch_id] = block_view.next_bps

    head = block_view
```

## Verificación de firma

Para simplicar el protocolo requerimos que el bloque siguiente y el bloque después del siguiente están en el mismo epoch que el blque al que `LightClientBlockView` corresponde. Está garantizado que cada epoch tiene al menos un bloque final para el cual los siguientes dos bloques que se construyen encima de el están en el mismo epoch.

Por construcción en el momento en que se valida `LightClientBlockView`, se conoce el conjunto de productores de bloques para su epoch. Específicamente, cuando la primer vista de cliente ligero del epoch anterior fue procesada, debido a (3) el `next_bps` no era `None`, y debido que a (6) correspondía al `next_bp_hash` en el header del bloque.

La suma de todos los stakes de `next_bps` en el epoch anterior es `total_stake` mencionado en (5) anteriormente.

Las firmas en el `LightClientBlockView::approvals_after_next` son firmas en `approval_message`. La firma `i`-ésima en `approvals_after_next`, si está presente, debe validar contra la `i`-ésima llave pública en `next_bps` del epoch anterior. `approvals_after_next` puede contener menos elementos que `next_bps` en el epoch anterior.

`approvals_after_next` también puede contener más firmas que la longitud de `next_bps` en el epoch anterior. Esto es debido al hecho de que, por [consenso](./Consensus.md), los últimos bloques en cada epoch contienen firmas de los productores de bloques del epoch actual y del epoch siguiente. Las girmas finales pueden ser seguramente ignoradas por la implementación del cliente ligero.

## Verificación de prueba

[Prueba del resultado de la transacción]: #transaction-outcome-proofs
### Pruebas de resultados de transacciones

Para verificar que una transacción o recibo pasa en la cadena, un cliente ligero puede pedir una prueba a través de rpc proporcionando un `id`, que es de tipo
```rust
pub enum TransactionOrReceiptId {
    Transaction { hash: CryptoHash, sender: AccountId },
    Receipt { id: CryptoHash, receiver: AccountId },
}
```
y el hash del bloque de la cabeza del cliente ligero. El rpc regresará la siguiente estructura
```rust
pub struct RpcLightClientExecutionProofResponse {
    /// Proof of execution outcome
    pub outcome_proof: ExecutionOutcomeWithIdView,
    /// Proof of shard execution outcome root
    pub outcome_root_proof: MerklePath,
    /// A light weight representation of block that contains the outcome root
    pub block_header_lite: LightClientBlockLiteView,
    /// Proof of the existence of the block in the block merkle tree,
    /// which consists of blocks up to the light client head
    pub block_proof: MerklePath,
}
```
que incluye todo lo que un cliente ligero necesita para probar la salida de la ejecución de la transacción o recibo dado.
Aquí `ExecutionOutcomeWithIdView` es
```rust
pub struct ExecutionOutcomeWithIdView {
    /// Proof of the execution outcome
    pub proof: MerklePath,
    /// Block hash of the block that contains the outcome root
    pub block_hash: CryptoHash,
    /// Id of the execution (transaction or receipt)
    pub id: CryptoHash,
    /// The actual outcome
    pub outcome: ExecutionOutcomeView,
}
```

La prueba de verificación puede ser partida en dos pasos, verificación de la raíz de la salida de ejecución y verificación de la raíz
de bloque merkle.

#### Verificación de la raíz de la salida de ejecución
Si la raíz de la salida de la transacción o recibo está incluída en el bloque `H`, entonces `outcome_proof` incluye en hash del bloque
`H`, así como la prueba merkle de la salida de la ejecución en su fragmento dado. La salida de la raíz en `H` puede ser
reconstruída por
```python
shard_outcome_root = compute_root(sha256(borsh(execution_outcome)), outcome_proof.proof)
block_outcome_root = compute_root(sha256(borsh(shard_outcome_root)), outcome_root_proof)
```

Esta raíz de salida debe empatar con la raíz de salida en `block_header_lite.inner_lite`.

#### Verificación de la raíz de bloque merkle.

Recuerda que el hash del bloque puede ser calculado desde `LightClientBlockLiteView` por
```rust
sha256(concat(
    sha256(concat(
        sha256(borsh(inner_lite)),
        sha256(borsh(inner_rest))
    )),
    prev_hash
))
```

La raíz del bloque merkle esperada puede ser caluculada por
```python
block_hash = compute_block_hash(block_header_lite)
block_merkle_root = compute_root(block_hash, block_proof)
```
que debe empatar con la raíz del bloque merkle en el bloque del cliente ligero de la cabeza del mismo cliente ligero.

## End-points RPC

### Bloque de cliente ligero

Hay un punto final único que exponen los nodos completos que los clientes ligeros pueden usar para obtener nuevos `LightClientBlockView`s:

```
http post http://127.0.0.1:3030/ jsonrpc=2.0 method=next_light_client_block params:="[<last known hash>]" id="dontcare"
```

El RPC devuelve el `LightClientBlock` para el bloque lo más lejos posible en el futuro del último hash conocido para que el cliente ligero aún lo acepte. Específicamente, regresa el último bloque final del epoch siguiente, o el último bloque final conocido. Si no hay bloques finales más nuevos que el que el cliente ligero conoce, el RPC regresa un resultado vacío.

Un cliente ligero independiente arrancaría solicitando bloques siguientes hasta que reciba un resultado vacío, y después periódicamente solicitar el siguiente bloque de cliente ligero.

Un cliente ligero basado en contratos inteligentes que permite un puente con NEAR en una blockchain diferente no puede solicitar bloques por sí mismo. En lugar de eso oracles externos consultan el bloque de cliente ligero siguiente de uno de los nodos completos, y lo envían al contrato inteligente del cliente ligero. El cliente ligero basado en contratos inteligentes realiza las mismas verificaciones mencionadas anteriornmente, por lo que no es necesario confiar en el oracle.

### Prueba de cliente ligero

El siguiente end-point rpc regresa `RpcLightClientExecutionProofResponse` que un cliente ligero necesita para verificar las salidas de ejecución.

Para la salida de ejecución de una transacción, el rpc es

```
http post http://127.0.0.1:3030/ jsonrpc=2.0 method=EXPERIMENTAL_light_client_proof params:="{"type": "transaction", "transaction_hash": <transaction_hash>, "sender_id": <sender_id>, "light_client_head": <light_client_head>}" id="dontcare"
```

Para la salida de ejecución de un recibo, el rpc es

```
http post http://127.0.0.1:3030/ jsonrpc=2.0 method=EXPERIMENTAL_light_client_proof params:="{"type": "receipt", "receipt_id": <receipt_id>, "receiver_id": <receiver_id>, "light_client_head": <light_client_head>}" id="dontcare"
```
