# Recibo

Toda la comunicación cross-contrato (asumimos que cada cuenta vive en su propio fragmento) en Near pasa a través de Recibos.
Los recibos tienen estado en el sentido en el que no solo sirven como mensajes entre cuentas pero también pueden ser almacenados en el almacenamiento de la cuenta 
para esperar a los DataReceipts.

Cada recibo tiene un [`predecessor_id`](#predecessor_id) (el que lo envió) y un [`receiver_id`](#receiver_id), la cuenta actual.

Los recibos son de dos tipos: recibos de acción o recibos de datos.

Los recibos de datos son recibos que contienen ciertos datas para algunos `ActionReceipt` con el mismo `receiver_id`.
Los recibos de datos tienen 2 campos: el identificador de datos único `data_id` y `data`, el resultado que se recibió.
`data` es un campo de tipo `Option` e indica si el resultado fue un éxito o un fracaso. Si es de tipo `Some`, significa
que la ejecución remota fue exitosa y representa el resultado de un vector de bytes.

Cada `ActionReceipt` también contiene campos relacionados a los datos:

- [`input_data_ids`](#input_data_ids) - un vector de una entrada de datos con los `data_id`s requeridos para la ejecución de este recibo.
- [`output_data_receivers`](#output_data_receivers) - un vector de receptores de datos de salida. Indica a donde enviar los datos salientes.
Cada `DataReceiver` consiste de `data_id` y un `receiver_id` para el enrutamiento.

Antes de que se ejecute un recibo de acción, todas las dependencias de entradas de datos deben ser satisfechas.
Lo que significa que se deben recibir todos los recibos de datos correspondientes.
Si alguno de las dependencias de datos no se encuentra, el recibo de acción se pospone hasta que todas las dependencias de datos faltantes terminan de llegar.

Porque la cadena y el tiempo de ejecución garantizan que ningún recibo se pierde, podemos confiar que cada recibo de acción será ejecutado eventualmente ([Explicación de conciliación de recibos](#receipt-matching)).

Cada `Receipt` tiene los siguientes campos:

#### predecessor_id

- **`tipo`**: `AccountId`

El account_id que emitió el recibo.
En caso de gas o un depósito de reembolso, el ID de cuenta es `system`.

#### receiver_id

- **`tipo`**: `AccountId`

El account_id destino.

#### receipt_id

- **`tipo`**: `AccountId`

Un id único para el recibo.

#### receipt

- **`tipo`**: [ActionReceipt](#actionreceipt) | [DataReceipt](#datareceipt)

Hay dos tipos de recibos: [ActionReceipt](#actionreceipt) y [DataReceipt](#datareceipt). Un `ActionReceipt` es una petición para aplicar [Acciones](Actions.md),
mientras que `DataReceipt` es un resultado de la aplicación de estas acciones.

## ActionReceipt

`ActionReceipt` representa una petición para aplicar acciones en el lado del `receiver_id`. Puede ser derivado como un resultado de una ejecución `Transacción`
u otro procesamiento de un `ActionReceipt`. `ActionReceipt` consite de los siguientes campos:

#### signer_id

- **`tipo`**: `AccountId`

Un account_id que firma la [transacción](Transaction.md) original.
En caso de un depósito de reembolso, el ID de cuenta es `system`.

#### signer_public_key

- **`tipo`**: `PublicKey`

La llave pública de una [Llave de acceso](../Primitives/AccessKey.md) que fue usada para firmar la transacción original.
En caso de un depósito de reembolso, la llave pública está vacía (todos los bytes son 0).

#### gas_price

- **`tipo`**: `u128`

El precio del gas se estableción en el bloque donde la [transacción](Transaction.md) original se aplicó.

#### output_data_receivers

- **`tipo`**: `[DataReceiver{ data_id: CryptoHash, receiver_id: AccountId }]`

Si un contrato inteligente termina su ejecución con algún valor (no Promesa), el tiempo de ejecución crea un [`DataReceipt`] para cada uno de los `output_data_receivers`.

#### input_data_ids

- **`tipo`**: `[CryptoHash]_`

`input_data_ids` son los recibos de dependencia de datos. `input_data_ids` corresponden a `DataReceipt.data_id`.

#### actions

- **`type`**: [`FunctionCall`](Actions.md#functioncallaction) | [`TransferAction`](Actions.md#transferaction) | [`StakeAction`](Actions.md#stakeaction) | [`AddKeyAction`](Actions.md#addkeyaction) | [`DeleteKeyAction`](Actions.md#deletekeyaction) | [`CreateAccountAction`](Actions.md#createaccountaction) | [`DeleteAccountAction`](Actions.md#deleteaccountaction)

## DataReceipt

`DataReceipt` representa el resultado final de la ejecución de algún contrato.

#### data_id

- **`tipo`**: `CryptoHash`

Un identificador `DataReceipt` único.

#### data

- **`tipo`**: `Option([u8])`

Datos asociados en bytes. `None` indica un error durante la ejecución.

# Creando recibos

Los recibos pueden ser generados durante la ejecución de una [Transacción firmada](./Transactions.md#SignedTransaction) (vea este [ejemplo](./Scenarios/FinancialTransaction.md)) 
o durante la aplicación de algún `ActionReceipt` que contiene una acción de tipo [`FunctionCall`](#actions). El resultado de `FunctionCall` puede ser ya sea
un `ActionReceipt` o un `DataReceipt` (datos retornados).

# Conciliación de recibos

El tiempo de ejecución no requiere que los Recibos vengan en un orden en particular. Cada Recibo es procesado individualmente. La meta del procesamiento de `Conciliación de Recibos` es de hacer coincidir todas los [`ActionReceipt`s](#actionreceipt) con los correspondientes [`DataReceipt`s](#datareceipt).

## Procesando ActionReceipt

Para cada [`ActionReceipt`](#actionreceipt) el tiempo de ejecución revisa si tenemos todos los [`DataReceipt`s](#datareceipt) (definido como [`ActionsReceipt.input_data_ids`](#input_data_ids)) requeridos para le ejecución. Si todos los [`DataReceipt`s](#datareceipt) requeridos están ya en el [almacenamiento](#received-datareceipt), el tiempo de ejecución puede aplicar este `ActionReceipt` inmediatamente. De lo contrario guardaremos este recibo como un [ActionReceipt pospuesto](#postponed-actionreceipt). También salvamos el [Contador de DataReceipts pendientes](#pending-datareceipt-count) y [un enlace del `DataReceipt` pendiente a el `ActionReceipt Pospuesto`](#pending-datareceipt-for-postponed-actionreceipt). Ahora el tiempo de ejecución esperará por todos los `DataReceipt`s para aplicar el `ActionReceipt Pospuesto`.

#### ActionReceipt Pospuesto

Un recibo el cual el tiempo de ejecución almacena hasta que el [`DataReceipt`s](#datareceipt) designado llega.

- **`llave`** = `account_id`,`receipt_id`
- **`valor`** = `[u8]`

_Donde `account_id` es [`Receipt.receiver_id`](#receiver_id), `receipt_id` es [`Receipt.receiver_id`](#receipt_id) y el valor es un [`Recibo`](#receipt) serializado (el cual el [tipo](#type) debe de ser [ActionReceipt](#actionreceipt))._

#### Contador de DataReceipt Pendiente

Un contador que cuenta los [`DataReceipt`s](#DataReceipt) pendientes para un [Recibo Pospuesto](#postponed-receipt) inicialmente establecido a la longitud de los [`input_data_ids`](#input_data_ids) faltantes de los `ActionReceipt` entrantes. Se decrementa con cada [`DataReceipt`](#datareceipt) nuevo recibido:

- **`llave`** = `account_id`,`receipt_id`
- **`valor`** = `u32`

_Donde `account_id` es AccountId, `receipt_id` es un CryptoHash y el valor es un entero._

#### DataReceipt Pendiente para ActionReceipt Pospuesto

Indexamos cada `DataReceipt` pendiente para que cada vez que llega un [`DataReceipt`](#datareceipt) nuevo lo conectemos con su [Recibo Pospuesto](#postponed-receipt) al que pertenece.

- **`llave`** = `account_id`,`data_id`
- **`valor`** = `receipt_id`

## Procesando DataReceipt

#### DataReceipt Recibidos

Primero que nada, el tiempo de ejecución guarda los `DataReceipt` entrantes en el almacenamiento como:

- **`llave`** = `account_id`,`data_id`
- **`valor`** = `[u8]`

_Donde `account_id` es [`Receipt.receiver_id`](#receiver_id), `data_id` es [`DataReceipt.data_id`](#data_id) y el valor es un [`DataReceipt.data`](#data) (que es típicamente un resultado serializado de la llamada a un contrato en particular)._

Después, el tiempo de ejecución revisa si hay algún [`ActionReceipt Pospuesto`](#postponed-actionreceipt) esperando por este `DataReceipt` consultando al [`DataReceipt Pendiente` a el Recibo Pospuesto](#pending-datareceipt-for-postponed-actionReceipt). Si no hay un `receipt_id` pospuesto aún, no hacemos nada. Si hay un `receipt_id` pospuesto, hacemos lo siguiente:

- decrementa el [`Contador de Datos Pendientes`](#pending-datareceipt-count) para el `receipt_id` pospuesto.
- remueve el [`DataReceipt Pendiente` encontrado para el `ActionReceipt Pospuesto`](#pending-datareceipt-for-postponed-actionreceipt)

Si el [`Contador de Datos Pendiente`](#pending-datareceipt-count) es 0 significa que todos los [`Receipt.input_data_ids`](#input_data_ids) están almacenados y el tiempo de ejecución puede aplicar el [Recibo Pospuesto](#postponed-receipt) seguramente y lo remueve del almacenamiento.

## Caso 1: Llamada a múltiples contratos y esperar respuestas

Supongamos que el tiempo de ejecución tiene el siguiente `ActionReceipt`:

```python
# Los campos no relevantes son omitidos
Receipt{
    receiver_id: "alice",
    receipt_id: "693406"
    receipt: ActionReceipt {
        input_data_ids: []
    }
}
```

Si la ejecución regresa Result::Value

Supongamos que el tiempo de ejecución recibió el siguiente `ActionReceipt` (usamos un pseudo código parecido a python):

```python
# Los campos no relevantes son omitidos.
Receipt{
    receiver_id: "alice",
    receipt_id: "5e73d4"
    receipt: ActionReceipt {
        input_data_ids: ["e5fa44", "7448d8"]
    }
}
```

No podemos aplicar este recibo inmediatamente: hay DataReceipts faltantes con los ID: ["e5fa44", "7448d8"]. El tiempo de ejecución hace lo siguiente:

```python
postponed_receipts["alice,5e73d4"] = borsh_serialize(
    Receipt{
        receiver_id: "alice",
        receipt_id: "5e73d4"
        receipt: ActionReceipt {
            input_data_ids: ["e5fa44", "7448d8"]
        }
    }
)
pending_data_receipt_store["alice,e5fa44"] = "5e73d4"
pending_data_receipt_store["alice,7448d8"] = "5e73d4"
pending_data_receipt_count = 2
```

_Nota: los Recibos subsecuentes pueden llegar en el bloque actual o el siguiente, es por eso que guardamos los [ActionReceipt Pospuestos](#postponed-actionreceipt) en el almacenamiento_

Luego llega el primer `Pending DataReceipt` pendiente:

```python
# Los campos no relevantes son omitidos.
Receipt {
    receiver_id: "alice",
    receipt: DataReceipt {
        data_id: "e5fa44",
        data: "some data for alice",
    }
}
```

```python
data_receipts["alice,e5fa44"] = borsh_serialize(Receipt{
    receiver_id: "alice",
    receipt: DataReceipt {
        data_id: "e5fa44",
        data: "some data for alice",
    }
};
pending_data_receipt_count["alice,5e73d4"] = 1`
del pending_data_receipt_store["alice,e5fa44"]
```

Y finalmente el último `Pending DataReceipt` llega:

```python
# Los campos no relevantes son omitidos.
Receipt{
    receiver_id: "alice",
    receipt: DataReceipt {
        data_id: "7448d8",
        data: "some more data for alice",
    }
}
```

```python
data_receipts["alice,7448d8"] = borsh_serialize(Receipt{
    receiver_id: "alice",
    receipt: DataReceipt {
        data_id: "7448d8",
        data: "some more data for alice",
    }
};
postponed_receipt_id = pending_data_receipt_store["alice,5e73d4"]
postponed_receipt = postponed_receipts[postponed_receipt_id]
del postponed_receipts[postponed_receipt_id]
del pending_data_receipt_count["alice,5e73d4"]
del pending_data_receipt_store["alice,7448d8"]
apply_receipt(postponed_receipt)
```

## Error de Validación de Recibo

Se realiza una validación de postprocesamiento después de que el recibo de acción es aplicado. La validación incluye:
* Si los recibos generados son válidos. Un recibo generado puede ser inválido, si, por ejemplo, una llamada de función
genera  un recibo para llamar otra función en algún otro contrato, pero el nombre del contrato en inválido. Aquí están
los dos tipos de errores principales:
- el id de cuenta es inválido. Si el id receptor del recibo es inválido, un error
```rust
/// El `receiver_id` de un Recibo no es válido.
InvalidReceiverId { account_id: AccountId },
``` 
es regresado.
- alguna acción es inválida. Los errores regresados aquí son los mismos que los errores de validación mencionados en [acciones](Actions.md).
* Si la cuenta todavía tiene el balance suficiente para pagar por el almacenamiento. Si, por ejemplo, la ejecución de una acción de llamada de función
lleva a algunos recibos que requiere que genere transferencia para ser generado como un resultado, la cuenta tal vez no tenga el suficiente
balance después de que la cantidad transferida es deducida. En este caso, un
```rust
/// ActionReceipt can't be completed, because the remaining balance will not be enough to cover storage.
/// ActionReceipt no puede ser completado, porque el balance restante no será suficiente para cubrir el almacenamiento.
LackBalanceForState {
    /// An account which needs balance
    account_id: AccountId,
    /// Balance required to complete an action.
    amount: Balance,
},
```
