# Token Fungible ([NEP-141](https://github.com/near/NEPs/issues/141))

Versión `1.0.0`

## Resumen

Una interface estándar para tokens fungibles que acepta transferencias normales así como tambien Una interfaz estándar para tokens fungibles que permite una transferencia normal, así como una llamada de método y transferencia en una sola transacción. El [estándar de almacenamiento](../StorageManagement.md) aborda las necesidades (y la seguridad) del stakeo del almacenamiento.
El [estándar de metadata de tokens fungibles](Metadata.md) proviciona los campos necesarios para la ergonomía en las dApps y maketplaces.

## Motivación

El Protocolo NEAR usa un tiempo de ejecución fragemtado y asíncrono. Esto significa lo siguiente:
 - El almacenamiento para los diferentes contratos y cuentas pueden ser localizados en los diferentes fragmentos.
 - Dos contratos pueden ser ejecutados al mismo tiempo en diferentes fragmentos.

Si bien esto aumenta el rendimiento de la transacción de forma lineal con la cantidad de fragmentos, también crea algunos retos para el desarrollo de cross-contratos. Por ejemplo, si un contrato quiere consultar alguna información del estado de otro contato (e.j. balance actual), para el tiempo en el que el primero contrato reciba el balance, el balance real puede cambiar. En un sistema tan asíncrono, un contrato no puede confiar en el estado de otro contrato y asumir que no va a cambiar.

Al contrario el contrato puede confiar en un bloqueo parcial temporal del estado con un método callback para actuar o desbloquear, pero requiere una ingeniería cuidadosa para evitar bloqueos mutuos. En este estándar estamos tratando de evitar la aplicación de bloqueos. El abordamiento típico a este problema incluye un sistema escrow con allowances. Este enfoque fue inicialmente desarrollado para [NEP-21](https://github.com/near/NEPs/pull/21) que es similar al estándar ERC-20 de Ethereum. Hay algunos problemas al usar escrow como la única forma de pago por un servicio con token fungible. Esto requiere frecuentemente más de una transacción para escenarios comunes donde los tokens fungibles son dados como pago con la expectativa de que posteriormente un método será llamado.

Por ejemplo, un contrato oracle podría ser pagado con tokens fungibles. Un contrato cliente que desea usar el oracle (oráculo) debe de incrementar el allowance del escrow antes de cada petición al contrato oracle, o alojar un allowance grande que cubra múltiples llamadas. Ambos tienen inconvenientes y últimadamente sería ideal poderenviar tokens fungibles y llamar a un método en una sola transacción. Esta preocupación es abordada en el método `ft_transfer_call`. El poder de esto viene del contrato receptor que trabaja en concierto con el contrato del token fungible en una manera segura. Esto es, si un contrato receptor cumple con el estándar, una sola transacción podría transferir y llamar a un método.

Nota: no hay razón para que un sistema escrow no pueda ser incluído en una implementación de tokens fungibles, pero simplemente no es necesario en el estándar básico. La lógica escrow debería ser movida a un contrato separado para manjear esa funcionalidad. Una razón para eso es porque el [Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) transferirá tokens fungibles de Ethereum a NEAR, donde el locker de tokens (una fábrica) utilizará el estándar central de tokens fungibles.

Estado de la técnica:

- [ERC-20 standard](https://eips.ethereum.org/EIPS/eip-20)
- Estándar NEP#4 NEAR NFT: [near/neps#4](https://github.com/near/neps/pull/4)

Aprenda sobre NEP-141:

- [Figment Learning Pathway](https://learn.figment.io/network-documentation/near/tutorials/1-project_overview/2-fungible-token)

## Explicación a nivel de guía:

Debaríamos poder hacer lo siguiente:
- Inicializar el contrato una vez. El suministro total dado será propiedad del ID de cuento dado.
- Obtener el suministro total.
- Tranferir tokens al usuario nuevo.
- Transferir tokens de un usuario a otro.
- Transferir tokens a un contrato, tenemos el contrato receptor llamando a un método y "regresa" los tokens fungible no usado.
- Remueve el estado del par llave/valor correspondiente con la cuenta de un usuario, retirando un balance nominal de Ⓝ que fue usado para el almacenamiento.

Hay algunos conceptos en los escenarios anteriores:
- **Suministro total**: el número total de tokens en circulación.
- **Balance owner**: un ID de cuenta que posee alguna cantidad de tokens. 
- **Balance**: una cantidad de tokens.
- **Transferencia**: una acción que mueve un monto de una cuenta a otra, puede ser una cuenta 
- **Transferencia y llamada**: una acción que mueve una cantidad de una cuenta a una cuenta contrato donde el receptor llama a un método.
- **Cantidad de almacenamiento**: la cantidad del almacenamiento usado para que una cuenta sea "registrada" en el token fungible. Esta cantidad está denominada en Ⓝ, no en bytes, y representa el [almacenamiento stakeado](https://docs.near.org/docs/concepts/storage-staking).

Note que la precisión (el número de lugares decimales soportados por un token) no es parte de este estándar básico, ya que no es requerido para realizar acciones. El valor mínimo es siempre 1 token. Vea el [Estándar de metadata de token fungible](Metadata.md) para aprender a admitir precisión/decimales de forma estandarizada.

Dado que múltiples usuarios usarán un contrato de Token Fungible, y su actividad resultará en un [stakeo de almacenamiento](https://docs.near.org/docs/concepts/storage-staking) incrementado para la cuenta del contraro, este estándar está diseñado para interoperar muy bien con [el Estándar de almacenamiento](../StorageManagement.md) para depósitode de almacenamiento y reembolsos.

### Escenarios de ejemplo

#### Transferencia simple

Alice quiere enviar 5 tokens wBTC a Bob.

**Suposiciones**

- El contrato de token wBTC es `wbtc`.
- La cuenta de Alice es `alice`.
- La cuenta de Bob es `bob`.
- La precisión ("decimales" en el estándar de metadata) en el contrato wBTC es de `10^8`.
- Los 5 tokens son `5 * 10^8` o como número son `500000000`.

**Explicación de alto nivel**

Alice necesita emitir una transacción a un contrato wBTC para transferir 5 tokens (multiplicados por la precisión) a Bob.

**Llamadas técnicas**

1. `alice` llama a `wbtc::ft_transfer({"receiver_id": "bob", "amount": "500000000"})`.

#### Depósito de tokens a un contrato

Alice quiere depositar 1000 tokens DAI a un contrato de interés compuesto para ganar tokens extra.

**Suposiciones**

- El contrato de token DAI es `dai`.
- La cuenta de Alice es `alice`.
- El interés compuesto del contrato es `compound`.
- La precisión ("decimales" en el estándar de metadata) en el contrato DAI es de `10^18`.
- Los 1000 tokens son `1000 * 10^18` o como número son `1000000000000000000000`.
- El contrato compuesto puede trabajar con múltiples tipos de token.

<details style="background-color: #000; padding: 3px; color: #fff">
<summary>Para este ejemplo, puede expandir esta sección para ver como un estándar de token fungible anterior que usa escrows lidiaría con el escenario.</summary>
<hr/>

**Explicación de alto nivel** (NEP-21 standard)

Alice necesita emitir 2 transacciones. La primera a `dai` para establecer un allowance a `compound` para que pueda retirar tokens de `alice`.
La segunda transacción es para que `compound` empiece el proceso de depósito. Compound revisará que los tokens DAI son soportados y tratará de retirar el monto de DAI deseado de `alice`.
- Si la transferencia tiene éxito, `compuesto` puede aumentar la propiedad local de `alice` a 1000 DAI.
- Si la transferencia falla, `compound` no necesita de hacer nada en el ejemplo actual, pero tal vez pueda notificar a `alice` de una transferencia fallida.

**Llamadas técnicas** (NEP-21 standard)

1. `alice` llama a `dai::set_allowance({"escrow_account_id": "compound", "allowance": "1000000000000000000000"})`.
2. `alice` llama a `compound::deposit({"token_contract": "dai", "amount": "1000000000000000000000"})`. Dueante la llamada `deposit`, `compound` hace lo siguiente:
   1. hace una llamada asíncrona a `dai::transfer_from({"owner_id": "alice", "new_owner_id": "compound", "amount": "1000000000000000000000"})`.
   2. liga un callback a `compound::on_transfer({"owner_id": "alice", "token_contract": "dai", "amount": "1000000000000000000000"})`.
<hr/>
</details>

**Explicación de alto nivel**

Alice necesita emitir una transacción, a diferencia del típico flujo de trabajo escrow que emite 2.

**Llamadas técnicas**

1. `alice` llama a `dai::ft_transfer_call({"receiver_id": "compound", "amount": "1000000000000000000000", "msg": "invest"})`. Durante la llamada `ft_transfer_call`, dai hace lo siguiente:
   1. hace una llamada asíncrona a `compound::ft_on_transfer({"sender_id": "alice", "amount": "1000000000000000000000", "msg": "invest"})`.
   2. liga un callback a `dai::ft_resolve_transfer({"sender_id": "alice", "receiver_id": "compound", "amount": "1000000000000000000000"})`.
   3. compound termina de invertir, usando los tokens fungible ligados `compound::invest({…})`, después regresa el valor de los tokens que no fueron usados o necesitados. En este caso, Alice pidió que los tokens fueran invertidos, así que regresará 0. (En algunos casos un método tal vez no necesite usar todos los tokens fungibles, y regrese el restante)
   4. La función `dai::ft_resolve_transfer` recibe éxito/fallo de la promesa. Si tiene éxito, contendrá los tokens no usados. Entonces el contrato `dai` usa aritmética simple (no se necesita en este caso) y actualiza el balance de Alice.

#### Intercambiar un token por otro a través de un Creador de Mercado Automatizado (AMM por sus siglas en inglés) como Uniswap

Alice quiere intercambiar 5 NEAR envueltos (wNEAR) por tokens BNNA a la tasa de mercado actual, con menos del 2% de slippage.

**Suposiciones**

- El contrato de token wNEAR es `wnear`.
- La cuenta de Alice es `alice`.
- El contrato de AMM es `amm`.
- El contrato de BNNA es `bnna`.
- La presición ("decimales" en el estándar de metadata) en el contrato wNEAR es de `10^24`.
- Los 5 tokens son `5 * 10^24` o como número es `5000000000000000000000000`.

**Explicación de alto nivel**

Alice necesita emitir una transacción a un contrato wNEAR para transferir 5 tokens (multiplicados por la precisión) a `amm`, especificando la acción deseada (intercambio), el token al que se quiere cambias (BNNA) y el slippage mínimo (<2%) en `msg`.

Alice probablemente hará esta llamada a través de una interface de usuario que sabe como construir un `msg` de una manera que el contrato `amm` entenderá. Sin embargo, es posible que el contrato `amm` provea funciones view que tomen la acción deseada, el token que se quiere y el slippage como entrada y datos de regreso ya listos para pasar a `msg` para `ft_transfer_call`. Para seguir con este ejemplo, digamos que `amm` implementa una función view llamada `ft_data_to_msg`.

Alice necesita agregar un yoctoNEAR. Esto resultará en ella viendo la página de confirmación en su billetera de NEAR preferida. Las implementaciones de billeteras NEAR proveerán (eventualmente) información útil en esta página de confirmación, así los contratos receptores deberán seguir un estándar estricto en como le dan formato a `msg`. Actualizaremos este documento con una documentación, a medida que surja el consenso de la comunidad.

Entonces, Alice puede dar dos pasos, aunque el primero puede ser un detalle que la aplicación que ella usa pide.

**Llamadas técnicas**

1. Vea `amm::ft_data_to_msg({ action: "swap", destination_token: "bnna", min_slip: 2 })`. Usando [NEAR CLI](https://docs.near.org/docs/tools/near-cli):

      near view amm ft_data_to_msg '{
        "action": "swap",
        "destination_token": "bnna",
        "min_slip": 2
      }'

   Luego, Alice (o la aplicación que usa) conservará el resultado y lo usará en el siguiente paso. Digamos que este resultado es `"swap:bnna,2"`.

2. Llama a `wnear::ft_on_transfer`. Usando NEAR CLI:
       near call wnear ft_transfer_call '{
         "receiver_id": "amm",
         "amount": "5000000000000000000000000",
         "msg": "swap:bnna,2"
       }' --accountId alice --depositYocto 1

   Durante la llamada a `ft_transfer_call`, `wnear` hace lo siguiente:
   
   1. Decrementa el balance de `alice` e incrementa el balance de `amm` por 5000000000000000000000000.
   2. Hace una llamada asincrónica a `amm::ft_on_transfer({"sender_id": "alice", "amount": "5000000000000000000000000", "msg": "swap:bnna,2"})`.
   3. Agrega un callback `wnear::ft_resolve_transfer({"sender_id": "alice", "receiver_id": "compound", "amount": "5000000000000000000000000"})`.
   4. `amm` termina el intercambio, ya sea intercambiando exitosamente los 5 wNEAR dentro del slippage deseado, o fallando.
   5. La función `wnear::ft_resolve_transfer` recibe el éxito/fallo de la promesa. Suponiendo que `amm` implementa transacciones todo-o-nada (es decir, no transferirá menos del monto especificado para así cumplir con los requerimientos del slippage), `wnear` no hará nada en este punto si el intercambio fue exitoso, o sino decrementará el balance de `amm` e incrementará el balance de `alice` por 5000000000000000000000000.


## Explicación a nivel referencia

**NOTES**:
- Todos los montos, balances y allowances están limitados por `U128` (valor máximo `2**128 - 1`).
- El estándar del token usa JSON para la serialización de argumentos y resultados.
- Los montos que los argumentos y resultados tienen están serializados en cadenas de Base-10, e.j. `"100"`. Esto se hace para evitar la limitación de JSON del valor entero máximo de `2**53`.
- El contrato debe monitorear el cambio en el almacenamiento cuando agrega y remueve de colecciones. Esto no se incluye en este estándar básico de token fungible, sino que se encuentra en [Estándar de almacenamiento](../StorageManagement.md).
- Para prevenir prevenir que el contrato desplegado sea modificado o eliminado, no debe tener llaves de acceso en su cuenta.

**Interface**:

```javascript
/***************************************/
/* MÉTODOS DE CAMBIO en token fungible */
/***************************************/
// Transferencia simple a un receptor.
//
// Requerimientos:
// * El llamante del método debe adjuntar un depósito de 1 yoctoⓃ por motivos de seguridad
// * La persona que llama debe tener un número mayor o igual que el `amount` que se solicita
//
// Arguments:
// * `receiver_id`: la cuenta NEAR válida que estará recibiendo los tokens fungibles..
// * `amount`: el número de tokens a transferir, envueltos en comillas y tratados como
//   una cadena, aunque el número se almacenará como un unsigned integer
//   con 128 bits.
// * `memo` (opcional): para casos de uso que pueden beneficiarse de la indexación o
//    o el suministro de información para una transferencia.
function ft_transfer(
    receiver_id: string,
    amount: string,
    memo: string|null
): void {}

// Transfiere tokens y llama a un método en un contrato receptor. Un flujo
// exitoso terminará en una salida de ejecución exitosa hacia el callback en el mismo
// contrato en el método `ft_resolve_transfer`.
//
// Puedes pensar de esto como agregar tokens NEAR nativos a una llamada
// de función. Te permite agregar cualquier Token Fungible a una llamada a
// contrato receptor.
//
// Requerimientos:
// * El llamante del método debe de agregar el depósito de 1 yoctoⓃ por propósitos
//   de seguridad
// * El llamante debe tener mayor o igual cantidad al `amount` requerido
// * El contrato receptor debe implementar `ft_on_transfer` acorde con el
//   estándar. Sino, la función `ft_resolve_transfer` del contrato debe lidiar
//   con la llamada cross-contrato resultante fallida y revertir la transferencia.
// * El contrato de implementar el comportamiento descrito en `ft_resolve_transfer`
//
// Argumentos:
// * `receiver_id`: la cuenta NEAR válida que recibe los tokens fungibles.
// * `amount`: el número de tokens a transferir, envueltos en comillas y tratados como
//   una cadena, aunque el número se almacenará como un unsigned integer
//   con 128 bits.
// * `memo` (opcional): para casos de uso que pueden beneficiarse de la indexación o
//    o el suministro de información para una transferencia.
// * `msg`: especifica la información necesitada por el contrato receptor para
//    así manejar propiamente la transacción. Puede indicar una funcion a
//    llamar y los parámetros a pasar a esa función.
function ft_transfer_call(
   receiver_id: string,
   amount: string,
   memo: string|null,
   msg: string
): Promise {}

/*********************************************/
/* MÉTODOS DE CAMBIO en contratos receptores */
/*********************************************/

// Esta función es implementada en el contrat receptor.
// Como se mencionó el argumento `msg` contiene la información necesaria para hacerle saber al contrato receptor como procesar la petición. Esto tal vez incluya nombres de método y/o argumentos.
// Regresa un valor, o promesa que se resuelve con un valor. El valor es el
// número de tokens no usados en forma de cadena. 
// number of unused tokens in string form. Por ejemplo, si `amount` es 10 pero solo 9 son
// necesitados, regresará "1".
function ft_on_transfer(
    sender_id: string,
    amount: string,
    msg: string
): string {}

/****************/
/* MÉTODOS VIEW */
/****************/

// Regresa el suministro total de tokens fungibles como una cadena que representa el valor de un entero de 128-bits no firmado (unsigned 128-bit integer).
function ft_total_supply(): string {}

// Returns the balance of an account in string form representing a value as an unsigned 128-bit integer. If the account doesn't exist must returns `"0"`.
// Regresa el balance de una cuenta como una cadena y representa el valor como un entero de 128-bits no firmado (unsigned 128-bit integer). Si la cuenta no 
// existe deberá regresar `"0"`.
function ft_balance_of(
    account_id: string
): string {}
```

El siguiente comportamiento es requerido, pero los autores de los contratos tal vez llamen a esta función de diferente manera a la estandarizada `ft_resolve_transfer` como se usa aquí:

```ts
// Finalice una cadena `ft_transfer_call` de llamadas cross-contract.
//
// `ft_transfer_call` procesa:
//
// 1. El remitente llama a `ft_transfer_call` en el contrato FT
// 2. El contrato FT transfiere `amount` tokens del remitente al receptor
// 3. El contrato FT llama a `ft_on_transfer` en el contrato del receptor
// 4+. [el contrato receptor puede hacer otras llamadas cross-contract]
// N. El contrato FT resuelve la cadena de promesas con `ft_resolve_transfer`, y podría
//    reembolsar al remitente parte o la totalidad del `amount` original
//
// Requerimientos:
// * El contrato DEBE prohibir las llamadas a esta función por cualquier cuenta excepto la propia
// * Si la cadena de promesas falló, el contrato DEBE revertir la transferencia del token
// * Si la cadena de promesas se resuelve con una cantidad distinta de cero regresada como una cadena,
//   el contrato DEBE devolver esta cantidad de tokens a `sender_id`
//
// Argumentos:
// * `sender_id`: el remitente de `ft_transfer_call`
// * `receiver_id`: el argumento `receiver_id` dado a `ft_transfer_call`
// * `amount`: el argumento `amount` dado a `ft_transfer_call`
//
// Regresa una representación de cadena, una versión de un entero de 128-bits no
// firmado de cuantos tokens totales fueron gastados por sender_id. Ejemplo: si el remitente
// llama a `ft_transfer_call({ "amount": "100" })`, pero `receiver_id` solo usa
// 80, `ft_on_transfer` se resolverá con `"20"`, y `ft_resolve_transfer`
// regresará `"80"`.
function ft_resolve_transfer(
   sender_id: string,
   receiver_id: string,
   amount: string
): string {}
```

## Desventajas

- El argumento `msg` para `ft_transfer` y `ft_transfer` es de forma libre, que tal vez necesita estandarización.
- El paradigma de un sistema escrow tal vez sea familiar para los desarrolladores y usuarios, y es posible que se necesite educación sobre el manejo adecuado de esto en otro contrato.

## Posibilidades futuras

- Support for multiple token types
- Soporte para múltiples tipos de tokens
- Minting y quemado

## Historia

Vea también las discuciones:
- [Núcleo de token fungible](https://github.com/near/NEPs/discussions/146#discussioncomment-298943)
- [Metadata de token fungible](https://github.com/near/NEPs/discussions/148)
- [Estándar de almacenamiento](https://github.com/near/NEPs/discussions/145)
