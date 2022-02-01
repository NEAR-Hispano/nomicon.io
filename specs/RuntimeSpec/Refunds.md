# Reembolsos

Cuando la ejecución de un recibo falla o hay algún monto que no se ha usado de gas prepagado después de una función de llamada, el tiempo de ejecución genera recibos de reembolso.

Hay 2 tipos de reembolsos.
- Reembolsos por recibo fallido de depósitos ligados. Vamos a llamarles reembolsos de depósitos.
- Reembolsos por el gas no usado y tarifas. Vamos a llamarles reembolsos de gas.

Los recibos de reembolso se identifican por tener `predecessor_id == "system"`. Son también especiales porque no cuesta gas generarlos o ejecutarlos. Como resultado, estos no contribuyen al límite de gas del bloque.

Si la ejecución de un reembolso falla, el monto del reembolsada se quema.
El recibo de reembolso es un `ActionReceipt` que consiste de una sola acción `Transfer` con el monto `deposit` del reembolso.

## Reembolsos de depósito

Los reembolsos de depósito son generados cuando un recibo de acción falla su ejecución. Todos los montos de depósito ligados se suman y
se envían como un reembolso a `predecessor_id`. Porque solo el predecesor puede ligar depósitos.

Los reembolsos de depósito tienen los siguientes campos en el ActionReceipt:
- `signer_id` es `system`
- `signer_public_key` es una llave ED25519 con datos iguales a 32 bytes de `0`.

## Reembolsos de gas

Los reembolsos de gas son generados cuando un recibo usó un monto de gas menor al monto de gas ligado.

Si la ejecución del recibo tiene éxito, el monto de gas es igual a `prepaid_gas + execution_gas - used_gas`.

Si la ejecución del recibo falla, el monto del gas es igual a `prepaid_gas + execution_gas - burnt_gas`. 

La diferencia entre `burnt_gas` y `used_gas` es el `used_gas` que también incluye las tarifas y el gas prepagado de
recibos recientemente creados, e.j. de llamadas cross-contrato en acciones de funciones de llamada.

Luego el monto del gas es convertido a tokens multiplicando por el precio del gas en el que la transacción original fue originada.

Los reembolsos de gas tienen los siguientes campos en el ActionReceipt:
- `signer_id`es el `signer_id` del recibbo que genera este reembolso.
- `signer_public_key` es el `signer_public_key` de el recibo que genera este reembolso.

## Reembolsos del allowance de la llave de acceso

Cuando una cuenta usó una llave de acceso restringida con `FunctionCallPermission`, podría tener un allowance limitado.
El allowance se cargó por el monto completo de las tarifas del recibo incluyendo el gas prepagado completo.
Para reembolsar el allowance distinguimos entre reembolsos de Depósito y reembolsos de Gas usando `signer_id` en el recibo de acción.

Si el `signer_id == receiver_id && predecessor_id == "system"` significa que es un reembolso de gas y el tiempo de ejecución debaría de tratar de reembolsar el allowance.

Note, que no es siempre posible el reembolso del allowance, porque la llave de acceso puede ser borrada entre el momento en que la transacción fue
emitida y cuando el reembolso de gas llegó. En este caso hacemos el mejor esfuerzo en reembolsar el allowance. Esto significa:
- la llave de acceso en la cuenta `signer_id` con la llave de acceso pública `signer_public_key` debe de existir
- el permiso de la llave de acceso debe de ser `FunctionCallPermission`
- el allowance se debe establecer a un valor limitado `Some`, en vez de un allowance ilimitado (`None`)
- el tiempo de ejecución usa una agregación staurada para incrementar el allowance, para así evitar desbordamientos
