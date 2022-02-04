# Gestión de Aprovación de Token No Fungible ([NEP-178](https://github.com/near/NEPs/discussions/178))

Version `1.0.0`

## Resumen

Un sistema para permitir a un conjunto de usuarios o contratos el transgeriro Tokens No Fungibles específicos a nombre de un propietario. Similar a los sistemas de gestión de aprobación en los estándares como [ERC-721].

  [ERC-721]: https://eips.ethereum.org/EIPS/eip-721

## Motivación

Las personas familiarizadas con [ERC-721] puede esperar necesitar un sistema de gestión de aprobación para transferencias básicas, donde una simple tranferencia de Alice a Bob requiere que Alice primero _apruebe_ a Bob para gastar uno de sus tokens, después de eso Bob puede llamar a `transfer_from` para transferir el token a sí mismo.

El [estándar básico de Token No Fungible](Core.md) incluye un buen soporte para transferencias atómicas seguras sin tal complejidad. Incluso proporciona la funcionalidad "transfiere y llama" (`nft_transfer_call`) que permite a un token en específico ser "agregado" a la llamada a un contrato separado. Para muchos flujos de Token No Fungible, estas opciones pueden circunvenir las necesidades para un sistema de Gestión de Aprobaciones completo.

Sin embargo, algunos desarrolladores de Tokens No Fungibles, marketplaces, dApps, o artistas pueden requerir un mayor control. Este estándar proporciona una interfaz uniforme permitiendo a los dueños de tokens aprobar otras cuentas NEAR, individuos o contratos, para transferir tokens específicos en nombre del propietario.

Estado de la técnica:

- Ethereum's [ERC-721]
- [NEP-4](https://github.com/near/NEPs/pull/4), Estándar antiguo de NEAR para NFTs que no incluye la aprobación por ID de token

## Escenarios de ejemplo

Vamos a considerar algunos ejemplos. Nuestro elenco de personajes y aplicaciones:

* Alice: tiene la cuenta `alice` sin contratos desplegados en ella
* Bob: tiene la cuenta `bob` sin contratos desplegados en ella
* NFT: un contrato con la cuenta `nft`, implementa solo el [estándar básico NFT](Core.md) con esta extensión de Gestión de Aprobación
* Mercado: un contrato con la cuenta `market` que vende tokens desde `nft` así como de otros contratos de NFT
* Bazar: similar a Mercado, pero implementado de manera diferente (spoiler alert: no tiene la función `nft_on_approve`!), tiene la cuenta `bazaar`

Alice y Bob están ya [registrados](../StorageManagement.md) con NFT, Mercado, y Bazar, y Alice posee un token en el contrato NFT con el ID=`"1"`.

Examinemos las llamadas técnicas a través de los siguiente escenarios:

1. [Aprobación simple](#1-simple-approval): Alice aprueba a Bob para transferir su token.
2. [Aprobación con llamada cross-contrato (XCC)](#2-approval-with-cross-contract-call): Alice aprueba a Mercado para transferir uno de sus tokens y pasa un `msg` para que NFT llame a `nft_on_approve` en el contrato de Mercado.
3. [Aprobación con XCC, caso extremo](#3-approval-with-cross-contract-call-edge-case): Alice aprueba a Bazar y pasa un `msg` otra vez, pero, ¿qué es esto? Bazar no tiene `nft_on_approve` implementado, entonces Alice ve un error en el resultado de la transacción. No hay de que preocuparse, sin embargo, ella revisa `nft_is_approved` y ve que ella aprobó exitosamente a Bazar, a pesar del error.
4. [IDs de aprobación](#4-approval-ids): Bob compra el token de Alice a través de Mercado.
5. [IDs de aprobación, caso extremo](#5-approval-ids-edge-case): Bob transfiere el mismo token de regreso a Alice, Alice vuelve a aprobar a Mercado y a Bazar. Bazar tiene un cache antiguo. Bob trata de comprar de Bazar usando el precio anterior.
6. [Revoca uno](#6-revoke-one): Alice revoca la aprobación de Mercado para este token.
7. [Revoca todos](#7-revoke-all): Alice revoca la aprobación de todos para este token.

### 1. Aprobación simple

Alice aprueba a Bob para transferir su token.

**Explicación de alto nivel

1. Alice aprueba a Bob
2. Alice consulta el token para verificar
3. Alice verifica de una manera diferente

**Llamadas técnicas**

1. Alice llama a `nft::nft_approve({ "token_id": "1", "account_id": "bob" })`. Ella adjunta 1 yoctoⓃ, (.000000000000000000000001Ⓝ). Usa [NEAR CLI](https://docs.near.org/docs/tools/near-cli) para hacer esta llamada, el comando sería:

       near call nft nft_approve \
         '{ "token_id": "1", "account_id": "bob" }' \
         --accountId alice --depositYocto 1

   La respuesta:

       ''

2. Alice llama al método view `nft_token`:

       near view nft nft_token \
         '{ "token_id": "1" }'

   La respuesta:

       {
         "id": "1",
         "owner_id": "alice.near",
         "approvals": {
           "bob": 1,
         }
       }

3. Alice llama al método view `nft_is_approved`:

       near view nft nft_is_approved \
         '{ "token_id": "1", "approved_account_id": "bob" }'

   La respuesta:

       true

### 2. Aprobación con llamada cross-contrato

Alice aprueba a Mercado para transferir uno de sus tokens y pasa un `msg` para que NFT llame a `nft_on_approve` en el contrato Mercado. Ella probablemente hace esto desde la interfaz de la aplicación de Mercado que debería de saber como construir `msg` de una manera útil.

**Explicación de alto nivel**

1. Alice llama a `nft_approve` para aprobar a `market` para que transfiera su token, y pasa un `msg`
2. Como `msg` está incluído, `nft` agendará una llamada cross-contrato a `market`
3. Mercado puede hacer lo que quiera con esta información, como listar el token a la venta a el precio dado. El resultado de esta operación es regresado como la salida de la promesa de la llamada original `nft_approve`.

**Llamadas técnicas**

1. Usando near-cli:

       near call nft nft_approve '{
         "token_id": "1",
         "account_id": "market",
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId alice --depositYocto 1
   
   En este punto, near-cli esperará hasta que la cadena de la llamada cross-contrato se resuelva por completo, que también sería true si Alice usó una interfaz de Mercado que usa [near-api-js](https://docs.near.org/docs/develop/front-end/near-api-js). Aunque la parte de Alice está terminada, lo demás pasa detrás de escena.

2. `nft` agenda una llaada a `nft_on_approve` en `market`. Usando la notación near-cli notation para una referencia cruzada fácil dado lo anterior, esto se vería como:

       near call market nft_on_approve '{
         "token_id": "1",
         "owner_id": "alice",
         "approval_id": 2,
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId nft

3. `market` ahora sabe que puede enviar el token de Alice por 100 [nDAI](https://explorer.mainnet.near.org/accounts/6b175474e89094c44da98b954eedeac495271d0f.factory.bridge.near), y que cuando los transfiera a un comprador usando `nft_transfer`, puede pasar también el `approval_id` dado para asegurar que Alice no haya cambiado su opinión. Puede agendar todas las llamadas cross-contratos que quiera, y si regresa estas promesas correctamente, la llamada inicial de Alice usando near-cli se resolverá con la salida del paso final en la cadena. Si Alice sí hizo esta llamada desde la interfaz de Mercado, la interfaz puede usar este valor retornado para algo útil.

### 3. Aprobación con llamada cross-contrato, caso extremo

Alice aprueba a Baazar y pasa un `msg` otra vez. Tal vez ella sí hace esto con near-cli, en lugar de usar la interfaz de Bazar, porque ¿qué es esto? Bazar no implementa `nft_on_approve`, así que Alice vee un error en el resultado de la transacción.

No hay de que preocuparse, sin embargo, revisa `nft_is_approved` y ve que ella exitosamente aprobó a Bazar, a pesar del error. Ella tendrá que encontrar una manera nueva de listar su token a la venta en Bazar, en vez de usar el mismo atajo de `msg` que funcionó para Mercado.

**Explicación de alto nivel**

1. Alice llama a `nft_approve` para aprobar a `bazaar` para transferir su token, y pasa un `msg`.
2. Como `msg` está incluído, `nft` agendará una llamada cross-contrato a `bazaar`.
3. Bazar no implementa `nft_on_approve`, por lo que esta llamada resulta en un error. La aprobación funcionó de todos modos, pero Alice ve un error en su salida del near-cli.
4. Alice revisa si `bazaar` fue aprobado, y ve que sí, a pesar del error.

**Llamadas técnicas**

1. Usando near-cli:

       near call nft nft_approve '{
         "token_id": "1",
         "account_id": "bazaar",
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId alice --depositYocto 1

2. `nft` agenda una llamada a `nft_on_approve` en `market`. Usando la notación near-cli notation para una referencia cruzada fácil dado lo anterior, esto se vería como:

       near call bazaar nft_on_approve '{
         "token_id": "1",
         "owner_id": "alice",
         "approval_id": 3,
         "msg": "{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
       }' --accountId nft

3. 💥 `bazaar` no implementa este método, así que la llamada resulta en un error. Alice vee este error en la salida de near-cli.

4. Alice revisa si la aprobación funcionó, a pesar del error en la llamada cross-contrato:

       near view nft nft_is_approved \
         '{ "token_id": "1", "approved_account_id": "bazaar" }'

   La respuesta:

       true

### 4. IDs de aprobación

Bob compra el token de Alice a través de Mercado. Bob probablemente hace esto con la interfaz de Mercado, que probablemente iniciará la transferencia con una llamada a `ft_transfer_call` en el contrato nDAI para transferir 100 nDAI a `market`. Como en la función del estándar NFT "transfiere y llama", `ft_transfer_call` que pertenece a [Token Fungible](../FungibleToken/Core.md) toma un `msg` que `market` puede usar para pasar información de que tendrá que pagar a Alice y transferir el NFT. La transferencia del NFT en la única parte que nos interesa aquí.

**Explicación de alto nivel**

1. Bob firma alguna transacción que resulta en `market` llamando a `nft_transfer` en el contrato `nft`, como se describe arriba. Para ser confiables y pasar las auditorías de seguridad, `market` necesita de pasar `approval_id` para que sepa que tiene la información actualizada.

**Llamadas técnicas**
Usando la notación near-cli para mantener coherencia:

    near call nft nft_transfer '{
      "receiver_id": "bob",
      "token_id": "1",
      "approval_id": 2,
    }' --accountId market --depositYocto 1

### 5. IDs de aprobación, caso extremo

Bob transfiere el mismo token de regreso a Alice, Alice vuelve a aprobar a Mercado y a Bazar, listando su token a un precio mayor al anterior. Bazar de alguna manera no sabe de estos cambios, y aún almacena internamente `approval_id: 3` al igual que el precio anterior de Alice. Bob trata de comprar de Bazar con el precio anterior. Como en el ejemplo anterior, esto probablemente empieza con una llamada a un contrato diferente, que eventualmente resulta en una llamada a `nft_transfer` en `bazaar`. Consideremos un escenario posible desde ese punto.

**Explicación de alto nivel**

Bob firma alguna transacción que resulta en `baazar` llamando a `nft_transfer` en el contrato `nft`, como se describió arriba. Para ser confiables y pasar las auditorías de seguridad, `baazar` necesita pasar `approval_id` para saber que tiene información actualizada. No tiene la información actualizada, así que la llamada falla. Si la llamada `nft_transfer` incial es parte de una cadena de llamadas originada por una llamada a `ft_transfer_call` en un token fungible, el pago de Bob será reembolsado y ningún activo cambiará de propietario.

**Llamadas técnicas**

Usando near-cli para mantener la coherencia:

    near call nft nft_transfer '{
      "receiver_id": "bob",
      "token_id": "1",
      "approval_id": 3,
    }' --accountId bazaar --depositYocto 1

### 6. Revoca uno

Alice revoca la aprobación de Mercado para este token.

**Llamadas técnicas**

Usando near-cli:

    near call nft nft_revoke '{
      "account_id": "market",
      "token_id": "1",
    }' --accountId alice --depositYocto 1
Note que `market` no obtendrá una llamada cross-contrato en este caso. Los implementadores de la app Mercado deberían implementar una funcionalidad de tipo [cron](https://es.wikipedia.org/wiki/Cron_(Unix)) para intermintentemente revisar que Mercado tiene el acceso que esperan.

### 7. Revoca todos

Alice revoca todas las aprobaciones para este token.

**Llamadas técnicas**

Usando near-cli:

    near call nft nft_revoke_all '{
      "token_id": "1",
    }' --accountId alice --depositYocto 1

Otra vez, note que los aprobadores anteriores no obtendrán llamadas cross-contrato en este caso.

## Explicación a nivel de referencia

La estructura `Token` regresada por `nft_token` debe incluír el campo `approvals`, que es un mapa de IDs de cuenta a aprobar. Usando la notación [Record type](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeystype) de TypeScript:

```diff
 type Token = {
   id: string,
   owner_id: string,
+  approvals: Record<string, number>,
 };
```

Ejemplo de datos de token:

```json
{
  "id": "1",
  "owner_id": "alice.near",
  "approvals": {
    "bob.near": 1,
    "carol.near": 2,
  }
}
```

### ¿Qué es un "ID de aprobación"?

Es un número único dado para cada aprobación que permite marketplaces bien intencionados u otros revendedores de NFT de terceros evitar una condición de carrera. La condición de carrera ocurre cuando:

1. Un token es listado en dos marketplaces, que los dos son almacenados en el token como cuentas aprobadas.
2. Un marketplace vende el token, que lo remueve de las cuentas aprobadas.
3. El nuevo dueño vende todo de regreso al dueño original.
4. El dueño original aprueba otra vez el token para el segundo marketplace para que lo liste a un precio nuevo. Pero por alguna razón el segundo marketplace todavía lista el token con el precio anterior y no sabe de las transferencias que están pasando.
5. El segundo marketplace, operando con información antigua, trata de otra vez vender el token al precio anterior.

Note que mientras esto describe un error honesto, la posibilidad de este bug también puede ser ventajosa para partes maliciosas a través del [front-running](https://defi.cx/front-running-ethereum/).

Para evitar esta posibilidad, el contrato NFT genera un ID de aprobación único cada vez que aprueba una cuenta. Luego cuando llamamos a `nft_transfer` o a `nft_transfer_call`, la cuenta aprobada pasa el `approval_id` con este valor para asegurarse que el estado subyacente del token no haya cambiado de lo que espera la cuenta aprobada.

Quedándonos con el ejemplo anterior, digamos que la aprobación incial del segundo marketplace generó los datos de `aprovals` siguientes:

```json
{
  "id": "1",
  "owner_id": "alice.near",
  "approvals": {
    "marketplace_1.near": 1,
    "marketplace_2.near": 2,
  }
}
```

Pero después de las transacciones y re-aprobaciones descritas arriba, el token tal vez tenga los `approvals` como:

```json
{
  "id": "1",
  "owner_id": "alice.near",
  "approvals": {
    "marketplace_2.near": 3,
  }
}
```

El marketplace luego trata de llamar a `nft_transfer`, pasándole información antigua:

```bash
# oops!
near call nft-contract.near nft_transfer '{ "approval_id": 2 }'
```


### Interfaz

El contrato NFT debe implementar los métodos siguientes:

```ts
/*********************/
/* MÉTODOS DE CAMBIO */
/*********************/
// Agregar una cuenta aprobada para un token específico
//
// Requerimientos
// * El llamante del método debe adjuntar un depósito de al menos 1 yoctoⓃ por
//   razones de seguridad
// * El contrato PUEDE requerir al llamante adjuntarr un depósito más grande, para cubrir
//   el costo del almacenamiento de los datos del aprobador
// * El contrato DEBE entrar en pánico si fue llamado por alguien que no sea el propietario del token
// * El contrato DEBE de entrar en pánico si la adición causa que `nft_revoke_all` exceda
//   el límite de gas de un solo bloque. Vea a continuación para más información.
// * El contrato DEBE de incrementar el ID de aprobación incluso si re-aprueba una cuenta
// * Si fue exitosamente aprobada o si ya estaba aprobada, y si `msg` está presente,
//   el contrato DEBE de llamar `nft_on_approv` en `account_id`. Vea la descripción de
//   `nft_on_approve` a continuación para más detalles.
//
// Arguments:
// * `token_id`: el token por el cual se agrega una aprobación
// * `account_id`: la cuenta a agregar `approvals`
// * `msg`: cadena opcional a pasar a `nft_on_approve`
//
// Regresa void, si no hay `msg`. De otra manera, regresa una llamada de promesa a
// `nft_on_approve`, que puede resolver con cualquier cosa que quiera
function nft_approve(
  token_id: TokenId,
  account_id: string,
  msg: string|null,
): void|Promise<any> {}

// Revocar una cuenta aprobada para un token específico.
//
// Requirements
// * El llamante del método debe adjuntar un depósito de al menos 1 yoctoⓃ por
//   razones de seguridad
// * Si el contrato requiere un depósito >1yN en `nft_approve`, el contrato
//   DEBE reembolsar el depósito de almacenamiento asociado cuando el dueño revoca la aprobación
// * El contrato DEBE entrar en pánico si fue llamado por alguien que no sea el propietario del token
//
// Argumentos:
// * `token_id`: el token por el cual revocar una aprobación
// * `account_id`: la cuenta que será removida de `approvals`
function nft_revoke(
  token_id: string,
  account_id: string
) {}

// Revoca todas las cuentas aprobadas para un token específico.
//
// Requirements
// * El llamante del método debe adjuntar un depósito de al menos 1 yoctoⓃ por
//   razones de seguridad
// * Si el contrato requiere un depósito >1yN en `nft_approve`, el contrato
//   DEBE reembolsar el depósito de almacenamiento asociado cuando el dueño revoca la aprobación
// * El contrato DEBE entrar en pánico si fue llamado por alguien que no sea el propietario del token
//
// Arguments:
// * `token_id`: el token con las aprobaciones a revocar
function nft_revoke_all(token_id: string) {}

/****************/
/* MÉTODOS VIEW */
/****************/

// Revisar si un token fue aprobado para transferir por una cuenta dada, opcionalmente
// revisando un approval_id
//
// Argumentos:
// * `token_id`: el token por el cual se revocará la aprobación
// * `approved_account_id`: la cuenta para revisar la existencia en `approvals`
// * `approval_id`: un ID de aprobación opciona para compara con el ID de aprovación actual para una cuenta dada
//
// Regresa:
// si el `approval_id` dado, `true` si `approved_account_id` es aprobado con el `approval_id` dado
// de otra manera, `true` si `approved_account_id` está en la lista con las cuentas aprobadas
function nft_is_approved(
  token_id: string,
  approved_account_id: string,
  approval_id: number|null
): boolean {}
```

### ¿Por qué `nft_approve` debe de entrar en pánico si `nft_revoke_all` llega a fallar después?

En la descripción de `nft_approve` anterior, se dice:

    El contrato DEBE de entrar en pánico si la adición causa que `nft_revoke_all` exceda
    el límite de gas de un solo bloque.

¿Qué significa esto?

Primero, es útil entender que lo que queremo decir cuando decimos "single-block gas limit". Esto se refiere a [límite máximo de gas por bloque en la capa de protocolo](https://docs.near.org/docs/concepts/gas#thinking-in-gas). Este número incrementará con el tiempo.

Remover datos de un contrato usa gas, así que si un NFT tiene un número suficientemente largo de aprobaciones, `nft_revoke_all` fallaría, porque llamarlo excedería el gas máximo

Los contratos deben de prevenir esto capturando el número de aprobaciones por un token dado. Sin embargo, depende del autor del contrato el determinar un límite sensible para su contrato (y el límite de gas para un bloque al momento que lo desplieguen). Como las implementaciones de los contratos pueden variar, algunas de ellas serán capaces de soportar un número mayor de aprobaciones que otras, incluso con el mismo límite gas por bloque.

Los autores de contratos pueden escoger el establecer un límite bajo y seguro como 10 aprobaciones, o ellos podrían calcular dinámicamente si una aprobación nueva rompería las llamadas futuras a `nft_revoke_all`. Pero cada contrato DEBE asegurar que nunca se romple la funcionalidad de `nft_revoke_all`.


### Interfaz de contrato de cuenta aprobada

Si un contrato es aprobado para transferir NFTs, puede implementar `nft_on_approve` para actualizar su propio estado cuando se le da la aprobación para un token:

```ts
// Responder a la notificación de cuando el contrato ha sido aprobado para un token.
//
// Notas
// * El contrato sabe el ID del contrato del token por `predecessor_account_id`
//
// Argumentos:
// * `token_id`: el token el cual aprobó a este contrato
// * `owner_id`: el dueño del token
// * `approval_id`: el ID de la aprobación guardado por el contrato NFT para esta aprobación.
//    Se espera que sea un número dentro del límite de 2^53 representable por JSON.
// * `msg`: especifica la información necesitada por el contrato aprobado para poder
//    manejar la aprobación. Puede indicar una función a llama y los
//    parámetros a pasar a esa función
function nft_on_approve(
  token_id: TokenId,
  owner_id: string,
  approval_id: number,
  msg: string,
) {}
```

Note que el contrato NFT va a ejecutar y olvidar esta llamada, ignorando cualquier valor regresado o errores generados. Esto significa que incluso si la cuenta aprobada no tiene un contrato o no implementa `nft_on_approve`, la aprobación seguirá funcionando correctamente desde el punto de vista de un contrato NFT.

También note que no hay un `nft_on_revoke` paralelo cuando se revocan una sola aprobación o de cuando se revocan todas. Esto es parcialmente causado porque agendar muchas llamadas `nft_on_revoke` cuando se revocan todas las aprobaciones podría incurrir en [tarifas de gas](https://docs.near.org/docs/concepts/gas) prohibitivas. Apps y contratos que guardan en cache las aprobaciones NFT pueden entonces no depender de tener información actualizada, y deberían actualizar sus caches periódicamente. Como esta será la realidad necesaria para lidiar con `nft_revoke_all`, no hay razón para complicar `nft_revoke` con una llamada `nft_on_revoke`.

### Costos no incurridos para el comportamiento básico NFT

Los contratos NFT deberían de ser implementados de una manera que se eviten tarifas de gas extras por la serialización y deserialización de `approvals` para llamadas a métodos `nft_*` que no sea `nft_token`. Vea[implementación de `ft_metadata` usando `LazyOption`](https://github.com/near/near-sdk-rs/blob/c2771af7fdfe01a4e8414046752ee16fb0d29d39/examples/fungible-token/ft/src/lib.rs#L71) de `near-contract-standards` como ejemplo de referencia.
