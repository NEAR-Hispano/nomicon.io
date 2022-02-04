# Evento de token no fungible

Versión `1.0.0`

## Resumen

Interfaces estándar para
Interfaces estándar para acciones de contrato NFT.

## Motivación

Las aplicaciones impulsadas por NFTs realizan acciones similares.
Por ejemplo - `minting`, `quemando` y `transfiriendo`.
Cada app puede tener su propia manera de realizar estas acciones.
Esto introduce inconsistencia al capturar estos eventos.
Esta extensión aborda eso.

Es común para las apps de NFT el transferir muchos o un token al mismo tiempo.
Otras aplicaciones necesitan monitorear estos y eventos similares consistentemente.
Si no, el estado del monitoreo a lo largo de muchas apps impulsadas por NFTs se vuelve inviable. 

Necesitamos una manera estandarizada de capturar eventos.
Esto se discutió aquí https://github.com/near/NEPs/issues/254.

## Eventos

Muchas apps usan diferentes interfaces que representan la misma acción.
Esta interface estandariza el proceso al introducir eventos de log.
No hay evento NEP al momento, así que este estándar crea el camino hacia eso.

Los eventos usan la capacidad de logs estándar de NEAR y se definen como una convención.
Los eventos son logs que empiezan con el prefijo `EVENT_JSON:` seguido de un solo documento JSON válido de la interface siguiente:

```ts
// Interface para capturar datos
// acerca de un evento
// Argumentos
// * `standard`: nombre del estándar e.j. nep171
// * `version`: e.j. 1.0.0
// * `event`: cadena
// * `data`: evento de datos asociado
interface EventLogData {
    standard: string,
    version: string,
    event: string,
    data?: unknown,
}
```

#### Logs de evento válidos:

```js
EVENT_JSON:{"standard": "nepXXX", "version": "1.0.0", "event": "xyz_is_triggered"}
```

```js
EVENT_JSON:{
  "standard": "nepXXX",
  "version": "1.0.0",
  "event": "xyz_is_triggered"
}
```

```js
EVENT_JSON:{"standard": "nepXXX", "version": "1.0.0", "event": "xyz_is_triggered", "data": {"triggered_by": "foundation.near"}}
```

#### Logs de evento inválidos:

* Dos eventos en un solo log (en lugar de eso, llame a `log` para cada uno de los eventos)
```
EVENT_JSON:{"standard": "nepXXX", "version": "1.0.0", "event": "xyz_is_triggered"}
EVENT_JSON:{"standard": "nepXXX", "version": "1.0.0", "event": "xyz_is_triggered"}
```
* Datos JSON inválidos
```
EVENT_JSON:invalid json
```

## Interface

Los eventos de Token No Fungible DEBE tener el parámetro estándar `standard` con el valor `"nep171"`, la versión estándar con el valor `"1.0.0"`, el valor `event` es uno de `nft_mint`, `nft_burn`, `nft_transfer`, y `data` debe ser uno de los siguientes tipos relevantes: `NftMintLog[] | NftTransferLog[] | NftBurnLog[]`:

```ts
interface EventLogData {
    standard: "nep171",
    version: "1.0.0",
    event: "nft_mint" | "nft_burn" | "nft_transfer",
    data: NftMintLog[] | NftTransferLog[] | NftBurnLog[],
}
```

```ts
// Un evento log para capturar el minteo de tokens
// Argumentos
// * `owner_id`: "account.near"
// * `token_ids`: ["1", "abc"]
// * `memo`: mensaje opcional
interface NftMintLog {
    owner_id: string,
    token_ids: string[],
    memo?: string
}

// Un evento log para capturar la quema de tokens
// Argumentos
// * `owner_id`: el dueño de los tokens a quemar
// * `authorized_id`: acuenta aprobada para quemar, si aplica
// * `token_ids`: ["1","2"]
// * `memo`: mensaje opcional
interface NftBurnLog {
    owner_id: string,
    authorized_id?: string,
    token_ids: string[],
    memo?: string
}

// Un evento log para capturar la transferencia de tokens
// Argumentos
// * `authorized_id`: cuenta aprobada para transferir
// * `old_owner_id`: "owner.near"
// * `new_owner_id`: "receiver.near"
// * `token_ids`: ["1", "12345abc"]
// * `memo`: mensaje opcional
interface NftTransferLog {
    authorized_id?: string,
    old_owner_id: string,
    new_owner_id: string,
    token_ids: string[],
    memo?: string
}
```

## Ejemplos

Minteo por lotes de un solo propietario (formateado para fines de legibilidad):

```js
EVENT_JSON:{
  "standard": "nep171",
  "version": "1.0.0",
  "event": "nft_mint",
  "data": [
    {"owner_id": "foundation.near", "token_ids": ["aurora", "proximitylabs"]}
  ]
}
```

Minteo por lotes de diferentes propietarios:

```js
EVENT_JSON:{
  "standard": "nep171",
  "version": "1.0.0",
  "event": "nft_mint",
  "data": [
    {"owner_id": "foundation.near", "token_ids": ["aurora", "proximitylabs"]},
    {"owner_id": "user1.near", "token_ids": ["meme"]}
  ]
}
```

Eventos diferentes (entradas de log separadas):

```js
EVENT_JSON:{
  "standard": "nep171",
  "version": "1.0.0",
  "event": "nft_burn",
  "data": [
    {"owner_id": "foundation.near", "token_ids": ["aurora", "proximitylabs"]},
  ]
}
```

```js
EVENT_JSON:{
  "standard": "nep171",
  "version": "1.0.0",
  "event": "nft_transfer",
  "data": [
    {"old_owner_id": "user1.near", "new_owner_id": "user2.near", "token_ids": ["meme"], "memo": "have fun!"}
  ]
}
```

## Otro métodos

Note que los eventos de ejemplo anteriores cubren dos tipos diferentes de eventos:
1. Eventos que no están especificados en el estándar NFT (`nft_mint`, `nft_burn`)
2. Un evento que es cubierto en el [Estándar básico de NFT](https://nomicon.io/Standards/NonFungibleToken/Core.html#nft-interface). (`nft_transfer`)

Este estándar de eventos también aplica más allá de los tres eventos resaltados aquí, donde los eventos futuros siguen la misma convención que el segundo tipo. Por ejemplo, si un contrato NFT usa el [estándar de gestión de aprobación](https://nomicon.io/Standards/NonFungibleToken/ApprovalManagement.html), puede emitir un evento para `nft_approve` si la comunidad de desarrolladores lo considera importante.
 
Por favor siéntete libre de abrir una pull request para extender los estándares de eventos detallados aquí a medida que surjan las necesidades.

## Desventajas

Hay una limitación conocida de cadenas de 16kb cuando se capturan logs.
Esto se puede observar si los `token_ids` varían en longitud entre aplicaciones diferentes.
Esto impacta la cantidad de eventos que se pueden procesar.
