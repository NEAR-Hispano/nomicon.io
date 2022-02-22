# Especificación de Unión

Esta es la interfaz de bajo nivel disponible para el contrato inteligente, consiste de las funciones que el host (representado por
el Wasmer dentro de near-vm-runner) expone al invitado (el contrato inteligente compilado para Wasm).

Debido a las restricciones de Wasm, los métodos operan solo con tipos primitivos, como `u64`.

También para todas las funciones en la especificación de uniones lo siguiente es verdadero:

- La ejecución de método puede resultar en el error `MemoryAccessViolation` si una de las siguientes pasa:
  - El método causa que el host rea una pieza de memoria del invitado pero apunta fuera de la memoria del invitado;
  - El invitado causa que el host lea del registro, pero el id del registro es inválido.

Ejecución de una llamada de función de uniones da como resultado la generación de un error. Este error causa la terminación de la ejecución
del contrato inteligente y el mensaje de error se escribe en los logs de la transacción que causaron la ejecución. Muchas funciones de 
unión puede lanzar mensajes de error especializados, pero hay también una lista de mensajes de error que pueden ser lanzados por casi
cualquier función:

- `IntegerOverflow` -- pasa cuando el invitado pasa alguna información al host pero cuando el host trata de aplicar una operación aritmética
  en el, causa un overflow o underflow;
- `GasExceeded` -- pasa cuando la operación realizada por el invitado causa más gas del gas prepagado restante;
- `GasLimitExceeded` -- pasa cuando la ejecución usa más gas del permitido por el límite global, el cual se impuso en la configuración
  de la economía
- `StorageError` -- pasa cuando el método falla al hacer alguna operación en el trie.

Los siguientes métodos de unión no pueden ser invocados en una llamada view:

- `signer_account_id`
- `signer_account_pk`
- `predecessor_account_id`
- `attached_deposit`
- `prepaid_gas`
- `used_gas`
- `promise_create`
- `promise_then`
- `promise_and`
- `promise_batch_create`
- `promise_batch_then`
- `promise_batch_action_create_account`
- `promise_batch_action_deploy_account`
- `promise_batch_action_function_call`
- `promise_batch_action_transfer`
- `promise_batch_action_stake`
- `promise_batch_action_add_key_with_full_access`
- `promise_batch_action_add_key_with_function_call`
- `promise_batch_action_delete_key`
- `promise_batch_action_delete_account`
- `promise_results_count`
- `promise_result`
- `promise_return`

Si se invocan la ejecución del contrato inteligente entrará en pánico con `ProhibitedInView(<method name>)`.
