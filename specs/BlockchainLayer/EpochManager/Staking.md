# Staking y slashing

## Invariante Stake
`Account` tiene dos campos representando sus tokens: `amount` y `locked`. `amount + locked` es el número total de
tokens que una cuenta tiene: las acciones de bloqueo/desbloqueo involucra transferencia de balance entre los dos campos, y el
slashing es realizado al restar del valor `locked`.

En una acción stake el balance se bloquea inmediatamente (el balance bloqueado solo puede incrementar), y la propuesta de stake es
pasado al epoch manager. Las propuestas se acumulan durante un epoch y se procesan todas de una cuando un epoch es finalizado.
El desbloqueo solo pasa al inicio de un epoch.

El stake de la cuenta es definido por epoch y es almacenado en los `validators` `EpochInfo` y los conjuntos `fishermen`. `locked`
siempre es igual al máximo de los últimos tres stakes y la propuesta más alta en el epoch actual.


### Regresando el stake
`locked` es el número de tokens bloqueados para staking, se calcula de siguiente manera:
- inicialmente es el valor en génesis o `0` para las nuevas cuentas
- en una propuesta de staking con un valor más alto que `locked`, se incrementa a ese valor
- al inicio de cada epoch se recalcula:
    1. considera los 3 epochs más recientes
    2. para las cuentas no slasheadas, toma el máximo de sus stakes en esos epochs
    3. si una cuenta hizo una propuesta en el bloque que empieza el epoch, también toma el máximo con el valor de la propuesta
    4. cambia `locked` al valor resultante (y actualiza `amount` para que `amount + locked` se quede igual)

### Slashing
TODO.

