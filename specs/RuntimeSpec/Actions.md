# Acciones

Hay una variedad de tipos de acción en Near:

```rust
pub enum Action {
    CreateAccount(CreateAccountAction),
    DeployContract(DeployContractAction),
    FunctionCall(FunctionCallAction),
    Transfer(TransferAction),
    Stake(StakeAction),
    AddKey(AddKeyAction),
    DeleteKey(DeleteKeyAction),
    DeleteAccount(DeleteAccountAction),
}
```

Cada transacción consiste de una lista de acciones a ser realizadas en el lado del `receiver_id`. Dado que las transacciones son convertidas
a recibos primero cuando estas son procesadas, nos preocuparemos principalmente con las acciones en el contexto del procesamiento
de recibos.
 
Para las siguientes acciones, se requiere que `predecessor_id` y `receiver_id` sean iguales:
- `DeployContract`
- `Stake`
- `AddKey`
- `DeleteKey`
- `DeleteAccount`

NOTA: si la primera acción en la lista de acciones es `CreateAccount`, `predecessor_id` se vuelve `receiver_id`
para el resto de las acciones hasta llegar a `DeleteAccount`. Esto da el permiso a otra cuenta para actuar en la cuenta recién creada.

## CreateAccountAction

```rust
pub struct CreateAccountAction {}
```

Si `receiver_id` tiene una longitud de 64, este id de cuanta será considerado como `hex(public_key)`, lo que significa que la creación de la cuenta solo tiene 
éxito si esta viene seguida de la acción `AddKey(public_key)`.

**Resultados**:
- crea una cuenta con `id` = `receiver_id`
- establece el `storage_usage` de una cuenta a `account_cost` (configuración de génesis)

### Errores

**Errores de ejecución**:
- Si la acción trata de crear una cuenta de nivel alto la cual la longitud no es mayor a 32 carácteres, y `predecessor_id` no es
`registrar_account_id`, el cual es definido por el protocolo, el siguiente error será regresado
```rust
/// Un ID de cuenta de alto nivel solo puede ser creado por registrar.
CreateAccountOnlyByRegistrar {
    account_id: AccountId,
    registrar_account_id: AccountId,
    predecessor_id: AccountId,
}
```

- Si la acción trata de crear una cuenta de alto nivel o una subcuenta de `predecessor_id`,
el siguiente error será regresado
```rust
/// Una cuenta recién creada debe estar bajo un espacio de nombres de la cuenta del creador
CreateAccountNotAllowed { account_id: AccountId, predecessor_id: AccountId },
```

## DeployContractAction

```rust
pub struct DeployContractAction {
    pub code: Vec<u8>
}
```

**Salida**:
- establece el código del contrato para la cuenta

### Errores

**Error de validación**:
- si la longitud de `code` excede `max_contract_size`, el cual es un parámetro génesis, el siguiente error será regresado:
```rust
/// El tamaño del código del contrato excede el límite en una acción DeployContract.
ContractSizeExceeded { size: u64, limit: u64 },
```

**Error de ejecución**:
- Si el estado o el almacenamiento se corrompe, tal vez regrese `StorageError`.

## FunctionCallAction

```rust
pub struct FunctionCallAction {
    /// Nombre de la función Wasm exportada
    pub method_name: String,
    /// Argumentos serializados
    pub args: Vec<u8>,
    /// Gas prepagado (gas_limit) para una llamada de función
    pub gas: Gas,
    /// Cantidad de tokens a transferir a receiver_id
    pub deposit: Balance,
}
```

Llama a un método de un contrato en particular. Vea los [detalles](./FunctionCall.md).

## TransferAction

```rust
pub struct TransferAction {
    /// Cantidad de tokens a transferir a receiver_id
    pub deposit: Balance,
}
```

**Salida**:
- transfiere el monto especificado en `deposit` de `predecessor_id` a una cuenta `receiver_id`

### Errores

**Error de ejecución**:
- Si el monto del depósito más el monto existente en la cuenta receptora excede `u128::MAX`,
un error `StorageInconsistentState("Account balance integer overflow")` será regresado.

## StakeAction

```rust
pub struct StakeAction {
    // Cantidad de tokens a stakear
    pub stake: Balance,
    // Esta llave pública es una llave pública del nodo validador
    pub public_key: PublicKey,
}
```

**Salida**:
- Una propuesta del validador que contiene la llave de acceso pública del staking y el monto dirigido al staking es generado y será incluído
en el próximo bloque.

### Errores

**Errores de validación**:
- Si `public_key` no es una clave ed25519 compatible con ristretto, el siguiente error será regresado:
```rust
/// Intento de staking con una llave pública que no es convertible a ristretto.
UnsuitableStakingKey { public_key: PublicKey },
```

**Errores de ejecución**:
- Si una cuenta no ha hecho staking pero trata de hacer unstaking, el siguiente error será regresado:
```rust
/// La cuenta no ha sido stakeada, pero trata de unstakear
TriesToUnstake { account_id: AccountId },
```

- Si una cuenta trata de stakear más de la cantidad de tokens que tiene, el siguiente error será regresado:
```rust
/// La cuenta no tiene el balance suficiente para incrementar el stake.
TriesToStake {
    account_id: AccountId,
    stake: Balance,
    locked: Balance,
    balance: Balance,
}
```

- Si la cantidad stakeada está por debajo del umbral mínimo de stake, el siguiente error será regresado:
```rust
InsufficientStake {
    account_id: AccountId,
    stake: Balance,
    minimum_stake: Balance,
}
```
El stake mínimo es determinado por `last_epoch_seat_price / minimum_stake_divisor` donde `last_epoch_seat_price` es el
precio asiento determinado al final del último epoch y `minimum_stake_divisor` es un parámetro génesis de configuración
y su valor actual es 10.

## AddKeyAction

```rust
pub struct AddKeyAction {
    pub public_key: PublicKey,
    pub access_key: AccessKey,
}
```

**Salidas**:
- Agrega una [Llave de acceso](AccessKey) a la cuenta receptora y la asocia con una `public_key`proporcionada.

### Errores:

**Errores de validación**:

Si la llave de acceso es de tipo `FunctionCallPermission`, los siguientes errores pueden ocurrir
- Si `receiver_id` en la `access_key` no es un ID de cuenta válido, el siguiente error será regresado
```rust
/// ID de cuenta inválido
InvalidAccountId { account_id: AccountId },
```

- Si la longitud de algún nombre de método excede `max_length_method_name`, que es un parámetro génesis (su valor actual es 256),
el siguiente error será regresado
```rust
/// La longitud de algún nombre de método excede el límite de la acción Add Key
AddKeyMethodNameLengthExceeded { length: u64, limit: u64 },
```

- Si la suma de la longitud de los nombres de método (con un carácter extra para cada nombre de método) excede `max_number_bytes_method_names`, que es un parámetro génesis (su valor actual es 2000),
el siguiente error será regresado
```rust
/// El número total de bytes de los nombres de método excedió el límite de la acción Add Key
AddKeyMethodNamesNumberOfBytesExceeded { total_number_of_bytes: u64, limit: u64 }
```

**Errores de ejecución**:
- Si una cuenta trata de añadir una llave de acceso con una llave pública dada, pero una llave de acceso existente con esta llave pública ya existe, el siguiente error será regresado
```rust
/// La llave pública ya fue usada por una clave de acceso existente
AddKeyAlreadyExists { account_id: AccountId, public_key: PublicKey }
```
- Si el estado o el almacenamiento se corrompe, un error de tipo `StorageError` será regresado.

## DeleteKeyAction

```rust
pub struct DeleteKeyAction {
    pub public_key: PublicKey,
}
```

**Salidas**:
- Elimina la [llave de acceso](AccessKey) asociada con la `public_key`.

### Errores

**Errores de ejecución**:

- Cuando una cuenta trata de borrar una llave de acceso que no existe, el siguiente error es regresado
```rust
/// La cuenta trata de remover una llave de exceso que no existe
DeleteKeyDoesNotExist { account_id: AccountId, public_key: PublicKey }
```
- `StorageError` es regresado si el estado o el almacenamiento se corrompe.

## DeleteAccountAction

```rust
pub struct DeleteAccountAction {
    /// El balance de la cuenta restante será transferido a este AccountId
    pub beneficiary_id: AccountId,
}
```

**Salidas**:
- La cuenta, y todos los datos guardados dentro de esta misma, será borrada y los tokens serán transferidos a `beneficiary_id`.

### Errores

**Errores de validación**
- Si `beneficiary_id` no es un id de cuenta válido, el siguiente error será regresado
```rust
/// ID de cuenta inválido.
InvalidAccountId { account_id: AccountId },
```

- Si esta acción no es la última en la lista de la acciones de un recibo, el siguiente error será regresado
```rust
/// La acción eliminar debe de ser la acción final en una transacción
DeleteActionMustBeFinal
```

- Si la cuenta todavía tiene balance bloqueado debido al staking, el siguiente error será regresado
```rust
/// La cuenta se encuentra stakeando y no puede ser eliminada
DeleteAccountStaking { account_id: AccountId }
```

**Errores de ejecución**:
- Si el estado o el almacenamiento se corrompen, un error de tipo `StorageError` es regresado.
