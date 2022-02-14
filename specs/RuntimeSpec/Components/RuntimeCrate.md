# Crate Runtime

El crate Runtime encapsula la lógica de como las transacciones y recibos deberían ser manejadas. Si encuentra
una llamada de contrato inteligente dentro de una transacción o a un recibo llama a `near-vm-runner`, para todas las otras acciones, como creación
de cuenta, las procesa en el momento.

## Clase Runtime

El punto de entrada principar de `Runtime` es el método `apply`.
Aplica una nueva transacción firmada y recibos entrantes para algún fragmento además del
trie dado y la raíz del estado dado.
Si la actualización de las cuentas del validador es proporcionada, actualizar las cuentas del validador.
Todas las transacciones firmadas nuevas deberían se válidas y ya verificadas por el productor de fragmentos.
Si alguna transacción es inválida, el método regresa un `InvalidTxError`.
En caso de éxito, el método regresa `ApplyResult` que contiene el nuevo estado de raíz, cambios en el trie,
nuevos recibos salientes, estadísticas de los validadores (e.j. renta total pagada por todas las cuentas afectadas),
salidas de ejecución.

### Argumentos de Apply

Toma los siguientes argumentos:

- `trie: Arc<Trie>` - el trie que contiene el estado más reciente.
- `root: CryptoHash` - el hash de la raíz del estado en el trie.
- `validator_accounts_update: &Option<ValidatorAccountsUpdate>` - campo opcional que contiene actualizaciones para cuentas de validador.
  Se proporciona al principio de cada epoch y cuando alguien es slasheado.
- `apply_state: &ApplyState` - contiene el índice del bloque y la marca de tiempo, logitud del epoch, precio del gas y el límite de gas.
- `prev_receipts: &[Receipt]` - la lista de os recibos entrantes, del bloque anterior.
- `transactions: &[SignedTransaction]` - la lista de transacciones nuevas firmadas.

### Lógica de Apply

La ejecución consiste de las siguiente etapas:

1. Hace un snapshot del estado inicial.
1. Aplica las actualizaciones de las cuentas de validador, si están disponibles.
1. Convierte transacciones firmadas nuevas en recibos.
1. Procesa los recibos.
1. Revisa que los balances entrantes y salientes cuadren.
1. Finaliza la actualización del trie.
1. Regresa `ApplyResult`.

## Actualización de las cuentas de validador

Las cuentas de validador son cuentas que stakearon algunos tokens para convertirse en un validador.
La actualización de cuentas de validador usualmente pasa cuando el fragmento actual es el primero del epoch.
También pasa cuando hay un reto en el bloque actual con uno de los participantes que pertenece al fragmento actual. 

Esta actualización distribuye las recompensas de validador, regresa tokens bloqueados y tal vez slashee algunas cuentas fuera de su stake.

## Conversión de transacción firmada

Las transacciones de transacciones nuevas firmadas son provistas por el productor de fragmentos en el fragmento. Estas transacciones deberían estar ordenadas y ya estar validadas.
Runtime vuelve a hacer otra validación por las siguientes razones:

- para cobrar a las cuentas por tarifas de transacción, transferencia de balances, gas prepagado y rentas de cuentas;
- para cerar recibos nuevos;
- para calcular el gas quemado;
- para validar las transacciones otra vez, en caso de que el productor de fragmentos sea malicioso.

Si la transacción tiene el mismo `signer_id` y `receiver_id`, entonces el recibo nuevo es agregado a la lista de los recibos locales nuevos,
de otra manera es agregado a la lista de nuevos recibos salientes.

## Procesamiento de recibos

Los recibos son procesados uno por uno en el siguiente orden:

1. Recibos anteriormente retrasados del estado.
1. Nuevos recibos locales.
1. Nuevos recibos entrantes.

Después de cada recibo procesado, comparamos el gas quemado total (hasta ahora) con el límite del gas.
Cuando el gas quemado total alcanza o excede el límite del gas, el procesamiento se detiene.
Los recibos restantes son considerados como retrasados y almacenados en el estado.

### Recibos retrasados

Los recibos retrasados son almacenados como una cola persistente en el estado.
Inicialmente, el primer índice no procesado y el siguiente índice disponible son inicializados en 0.
Cuando un nuevo recibo retrasado es agregado, se escribe bajo el siguiente índice disponible en el estado y el siguiente índice disponible se incrementa por 1.
Cuando un nuevo recibo retrasado es procesado, es leido del estado usando el primer índice no procesado y el primer índice no procesado es incrementado.
Al final del procesamiento de recibo, todos los recibos locales y entrantes restantes son considerados como restrasados y se guardan en el estado en su orden respectivo.
Si durante el procesamiento de recibos, hemos cambiado índices, entonces los ínidices de los recibos retrasados también son guardados en el estado.

### Algoritmo de procesamiento de recibo

El algoritmo de procesamiento de recibo es el siguiente:

1. Leer los índices del estado o inicializarlos con ceros.
1. Mientras que el primer índice no procesado es menor que el próximo índice disponible hacer lo siguiente
   1. Si el gas quemado total es al menos el límite del gas, detener.
   1. Leer el recibo del primero índice no procesado.
   1. Remover el recibo del estado.
   1. Incrementar el primer índice no procesado.
   1. Procesar el recibo.
   1. Agregar el nuevo gas quemado al total de gas quemado.
   1. Recordar que los ínidices de la cola retrasada han cambiado.
1. Procesar los recibos locales nuevos y después los recibos entrantes nuevos
   - Si el total de gas quemado es menor que el límite del gas:
     1. Procesar el recibo.
     1. Agregar el nuevo gas quemado al total de gas quemado.
   - Si no:
     1. Almacenar el recibo bajo el siguiente nuevo índice disponible.
     1. Incrementar el siguiente índice disponible.
     1. Recordar que los ínidices de la cola retrasada han cambiado.
1. Si los índices de la cola retrasada han cambiado, almacena los índices nuevos en el estado.

## Comprobador de balance

El comprobador de balance calcula el total de balance entrante y el total del balance saliente.

El total de balance entrante consite de lo siguiente:

- Recompensas de validador entrantes de la actualización de cuentas de validador.
- Suma de los balances iniciales de las cuentas para todas las cuentas afectadas. Lo calculamos usando el snapshot del estado incial.
- Balances de recibo entrantes. Las tarifas prepagadas y el gas multiplicado
- Incoming receipts balances. Las tarifas de prepago y gas multiplicaron sus precios de gas con los saldos adjuntos de transferencias y llamadas de función.
  Los reembolsos son considerados libres de cargos por tarifas, pero aún así tienen depositos adjuntos.
- Los balances para los recibos retrasados procesados.
- Los balances iniciales para los recibos pospuestos. Los recibos pospuestos son recibos de bloque anteriores que fueron procesados, pero no ejecutados.
  Son recibos de acción con algunos datos entrantes esperados. Usualmente para un callback además de la promesa esperada.
  Cuando los datos esperados llegan después que el recibo de acción, entonces el recibo de acción es pospuesto.
  Nota, los recibos de datos tienen un costo de 0, porque son prepagados cuando se emiten.

El balance total saliente consiste de lo siguiente:

- Suma de el balance final de las cuentas para todas las cuentas afectadas.
- Balances de recibos salientes.
- Nuevos recibos retrasados. Los recibos locales y salientes que no fueron procesados esta vez.
- Balances finales para los recibos pospuestos.
- Renta total pagada por todas las cuentas afectadas.
- Recompensas totales de nuevos validadores. Se calcula a partir de las recompensas totales de gas quemado.
- Balance total quemado. Encaso de que el balance sea quemado por alguna razón (e.j. la cuenta fue eliminada durante un reembolso), se contabiliza ahí.
- Balance total slasheado. En caso de que un validor sea slasheado por alguna razón, el balance se cuenta aquí.

Cuando sumas los balances entrantes y salientes, deben cuadrar.
Si no cuadran, arrojamos un error.
