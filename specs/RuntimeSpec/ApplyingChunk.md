# Applying chunk

## Entradas y salidas

Runtime.apply toma las siguientes entradas:
* trie y la raíz del estado actual
* *validator_accounts_update*
* *incoming_receipts*
* *transactions*

y produce las siguientes salidas:
* raíz del estado nuevo
* *validator_proposals*
* *outgoing_receipts*
* (recibo ejecutado) *outcomes*
* *proof*

## Orden de procesamiento

* Si este es el primer bloque de un epoch, otorgue [reconpensas epoch](../Economics/README.md#validator-rewards-calculation) a los validadores (en el orden de
*validator_accounts_update.stake_info*)
* Barra de saldo bloqueada por comportamiento malicioso (en el orden de *validator_accounts_update.slashing_info*)
* Si la [cuenta tesorera](../Economics/README.md#protocol-treasury) no fue uno de los validadores y este es el primer bloque de un epoch, otorgue un recompensa de la cuenta tesorera
* Si este es el primer bloque de una nueva versión o primer bloque con fragmento de una nueva versión, aplique las migraciones correspondientes
* Si este bloque no tiene fragmento para este bloque, finalice el proceso antes de tiempo
* Procesa [transacciones](Transactions.md) (en el orden de *transacciones*)
* Procesa [recibos](Receipts.md) locales (en el orden de *transacciones* que los generaron)
* Procesa [recibos](Receipts.md) retrasados (ordenados primeramente por el bloque en donde se generaron, después los primeros recibos locales basados en el orden de generación de *transacciones*, después los recibos entrantes, ordenados por el orden que ya tiene *incoming_receipts*)
* Procesa [receipts](Receipts.md) entrantes (ordenados por *incoming_receipts*)

Cuando se procesan los recibos monitoreamos el gas usado (incluyendo el gas usado en las migraciones). Si usamos el límite del gas, imediatamente detenemos el procesamiento de los recibos retrasados, y para los locales y recibos entrantes los agregamos a los recibos retrasados.

* Si alguno de los recibos retrasados fueron procesados o si algún recibo nuevo fue retrasado, se actualizan los índices del primer y último recibo no procesado dentro del estado
* Remueve las propuestas de validador duplicadas (para cada cuenta, solo mantenemos la última en el orden de procesamiento de recibos)

Por favor tenga en cuenta que los recibos locales son recibos generados por transacciones donde el receptor es el mismo que el que firmó la transacción

## Recibos retrasados

En cada bloque tenemos una cantidad máxima de Gas que podemos usar para procesar recibos y migraciones, actualmente es de 500 TGas. Si un recibo local o entrante no es procesado por la falta de Gas, se guardan en el estado con la llave <pre>TrieKey::DelayedReceipt { id }</pre> donde `id` es un índice único para cada fragmento, asignado consecutivamente. Actual mente el emisor no tiene que stakear ningún token para guardar recibos restrasados. El estado también contiene una llave especial <pre>TrieKey::DelayedReceiptIndices</pre> donde el primer y último id de los recibos retrasados aún no procesados son guardados.
