# Token no fungible ([NEP-171](https://github.com/near/NEPs/discussions/171))

Versión `1.0.0`

## Resumen

Una interface estándar para los tokens no fungibles (NFTs). 
A standard interface for non-fungible tokens (NFTs). Es decir, tokens que tienen cada uno una identificación única.

## Motivación

En los tres años desde que [ERC-721] fue ratificado por la comunidad de Ethereum, los Tokens No Fungibles se han probado como una increíble oportunidad nueva a través de una amplia gama de disciplinas: coleccionables, arte, videojuegos, finanzas, realidad virtual, bienes raíces, y más.

Este estándar construye las lecciones aprendidas por la experimentación temprana, y empuja las posibilidades más allá aprovechando los atributos únicos de la blockchain de NEAR:

- un tiempo de ejecución asíncromo y fragmentado, significando que dos contratos pueden ser ejecutados al mismo tiempo en diferentes fragmentos
- un modelo de [participación de almacenamiento] que separa las tarifas del [gas] de las demandas que tiene el almacenamiento a la red, permitiendo un almacenamiento on-chain más grande (vea la extensión [Metadata]) y tarifas de transacción ultra bajas.

Dados estos atributos, este estándar NFT puede cimplir con una interacción de usuario lo que otras blockchains necesitan en dos o tres. Lo más notable es `nft_transfer_call`, por el cual un usuario puede esencialmente adjuntar un token a una llamada a un contrato separado. Un escenario de ejemplo:

* Un contrato [Cadáver Exquisito](https://es.wikipedia.org/wiki/Cad%C3%A1ver_exquisito) permite tres enviar tres dibujos, uno para cada sección de una composición final, para ser minteado como su propio NFT y ser vendido en un marketplace, dividiendo las ganancias entre los artistas originales.
* Alice dibuja el tercio superior y lo envía, Bob el tercio de en medio, y Carol termina con el tercio inferior. Como ellos pueden usar `nft_transfer_call` para transferir su NFT al contrato Cadáver Exquisito así como también llamar al método `submit` en él, la llamada de Carol puede automáticamente arrancar el minteo de un NFT compuesto de los tres envíos, así como también listar este NFT compuesto en un marketplace.
* Cuando Dan intenta también llamar `nft_transfer_call` para enviar un tercio superior innecesario del dibujo, el contrato Cadáver Exquisito puede arrojar un error, y la transferencia será revertida para que así Bob mantenga la propiedad de su NFT.

Mientras esto ya es flexible y suficientemente poderoso para manejar todos los tipos de casos de uso nuevos y existentes, las aplicaciones como los marketplaces pueden todavía beneficiarse de la extensión [Gestión de Aprobación].

Estado de la técnica:

- [ERC-721]
- [EIP-1155 for multi-tokens](https://eips.ethereum.org/EIPS/eip-1155#non-fungible-tokens)
- [NEAR's Fungible Token Standard](../FungibleToken/Core.md), que fue el pionero de la técnica "transfer and call"

## Explicación a nivel referencia

**NOTAS**:
- Todos los montos, balances y allowances están limitados por `U128` (valor máximo de `2**128 - 1`).
- El estándar del token usa JSON para la serialización de argumentos y resultados.
- Los montos en los argumentos y resultados son serializados en strings de Base-10, e.j. `"100"`. Esto se hace para evitar la limitación de JSON del valor entero máximo de `2**53`. 
- El contrato debe de monitorear el cambio en el almacenamiento cuando se agrega o remueve de las colecciones. Esto no está incluído en este estándar básico de token fungible, pero sí está en el [Estándar de Almacenamiento](../StorageManagement.md).
- Para prevenir al contrato desplegado de ser modificado o eliminar, no debe tener ninguna clave de acceso en su cuenta.

### Interface NFT

```ts
// La estructura base que será regresada por un token. Si un contrato está usando
// extensions such as Approval Management, Metadata, or other
// extenciones como Gestión de Aprobación, Metadata u otros
// atributos pueden ser incluídos en esta estructura
type Token = {
  id: string,
  owner_id: string,
}

/*********************/
/* MÉTODOS DE CAMBIO */
/*********************/

// Transferencia simple. Transfiere un `token_ida` dado por el dueño actual
// al `receiver_id`.
//
// Requerimientos
// * El llamante del método debe adjuntar un depósito de 1 yoctoⓃ por motivos de seguridad
// * El contrato DEBE entrar en pánico si lo llama alguien que no sea el propietario del token o,
//   si utiliza Gestión de Aprobación, una de las cuentas aprobadas
// * `approval_id` es para usar con la extensión de Gestión de Aprobaciones, vea
//   ese documento para la explicación completa.
// * Si utiliza la gestión de aprobación, el contrato DEBE anular las cuentas aprobadas en
//   en la transferencia exitosa.
//
// Argumentos:
// * `receiver_id`: la cuenta NEAR válida que recibe el token
// * `token_id`: el token a transferir
// * `approval_id`: ID de aprobación esperado. Un número menor que
//    2^53 y, por lo tanto, representable como JSON. Consulte el estándar Gestión 
//    de Aprobaciones para una explicación completa.
// * `memo` (opcional): para casos de uso que pueden beneficiarse de la indexación o
//    el suministro de información para una transferencia
function nft_transfer(
  receiver_id: string,
  token_id: string,
  approval_id: number|null,
  memo: string|null,
) {}

// Devuelve `true` si el token se transfirió desde la cuenta del remitente.

// Transfiere el token y llama a un método en un contrato receptor. Un flujo
// terminará con un resultado de ejecución exitoso al callback dentro del contrato NFT
// en el método `nft_resolve_transfer`.
//
// Puede pensar que esto es similar a adjuntar tokens NEAR nativos a una
// llamada de función. Le permite adjuntar cualquier token no fungible en una llamada a
// un contrato de receptor.
//
// Requirements:
// * El llamante del método debe adjuntar un depósito de 1 yoctoⓃ por motivos de seguridad
// * El contrato DEBE entrar en pánico si lo llama alguien que no sea el propietario del token o,
//   si utiliza Gestión de Aprobación, una de las cuentas aprobadas
// * El contrato receptor DEBE implementar `nft_on_transfer` acorde al
//   estándar. Si no lo hace, `nft_resolve_transfer` del contrato de TF debe lidiar
//   con la llamada cross-contrato fallida y revertir la transferencia.
// * El contrato DEBE implementar el comportamiento descrito en `nft_resolve_transfer`
// * `approval_id` es para usarse con la extensión Gestión de Aprobación, vea
//   ese documento para una explicación completa.
// * Si se usa Gestión de Aprobación, el contrato DEBE anular las cuentas aprovadas
//   en una transferencia exitosa.
//
// Arguments:
// * `receiver_id`: la cuenta NEAR válida que recibe el token
// * `token_id`: el token a transferir
// * `approval_id`: ID de aprobación esperado. Un número menor que
//    2^53 y, por lo tanto, representable como JSON. Consulte el estándar Gestión 
//    de Aprobaciones para una explicación completa.
// * `memo` (opcional): para casos de uso que pueden beneficiarse de la indexación o
//    el suministro de información para una transferencia.
// * `msg`: especifica la información necesaria para el contrato receptor para
//    así manejar adecuadamente la transferencia. Puede indicar una función a
//    llamar y los parámetros a pasar a esa función.
function nft_transfer_call(
  receiver_id: string,
  token_id: string,
  approval_id: number|null,
  memo: string|null,
  msg: string,
): Promise {}


/****************/
/* VIEW METHODS */
/****************/

// Returns the token with the given `token_id` or `null` if no such token.
function nft_token(token_id: string): Token|null {}
```

El siguiente comportamiento es requerido, pero los autores de los contratos tal vez llamen a esta función de diferente manera a la estandarizada `nft_resolve_transfer` como se usa aquí:

```ts
// Finalice una cadena `nft_transfer_call` de llamadas cross-contract.
//
// `ft_transfer_call` procesa:
//
// 1. El remitente llama a `nft_transfer_call` en el contrato NFT
// 2. El contrato NFT transfiere `amount` tokens del remitente al receptor
// 3. El contrato NFT llama a `nft_on_transfer` en el contrato del receptor
// 4+. [el contrato receptor puede hacer otras llamadas cross-contract]
// N. El contrato NFT resuelve la cadena de promesas con `nft_resolve_transfer`, y podría
//    transferir el token de vuelta al remitente
//
// Requerimientos:
// * El contrato DEBE prohibir las llamadas a esta función por cualquier cuenta excepto la propia
// * Si la cadena de promesas falló, el contrato DEBE revertir la transferencia del token
// * Si la cadena de promesas se resuelve con `true`, el contrato DEBE regresar el token
//   a `sender_id`
//
// Argumentos:
// * `sender_id`: el remitente de `nft_transfer_call`
// * `receiver_id`: el argumento `receiver_id` dado a `nft_transfer_call`
// * `token_id`: el argumento `token_id` dado a `nft_transfer_call`
// * `approved_token_ids`: si se usa Gestión de Aprobación, el contrato DEBE proporcionar
//   un conjunto de cuentas originales aprobadas en este argumento, y restaurar
//   estas cuentas aprobadas en caso de reversión.
//
// Regresa true si el token fue transferido exitosamente a `receiver_id`.
function nft_resolve_transfer(
  owner_id: string,
  receiver_id: string,
  token_id: string,
  approved_account_ids: null|string[],
): boolean {}
```

### Interfaz receptora

Los contratos que quieran hacer uso de `nft_transfer_call` deberán implementar lo siguiente:

```ts
// Tomar alguna acción después de recibir un token no fungible
//
// Requerimientos:
// * El contrato deberá restringir las llamadas a esta función a un conjunto de contratos
//   NFT previamente aceptados
//
// Argumentos:
// * `sender_id`: el remitente de `nft_transfer_call`
// * `previous_owner_id`: la cuenta que poseía el NFT antes de ser enviado a 
//   este contrato, que puede diferir de `sender_id` si se usa
//   Gestión de Aprobación de cuentas
// * `token_id`: el argumento `token_id` dado a `nft_transfer_call`
// * `msg`: la información necesaria para que este contrato sepa como procesar la
//   petición. Esto puede incluir nombres de método y/o argumentos
//
// Returns true if token should be returned to `sender_id`
function nft_on_transfer(
  sender_id: string,
  previous_owner_id: string,
  token_id: string,
  msg: string,
): Promise<boolean>;
```

## Errata

* **2021-07-16**: actualizado el argumento `approval_id` de `nft_transfer_call` para ser de tipo `number|null` en vez de `string|null`. Como ya se dijo, No se espera que los ID de aprobación excedan el límite de JSON de 2^53.

  [ERC-721]: https://eips.ethereum.org/EIPS/eip-721
  [storage staking]: https://docs.near.org/docs/concepts/storage-staking
  [gas]: https://docs.near.org/docs/concepts/gas
  [Metadata]: Metadata.md
  [Gestión de Aprobación]: ApprovalManagement.md

