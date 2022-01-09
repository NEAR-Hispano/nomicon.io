# Cuentas

## ID de cuenta

[account_id]: #account_id

El protocolo NEAR tiene un sistema de nombres de cuenta. El ID de la cuenta es similar a un nombre de usuario. Los ID de cuenta tienen que seguir ciertas reglas.

### Reglas de los ID de las cuentas

- La cantidad mínima de caracteres es 2
- La cantidad máxima de caracteres es 64
- El **ID de la cuenta** consiste de las **partes de el ID de la cuenta** separadas por un `.`
- Una **parte del ID de la cuenta** consiste de símbolos alfanuméricos en minúscula separados por un `_` o `-`.
- El **ID de la cuenta** que tiene un largo de 64 caracteres y consiste de caracteres hexadecimales en minúscula es un **ID de cuenta implícito** específico.

Los nombres de cuenta son similares a los dominios de los sitios web.
Las cuentas de nivel superior (TLA por sus siglas en inglés) como `near`, `com`, `eth` solo pueden ser creadas al registrar una cuenta (vea la siguiente sección para más detalles).
Solo `near` puede crear `alice.near`. Y solo `alice.near` puede crear `app.alice.near` y así sucesivamente.
Ojo, `near` near NO PUEDE crear `app.alice.near` directamente.

Adicionalmente, hay un camino implícito para la creación de cuentas. Los id de cuentas, que tienen un largo de 64 caracteres, solo se pueden crear con una `llave de acceso` (AccessKey) que empata el id de la cuenta por medio de derivación `hexadecimal`. Permitiendo así la creación de un nuevo par de llaves – y al remitente de fondos para esta cuenta para realmente crear esta.

Expresión regular para un ID de cuenta completo, sin revisar el largo del mismo:

```regex
^(([a-z\d]+[\-_])*[a-z\d]+\.)*([a-z\d]+[\-_])*[a-z\d]+$
```
### Cuentas de nivel superior (TLA)

| Nombre | Valor |
| - | - |
| REGISTRAR_ACCOUNT_ID | `registrar` |
| MIN_ALLOWED_TOP_LEVEL_ACCOUNT_LENGTH | 32 |

Las cuentas de nivel superior son demasiado valiosas porque estas proveen la raíz de la confianza y el descubrimiento para las compañías, aplicaciones y usuarios.
Para permitir un acceso justo para ellas, los nombres de nivel superior que tienen un número menor de caracteres que el `MIN_ALLOWED_TOP_LEVEL_ACCOUNT_LENGTH` (largo mínimo para una cuenta de nivel superior) serán subastados.

Específicamente, solo las cuentas del tipo `REGISTRAR_ACCOUNT_ID` pueden crear cuentas de nivel superior tienen un número de caracteres menor al `MIN_ALLOWED_TOP_LEVEL_ACCOUNT_LENGTH`. `REGISTRAR_ACCOUNT_ID` implementa el estándar llamado Interfaz de Nombrado de Cuentas (link TODO) para permitir la creación de nuevas cuentas.

```python
def action_create_account(predecessor_id, account_id):
    """Llamado en la acción CreateAccount en el recibo."""
    if len(account_id) < MIN_ALLOWED_TOP_LEVEL_ACCOUNT_LENGTH and predecessor_id != REGISTRAR_ACCOUNT_ID:
        raise CreateAccountOnlyByRegistrar(account_id, REGISTRAR_ACCOUNT_ID, predecessor_id)
    # De lo contrario se crea una cuenta con el `account_id` dado.
```

*Nota: no vamos a implementar la subasta `registrar` al momento del lanzamiento, en lugar de eso se permitirá que Foundation lo implemente después del lanzamiento inicial. El link para los detalles de la subasta serán añadidos aquí en la siguiente edición del post de “next spec release” en MainNet.*

### Ejemplos

Cuentas válidas:

```
ok
bowen
ek-2
ek.near
com
google.com
bowen.google.com
near
illia.cheap-accounts.near
max_99.near
100
near2019
over.9000
a.bro
// Válido pero no puede ser creado, “a” es muy corto
bro.a
```

Cuentas no válidas:

```
not ok           // Los espacios en blanco no son permitidos
a                // Muy corto
100-             // Separador de sufijo
bo__wen          // Dos separadores seguidos
_illia           // Separador de prefijo
.near            // Punto deparador de prefijo
near.            // Punto deparador de sufijo
a..near          // Dos puntos separadores seguidos
$$$              // Los caracteres no alfanuméricos no son permitidos
WAT              // Las mayúsculas no son permitidas
me@google.com    // @ no es permitida (antes sí se permitía)
system           // No se puede usar el nombre de system, vea la sección Cuenta de sistema a continuación
// MUY LARGO:
abcdefghijklmnopqrstuvwxyz.abcdefghijklmnopqrstuvwxyz.abcdefghijklmnopqrstuvwxyz
```

## Cuenta del sistema
`system` es una cuenta especial que solo es usada para identificar recibos de reembolso. Para los recibos de reembolso, tomamos el id del predecessor (predecessor_id) como `system` para indicar que es un recibo de reembolso. Los usuarios no pueden crear o acceder la cuenta `system`. De hecho, esta cuenta no existe como parte del estado del sistema.

## ID implícitos de las cuentas

Las cuentas implícitas funcionan similar a las cuentas de Bitcoin/Ethereum.
Te permite reservar un ID de cuenta antes de ser creado al generar localmente un par de llaves de acceso ED25519.
Este par de llaves de acceso tiene una llave publica que se se asigna al ID de la cuenta. El ID de la cuenta es una representación hexadecimal en minúsculas de la llave pública.
La llave pública ED25519 es de 32 bytes y se asigna a un ID de cuenta con un largo de 64 caracteres.

Ejemplo: Una llave pública en base58 `BGCCDDHfysuuVnaNVtEhhqeT4k9Muyem3Kpgq2U1m9HX` se asignará a al ID de cuenta `98793cd91a3f870fb126f66285808c7e094afcfc4eda8a970f6648cdf0dbd6de`.

La llave de acceso secreta correspondiente te permite firmar transacciones a nombre de esta cuenta una vez que fue creada en la blockchain.

### Creación de cuentas implícitas

Una cuenta con un ID de cuenta implícito solo se puede crear al enviar una transacción/recibo con una sola acción `Transfer` para el ID de cuenta implícito que lo recibirá:
- La cuenta será cread con el ID de la cuenta.
- La cuenta tendrá una llave de acceso completo con la llave pública ED25519-curva que viene de `decode_hex(account_id)` y un nonce de `0`.
- El balance de la cuenta tendrá un saldo de transferencia depositado.

Esta cuenta no puede ser creada con la acción `CreateAccount` para así evitar el robo de la misma cuenta sin tener la llave de acceso privada correspondiente.

Una vez que una cuenta implícita es creada actúa como una cuenta regular hasta que se elimine.

## Cuenta

[account]: #account

Los datos para una sola cuenta son colocados en un solo fragmento. Los datos de la cuenta consisten en lo siguiente:

- Balance
- Balance bloqueado (para el staking)
- El código del contrato
- Almacenamiento llave-valor del contrato. Almacenados en un tipo de árbol ordenado (trie).
- [Llaves de acceso](AccessKey.md)
- [Recibos de acción pospuestos](../RuntimeSpec/Receipts.md#postponed-actionreceipt)
- [Recibos de datos recibidos](../RuntimeSpec/Receipts.md#received-datareceipt)

#### Balances

El balance total de la cuenta consiste del balance bloqueado y del balance desbloqueado.

El balance desbloqueado son tokens que la cuenta puede usar para las cuotas de transacción, transferencias staking y otras operaciones.

El balance bloqueado son los tokens que actualmente están siendo usados para staking, para ser un validador o para convertirse en un validador.
 El balance bloqueado puede convertirse en balance desbloqueado al inicio de un epoch. Vea [Staking] para más detalles.

#### Contratos

Un contrato (AKA smart contract) es un programa escrito en el lenguaje de programación WebAssembly que pertenece a una cuenta en específico.
Cuando una cuenta es creada, no tiene un contrato.
Un contrato tiene que ser implementado explícitamente, ya sea por el dueño de la cuenta o durante la creación de una.
Un contrato puede ser ejecutado por cualquiera que llame un método en tu cuenta. Un contrato tiene acceso al almacenamiento dentro de tu cuenta.

#### Almacenamiento

Toda cuenta tiene su propio almacenamiento. Es un árbol (trie) persistente con estructura llave-valor. Las llaves son ordenadas en orden lexicográfico.
El almacenamiento solo puede ser modificado por el contrato dentro de la cuenta.
La implementación actual, durante el tiempo de ejecución sólo le permite al contrato leer el almacenamiento de tu cuenta, pero esto puede cambiar en el futuro y otros contratos de otras cuentas puedan leer tu almacenamiento.

NOTA: A las cuentas se les cobra una cuota recurrente por el almacenamiento total. Esto incluye el almacenamiento de la misma, el código del contrato, el almacenamiento del contrato y todas las llaves de acceso.

#### Llaves de acceso

Una llave de acceso concede el acceso a una cuenta. Cada llave de acceso en la cuenta se identifica por una llave pública única.
Esta llave pública es usada para validar la firma de las transacciones.
Cada llave de acceso contiene un nonce único para diferenciar u ordenar las transacciones firmadas por esta llave de acceso.

Una llave de acceso tiene un permiso asociado. El permiso puede ser de dos tipos:

- Permiso completo. Concede el acceso completo a la cuenta.
- Permiso para el llamado de funciones. Concede el acceso solo para las transacciones de llamado de funciones.

Vea [Llaves de acceso] para más detalles.
