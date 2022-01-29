# Llamadas de función
En esta sección les daremos una explicación de como la acción `FunctionCall` funciona, cuáles son las entradas y las salidas. Supongamos que el tiempo de ejecución recibió el siguiente recibo de acción (ActionReceipt):

```rust
ActionReceipt {
     id: "A1",
     signer_id: "alice",
     signer_public_key: "6934...e248",
     receiver_id: "dex",
     predecessor_id: "alice",
     input_data_ids: [],
     output_data_receivers: [],
     actions: [FunctionCall { gas: 100000, deposit: 100000u128, method_name: "exchange", args: "{arg1, arg2, ...}", ... }],
 }
```
### input_data_ids a PromiseResult

`ActionReceipt.input_data_ids` deben de ser satisfechos antes de ser ejecutados (vea [Conciliación de recibos](#receipt-matching)). Cada uno de los `ActionReceipt.input_data_ids` será convertido a `PromiseResult::Successful(Vec<u8>)` si `data_id.data` es `Some(Vec<u8>)` por el contrario si `data_id.data` es `None` la promesa será `PromiseResult::Failed`.

## Entrada
La `FunctionCall` se ejecuta en el ambiente de la cuenta del `receiver_id`.

- un vector de los [Resultados de la promesa](#promise-results) que pueden ser accesados por `promise_result` importando [PromisesAPI](Components/BindingsSpec/PromisesAPI.md)
- Los parámetros `signer_id` y `signer_public_key` de la transacción original que vienen de ActionReceipt (e.j.  `method_name`, `args`, `predecessor_id`, `deposit`, `prepaid_gas` (que es el `gas` en FunctionCall))
- datos generales de una blockchain (e.j. `block_index`, `block_timestamp`)
- leer datos del almacenamiento de la cuenta

Una lista completa de los datos dispobibles para los contratos puede ser encontrada en [Context API](Components/BindingsSpec/ContextAPI.md) y [Trie](Components/BindingsSpec/TrieAPI.md)


## Ejecución

Primero que nada, el tiempo de ejecución prepara el archivo binario Wasm para ser ejecutado:
- carga el código del contrato desde el `receiver_id` del almacenamiento de la [cuenta](../Primitives/Account.md#account)
- deserializa y valida el `código` dentro del archivo binario Wasm (vea `prepare::prepare_contract`)
- inyecta la función que cuenta el gas llamada `gas` que cobrará el gas al principio de cada bloque de código
- instancía la [Especificación de vinculación](Components/BindingsSpec/BindingsSpec.md) con el archivo binario y llama a la función exportada `FunctionCall.method_name`

Durante la ejecución, la máquina virtual hace lo siguiente:

- cuenta el gas quemado en la ejecución
- cuenta el gas usado (que es el `gas usado` + el gas agregado a los nuevos recibos creados)
- cuenta cómo el almacenamiento de la cuenta incremente debido a la llamada
- colecta los logs producidos por el contrato
- establece los datos a regresar
- crea recibos nuevos mediante la [API de promesas](Components/BindingsSpec/PromisesAPI.md)

## Salidas

La salida de `FunctionCall`:

- actualizaciones del almacenamiento - cambia al trie de almacenamiento de la cuenta que será aplicado en una llamada exitosa
- `burnt_gas`, `used_gas` - vea [Tarifas del tiempo de ejecución](Fees/Fees.md)
- `balance` - balance de la cuenta no gastado (el balance de la cuenta podría ser gastado en depositos de `FunctionCall` creados recientemente o [`Acciones de transferencia`](Actions.md#transferaction) a otros contratos)
- `storage_usage` - storage_usage después de la aplicación de ActionReceipt
- `logs` - durante la ejecucón de un contrato, registros que vienen en formato de cadenas de texto del tipo utf8/16 pueden ser creados. Actualmente los logs no son persistidos.
- `new_receipts` - `ActionReceipts` nuevos creados durante la ejecucón. Estos recibos serán enviados al respectivo `receiver_id` (vea [Explicación de la conciliación de recibos](#receipt-matching))
- el resultado puede ser [`ReturnData::Value(Vec<u8>)`](#value-result) o [`ReturnData::ReceiptIndex(u64)`](#receiptindex-result)`


### Valor Resultado

Si el `ActionReceipt` aplicado contiene [`output_data_receivers`](Receipts.md#output_data_receivers), el tiempo de ejecución creará un `DataReceipt` para cada `data_id` y `receiver_id` y `data` será igual al valor regresado. Eventualmente, estos `DataReceipt` serán entregados a sus receptores correspondientes.

### ReceiptIndex Resultado

Un resultado exitoso no podría regresar ningún Valor, por el contrario genera un puñado de ActionReceipts nuevos. Un ejemplo podría ser una callback. En este caso, asumimos que el Recibo nuevo enviará su Valor Resultado a [`output_data_receivers`](Receipts.md#output_data_receivers) del actual `ActionReceipt`.

### Errores

Como con las otras acciones, los errores pueden ser divididos en dos categorías: error de validación y error de ejecución.

#### Error de validación

- Si hay cero gas ligado a la llamada de función, un error
```rust
/// La cantidad de gas ligada en una acción de llamada de función tiene que ser un número positivo.
FunctionCallZeroAttachedGas,
```
será regresado.

- Si la longitud del nombre del método llamado supera a `max_length_method_name`, un parámetro génesis el cual su valor es `256`, un error
```rust
/// La longitud del nombre del método está por encima del límite en una acción FunctionCall
FunctionCallMethodNameLengthExceeded { length: u64, limit: u64 }
```
es regresado.

- Si la longitud del argumento de la llamada de función excede `max_arguments_length`, un parámetro génesis el cual su valor es `4194304` (4MB), un error
```rust
/// La longitud del argumento supera el límite en una acción FunctionCall
FunctionCallArgumentsLengthExceeded { length: u64, limit: u64 }
```
es regresado.

#### Error de ejecución

Pueden haber tres tipos de errores regresados cuando se aplican acciones de llamada de función:
`FunctionCallError`, `ExternalError`, y `StorageError`.

* `FunctionCallError` incluye todo alrededor de la ejecución del archivo binario wasm,
desde compilar el archivo wasm a uno nativo hasta las trampas ocurridas mientras se ejecutan los binarios nativos compilados. Más específicamente
inclute los errores siguientes:
```rust
pub enum FunctionCallError {
    /// Error de compilación de Wasm
    CompilationError(CompilationError),
    /// Error de enlace del entorno binario Wasm
    LinkError {
        msg: String,
    },
    /// Import/export resolve error
    /// Error de resolución de import/export
    MethodResolveError(MethodResolveError),
    /// Ocurrió una trampa durante la ejecución de un binario 
    WasmTrap(WasmTrap),
    WasmUnknownError,
    HostError(HostError),
}
```
- `CompilationError` incluye los errores que pueden ocurrir durante la compilación de un archivo binario wasm.
- `LinkError` es retornado cuando el el tiempo de ejecución del wasmer no puede enlazar el módulo wasm con los imports proporcionados.
- `MethodResolveError` ocurre cuando el método en la acción no puede ser encontrado en le código del contrado.
- `WasmTrap` pasa cuando una trampa ocurre durante la ejecución de un binario. Las trampas incluyen
```rust
pub enum WasmTrap {
    /// Un código de operación `inalcanzable` fue ejecutado.
    Unreachable,
    /// Llamada indirecta 
    /// Trampa de firma incorrecta indirecta de llamada.
    IncorrectCallIndirectSignature,
    /// Trampa de memoria fuera de los límites.
    MemoryOutOfBounds,
    /// Trampa fuera de los límites de la llamada indirecta.
    CallIndirectOOB,
    /// Una excepción aritmética, e.j. dividir entre 0.
    IllegalArithmetic,
    /// Trampa de acceso atómico desalineada.
    MisalignedAtomicAccess,
    /// Trampa de punto de ruptura.
    BreakpointTrap,
    /// Desbordamiento de pila.
    StackOverflow,
    /// Trampa genérica.
    GenericTrap,
}
```
- `WasmUnknownError` ocurre cuando algo dentro del wasmer sale mal
- `HostError` incluye errores que tal vez sean retornados durante la ejecución de una función del host. Esos errores son
```rust
pub enum HostError {
    /// La codificación de la cadena es una secuencia UTF-16 incorrecta
    BadUTF16,
    /// La codificación de la cadena es una secuencia UTF-8 incorrecta
    BadUTF8,
    /// Gas prepagado excedido
    GasExceeded,
    /// Se excedió la cantidad máxima de gas permitida para quemar por contrato
    GasLimitExceeded,
    /// Se excedió el balance de la cuenta
    BalanceExceeded,
    /// Se trató de llamar un nombre de método vacío
    EmptyMethodName,
    /// El contrato inteligente entró en pánico
    GuestPanic { panic_msg: String },
    /// IntegerOverflow pasó durante la ejecución del contrato
    IntegerOverflow,
    /// `promise_idx` no corresponde a las promesas existentes
    InvalidPromiseIndex { promise_idx: u64 },
    /// Las acciones solo pueden agregarse a las promesas no conjuntas
    CannotAppendActionToJointPromise,
    /// Regresar una promesa conjunta actualmente está prohibido
    CannotReturnJointPromise,
    /// Índice resultante de la promesa inválido accesado
    InvalidPromiseResultIndex { result_idx: u64 },
    /// Id inválido de registro accesado
    InvalidRegisterId { register_id: u64 },
    /// Iterador `iterator_index` fue invalidado después de su creación a través de realizar una operación mutable en el trie
    IteratorWasInvalidated { iterator_index: u64 },
    /// Memoria fuera de los límites accesada
    MemoryAccessViolation,
    /// Lógica de la máquina virtual retornó un índice de recibo inválido
    InvalidReceiptIndex { receipt_index: u64 },
    /// El índice de iterador `iterator_index` no existe
    InvalidIteratorIndex { iterator_index: u64 },
    /// La máquina virtual retrnó un id de cuenta inválido
    InvalidAccountId,
    /// La máquina virtual retornó un nombre de método inválido
    InvalidMethodName,
    /// La lógica de la máquina virtual proporcionó una llave pública inválida
    InvalidPublicKey,
    /// `method_name` no es permitido en llamadas view
    ProhibitedInView { method_name: String },
    /// El número total de logs excederá el límite.
    NumberOfLogsExceeded { limit: u64 },
    /// La longitud de la llave de almacenamiento excedió el límite.
    KeyLengthExceeded { length: u64, limit: u64 },
    /// The storage value length exceeded the limit.
    /// La longitud del valor de almacenamiento excedió el límite.
    ValueLengthExceeded { length: u64, limit: u64 },
    /// La longitud de los logs totales excedió el límite.
    TotalLogLengthExceeded { length: u64, limit: u64 },
    /// El número máximo de promesas dentro de una FunctionCall excedió el límite.
    NumberPromisesExceeded { number_of_promises: u64, limit: u64 },
    /// El número máximo de las dependencias de los datos de entrada excedió el límite.
    NumberInputDataDependenciesExceeded { number_of_input_data_dependencies: u64, limit: u64 },
    /// La longitud del valor retornado excedió el límite.
    ReturnedValueLengthExceeded { length: u64, limit: u64 },
    /// El tamaño del contrato para la acción DeployContract excedió el límite.
    ContractSizeExceeded { size: u64, limit: u64 },
    /// La función host fue deprecada
    Deprecated { method_name: String },
}
```


* `ExternalError` inclute los errores que ocurren durante la ejecución dentro de `External`, que es una interface entre el tiempo de ejecución
y el resto del sistema. Los errores posibles son:
```rust
pub enum ExternalError {
    /// Error inesperado que es típicamente relacionado con la corrupción del almacenamiento del nodo.
    /// Es posible que el estado de la entrada sea inválido o malicioso.
    StorageError(StorageError),
    /// Error when accessing validator information. Happens inside epoch manager.
    /// Error cuando accesamos la información del validador. Pasa dentro del manejador de epoch.
    ValidatorError(EpochError),
}
```

* `StorageError` pasa cuando el estado o almacenamiento se corrompen.
