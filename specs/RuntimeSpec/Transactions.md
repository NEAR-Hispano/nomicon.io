# Transacciones

Una transacción en Near es una lista de [acciones](Actions.md) e información adicional:

```rust
pub struct Transaction {
    /// Una cuenta en cuyo nombre se firma la transacción
    pub signer_id: AccountId,
    /// Una llave de acceso que fue usada para firmar la transacción
    pub public_key: PublicKey,
    /// Nonce es usado para determinar el orden de transacción en el grupo.
    /// Se incrementa por una combinación de `signer_id` y `public_key`
    pub nonce: Nonce,
    /// Cuenta receptora para esta transacción. Si
    pub receiver_id: AccountId,
    /// El hash del bloque en la blockchain sobre la cual la transacción dada es válida
    pub block_hash: CryptoHash,
    /// Una lista de acciones a ser aplicada
    pub actions: Vec<Action>,
}
```

## Transacciones firmadas

`SignedTransaction` es lo que un nodo recibe de una billetera a través de un endpoint JSON-RPC y después al fragmento en donde `receiver_id` reside. La firma prueba la propiedad de la `public_key` correspondiente (que es una llave de acceso para una cuenta en particular) así como la autenticidad de la transacción en sí.

```rust
pub struct SignedTransaction {
    pub transaction: Transaction,
    /// La fima de un hash de una Transacción serializada Borsh
    pub signature: Signature,
```

Dale un vistazo a algunos [escenarios](Scenarios/Scenarios.md) sobre como las transacciones pueden ser aplicadas.

## Transacciones por lotes

Una `Transacción` puede contener una lista de acciones. Cuando hay más de una acción en una transacción, nos referimos a esta
transacción como transacción en lote. Cuando dicha transacción es aplicada, es equivalente a aplicar cada una de las acciones
por separado, excepto por: 
* Despúes de procesar una acción de tipo `CreateAccount`, el resto de la acción es aplicada en el nombre de la cuenta que apenas fue creada.
Esto permite a una, en una transacción, crear una cuenta, desplegar un contrato en la cuenta, y llamar alguna función de inicialización
en el contrato.
* La acción `DeleteAccount`, si está presente, debe de ser la última acción en la transacción.

El número de acciones en una transacción está limitada por `VMLimitConfig::max_actions_per_receipt`, del cual el valor actual
es 100.

## Transacciones, Validaciones y Errores

Cuando una transacción es recibida, varias revisiones serán realizadas para asegurar su validez. Esta sección enlista las revisiones
y potenciales errores a ser retornados cuando fallan.

### Validación básica

Una validación básica de una transacción puede ser realizada sin el estado. Dicha validación incluye
- Si `signer_id` es válido. Si no, un error
```rust
/// TX signer_id no está en un formato válido o no satisface los requerimientos, vea `near_core::primitives::utils::is_valid_account_id`
InvalidSignerId { signer_id: AccountId },
```
es regresado.
- Si `receiver_id` es válido. Si no, un error
```rust
/// TX receiver_id no está en un formato válido o no satisface los requerimientos, vea `near_core::primitives::utils::is_valid_account_id`
InvalidReceiverId { receiver_id: AccountId },
```
es regresado.
- Si `signature` es firmado por `public_key`. Si no, un error
```rust
/// TX signature no es válido
InvalidSignature
```
es regresado.
- Si el número de acciones incluídas en la transacción es mayor que `max_actions_per_receipt`. Si no, un error
```rust
 /// El número de acciones excede el límite dado.
TotalNumberOfActionsExceeded { total_number_of_actions: u64, limit: u64 }
```
es regresado.
- Dentro de las acciones en la transacción, si `DeleteAccount` está presente, es la última acción. Si no, un error
```rust
/// La acción eliminar debe ser una acción final en una transacción
DeleteActionMustBeFinal
```
es regresado.
- Si el gas prepagado total no excede `max_total_prepaid_gas`. Si no, un error
```rust
/// El gas prepagado total (para todas las acciones dadas) excedió el límite.
TotalPrepaidGasExceeded { total_prepaid_gas: Gas, limit: Gas }
```
es regresado.
- Si cada acción incluída es válida. Los detalles de esta revisión pueden encontrarse en [acciones](Actions.md).

### Validación con el estado

Después de que la validación básica es terminada, verificamos la transacción con el estado actual para realizar una validación adicional. Esto incluye
- Si `signer_id` existe. Si no, un error
```rust
/// TX signer_id no fue encontrado en el almacenamiento
SignerDoesNotExist { signer_id: AccountId },
```
es regresado.

- Si el nonce de la transacción es mayor que el nonce existente en la llave de acceso. Si no, un error
```rust
/// El nonce de la transacción debe ser account[access_key].nonce + 1
InvalidNonce { tx_nonce: Nonce, ak_nonce: Nonce },
```
error es regresado.

- Si la cuenta `signer_id` tiene balance suficiente para cubrir el costo de la transacción. Si no, un error
```rust
 /// La cuenta no tiene el balance suficiente para cubrir el costo de la TX
NotEnoughBalance {
    signer_id: AccountId,
    balance: Balance,
    cost: Balance,
}
```
error es regresado.

- Si la transacción es firmada por una llave de acceso para llamadas de función y la esta no tiene el allowance suficiente
para cubrir el costo de la transacción, un error
```rust
/// La llave de acceso no tiene el allowance suficiente para cubrir el costo de la transacción
NotEnoughAllowance {
    account_id: AccountId,
    public_key: PublicKey,
    allowance: Balance,
    cost: Balance,
}
```
error es regresado.

- Si la cuenta `signer_id` no tiene el balance suficiente para cubrir su almacenamiento después de pagar el costo de la transacción, un error
```rust
/// La cuenta que firma no tiene el balance suficiente después de la transacción.
LackBalanceForState {
    /// Una cuenta que no tiene el balance suficiente para cubrir el almacenamiento.
    signer_id: AccountId,
    /// Balance requerido para cubrir el estado.
    amount: Balance,
}
```
error es regresado.

- Si una transacción es firmada por una llave de acceso para llamadas de función, los siguientes errores son posibles:
* `InvalidAccessKeyError::RequiresFullAccess` si la transacción contiene más de una acción o si la única acción que contiene
no es una acción del tipo `FunctionCall`.
* `InvalidAccessKeyError::DepositWithFunctionCall` si la acción de llamada de función tiene un `deposit` diferente de 0.

```rust
/// El `receiver_id` de la transacción no cuadra con la llave de acceso del receiver_id
InvalidAccessKeyError::ReceiverMismatch { tx_receiver: AccountId, ak_receiver: AccountId },
```
es retornado cuando el `receiver_id` de la transacción no cuadra con el `receiver_id` de la llave de acceso.
*
```rust
/// El nombre del método de la transaccón no esta permitido por la llave de acceso
InvalidAccessKeyError::MethodNameMismatch { method_name: String },
```
es retornado si el nombre del método que la transacción trata de llamar no es permitido por la llave de acceso.
