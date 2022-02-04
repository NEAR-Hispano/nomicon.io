# Enumeración del Token No Fungible ([NEP-181](https://github.com/near/NEPs/discussions/181))

Versión `1.0.0`

## Resumen

Interfaces estándar para contar y obtener tokens, para un contrato NFT completo o para un propietario determinado.

## Motivación

Aplicaciones como marketplaces y billeteras necesitan una manera de mostrar todos los tokens que una cuenta posee y enseñar las estadísticas de todos los tokens para un contrato dado. Esta extensión provee una manera estandarizada de hacerlo.

Mientras que algunos contratos NFT pueden prescindir de esta extensión para ahorrar costos de almacenamiento, esto requiere a las aplicaciones tener una capa indexada personalizada off-chain. Esto hace más difícil para las apps integrarse con dichos NFTs. Las apps que se integran solo con NFTs que usan la extensión Enumeración ni siquiera necesitan un componente del lado del servidor, ya que ellas pueden recuperar toda la información que necesitan directamente de la blockchain.

Estado de la técnica:

- Extensión de enumeración [ERC-721]

## Interface

El contrato debe implementar los siguientes métodos view:

```ts
// Regresa el suministro total de tokens no fungibles como una cadena que representa a
// un entero unsigned de 128-bits para evitar el límite numérico de JSON de 2^53; y "0" si no hay tokens.
function nft_total_supply(): string {}

// Obtener una lista de tokens
//
// Argumentos:
// * `from_index`: una cadena que representa un entero unsigned de 128-bits,
//    representa el índice inicial de los tokens a regresar
// * `limit`: el número máximo de los tokens a regresar
//
// Regresa un arregla de objetos Token, como se describe en el estándar básico, y un arreglo vacío si no hay tokens
function nft_tokens(
  from_index: string|null, // default: "0"
  limit: number|null, // default: ilimitado (podría fallar debido al límite del gas)
): Token[] {}

// Obtener un número de token que posee una cuenta dada
//
// Argunmentos:
// * `account_id`: una cuenta NEAR válida
//
// Regresa el número de tokens no fungibles que posee `account_id` como una cadena,
// que representa un entero unsigned de 128-bits para evitar el límite numérico de JSON
// de 2^53; y "0" si no hay tokens.
function nft_supply_for_owner(
  account_id: string,
): string {}

// Obtener una lista de todos los tokens que posee una cuenta
//
// Argumentos:
// * `account_id`: una cuenta NEAR válida
// * `from_index`: una cadena que representa un entero unsigned de 128-bits,
//    representa el índice inicial de los tokens a regresar
// * `limit`: el número máximo de los tokens a regresar
//
// Regresa una lista paginada de todos los tokens que posee esta cuenta, y un arreglo vacío si no hay tokens
function nft_tokens_for_owner(
  account_id: string,
  from_index: string|null, // default: 0
  limit: number|null, // default: ilimitado (podría fallar debido al límite del gas)
): Token[] {}
```

## Notas

En el momento en el que se escribe esto, las colecciones especializada en el crate de Rust `near-sdk` son iterables, pero no todas tienen implementada la solución `iter_from`. Pueden haber ganancias en la eficiencia para las colecciones grandes y se invita a los desarrolladores a probar sus estructuras de datos con una cantidad grande de entradas.

  [ERC-721]: https://eips.ethereum.org/EIPS/eip-721
  [almacenamiento]: https://docs.near.org/docs/concepts/storage-staking
