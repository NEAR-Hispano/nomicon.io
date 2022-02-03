# Metadatos de Token No Fungible ([NEP-177](https://github.com/near/NEPs/discussions/177))

Version `2.0.0`

## Resumen

Una interface para los metadatos de tokens no fungibles. La meta es mantener los metadatos a prueba del futuro así como ligeros. Esto será importante para las dApps que necesiten información adicional acerca de las propiedades de un NFT, y ampliamente compatible con otro estándares de tokens tal que el [NEAR Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) pueda mover tokens entre cadenas.

## Motivación

El valor principal de los tokens no fungibles proviene de sus metadatos. Mientras que el [estándar básico](Core.md) proporciona la interface mínima que puede ser cosiderada un token no fungible, la mayoría de artistas, desarrolladores, y dApps querrán asociar más datos con cada NFT, y querrán una manera predecirble de interactuar con cualquier metadato de un NFT.

El enfoque único de NEAR [stakeo de almacenamiento](https://docs.near.org/docs/concepts/storage-staking) hace feasible el almacenar más información on-chain que otras blockchains.
Este estándar aprovecha esta fortaleza para atributos de metadatos comunes, y proporciona una manera estandarizada para enlazar información adicional off-chain para soportar la experimentación rápida de la comunidad.

Este estándar también provee una versión `spec`. Esto hace fácil para los consumidores de NFTs, como los marketplaces, el saber si soportan todas las características del token dado.

Estado de la técnica:

- [Estándar de metadatos de tokens fungibles](../FungibleToken/Metadata.md) de NEAR
- Discusión sobre el estándar NFT completo de NEAR: #171

## Interface

Los metadatos implementan el nivel contrato (`NFTContractMetadata`) y el nivel token (`TokenMetadata`). Los metadatos relevantes para cada uno:

```ts
type NFTContractMetadata = {
  spec: string, // requerido, esencialmente una versión como "nft-1.0.0"
  name: string, // requerido, ej. "Mochi Rising — Digital Edition" o "Metaverse 3"
  symbol: string, // requirido, ej. "MOCHI"
  icon: string|null, // URL de datos
  base_uri: string|null, // Puerta de enlace centralizada conocida para tener un acceso confiable a los activos de almacenamiento descentralizado a los que hace referencia por `reference` or URLs `media`
  reference: string|null, // URL a archivo JSON con más información
  reference_hash: string|null, // Hash sha256 codificado en Base64 de JSON del campo de referencia. Obligatorio si se incluye `reference`.
}

type TokenMetadata = {
  title: string|null, // ej. "Arch Nemesis: Mail Carrier" or "Parcel #5055"
  description: string|null, // descripción de formato libre
  media: string|null, // URL a medios asociados, preferiblemente a almacenamiento descentralizado dirigido a contenido
  media_hash: string|null, // Hash sha256 codificado en Base64 de JSON del campo de referencia. Obligatorio si se incluye `reference`.
  copies: number|null, // número de copias del conjunto de metadatos en existencia cuando el token fue minteado.
  issued_at: number|null, // Cuando el token fue emitido o minteado, un epoch Unix en milisegundos
  expires_at: number|null, // Cuando el token expira, un epoch Unix en milisegundos
  starts_at: number|null, // Cuando el token empieza a ser válido, un epoch Unix en milisegundos
  updated_at: number|null, // Cuando el token fue actualizado por última vez, un epoch Unix en milisegundos
  extra: string|null, // cualquier cosa extra que el NFT quiera guardar on-chain. Puede ser un JSON con formato de cadena.
  reference: string|null, // URL hacia un archivo JSON off-chain con más información
  reference_hash: string|null // Hash sha256 codificado en Base64 de JSON del campo de referencia. Obligatorio si se incluye `reference`.
}
```

Una nueva función DEBE de ser soportada en el contrato del NFT:

```ts
function nft_metadata(): NFTContractMetadata {}
```

Un atributo nuevo DEBE de ser agregado a cada estructura `Token`:

```diff
 type Token = {
   id: string,
   owner_id: string,
+  metadata: TokenMetadata,
 }
```

### Un contrato de implementación DEBE incluir los siguientes campos on-chain

- `spec`: una cadena que DEBE tener el formato `nft-1.0.0` para indicar que un contrato de Token No Fungible  se adhiere a las versiones actuales de esta especificación de Metadatos. Esto permitirá a los consumidores de los Tokens No Fungibles el saber si soportan las caracterísitcas de un contrato dado.
- `name`: el nombre legible por humanos del token.
- `symbol`: el símbolo abreviado del contrato, como MOCHI o MV3
- `base_uri`: Puerta centralizada conocida para tener un acceso confiable a los activos de almacenamiento descentralizado a los que hace referencia `reference` o `media`. Puede ser usado por otras interfaces para la recuperación inicial de activos, incluso si estos frontends después replican los datos a sus propies nodos descentralizados, que se les recomienda hacer.

### Un contrato de implementación puede incluír los siguientes campos on-chain

Para `NFTContractMetadata`:

- `icon`: una pequeña imagen asociada con este token. Debe ser una [URL de datos](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs), para ayudar a los consumidores a mostrarla rápido mientras se protegen los datos del usuario. Recomendación: use [SVG optimizados](https://codepen.io/tigt/post/optimizing-svgs-in-data-uris), que pueden resultar en imágenes de alta resolución con solo cientos de bytes de [costo de almacenamiento](https://docs.near.org/docs/concepts/storage-staking). (Note que estos costos de almacenamiento son incurridos al dueño/desplegador del token, pero que a la vez la consulta de estos íconos es una operación de lectura muy barata y cacheable para todos los consumidores del contrato en los nodos RPC que sirven los datos). Recomendación: cree íconos que funcionen bien con sitios web en modo claro y en modo oscuro, si bien usando esquemas de colores de tono medio, o [incrustando consultas `media` en el SVG](https://timkadlec.com/2013/04/media-queries-within-svg/).
- `reference`: un enlace a un archivo JSON válido conteniendo varias llaves ofreciendo detalles suplementarios en el token. Ejemplo: "/ipfs/QmdmQXB2mzChmMeKY47C43LxUdg1NDJ5MWcKMKxDu7RgQm", "https://example.com/token.json", etc. Si la información dada en este documento tiene conflictos con los atributos on-chain, los valores en `reference` se considerarán como la fuente de la verdad.
- `reference_hash`: el hash sha256 codificado en base64 del archivo JSON contenido en el campo `reference`. Esto es para protegerse contra la manipulación off-chain.

Para `TokenMetadata`:

- `name`: El nombre de este token en específico.
- `description`: Una descripción más larga del token.
- `media`: La URL asociada a los medios. Preferiblemente a un almacenamiento descentralizado dirigido a contenido.
- `media_hash`: el hash sha256 codificado en base64 del contenido al que hace referencia el campo `media`. Esto es para protegerse contra la manipulación off-chain.
- `copies`: El número de tokens con este conjunto de metadatos o `media` que se sabe que existen en el momento del minting.
- `issued_at`: Un epoch Unix en milisegundos de cuando el token fue emitido o minteado (un entero unsigned de 32-bit será suficiente hasta el año 2106)
- `expires_at`: Un epoch Unix en milisegundos de cuando el token expire
- `starts_at`: Un epoch Unix en milisegundos de cuando el token empiece a ser válido
- `updated_at`: Un epoch Unix en milisegundos de cuando el token fue actualizado por última vez
- `extra`: cualquier cosa extra que el NFT quiera guardar on-chain. Puede ser un JSON con formato de cadena.
- `reference`: URL a un archivo JSON off-chain con más información.
- `reference_hash`: Hash sha256 codificado en Base64 de JSON del campo de referencia. Requerido si `reference` está incluído.

### Sin costo incurrido por el comportamiento NFT central

Los contratos deberían ser implementados de una manera en la que se eviten tarifas de gas extras por serialización y deserialización de metadatos por llamadas a los métodos `nft_*` que no sean `nft_metadata` o `nft_token`. Vea `near-contract-standards` [implementación usando `LazyOption`](https://github.com/near/near-sdk-rs/blob/c2771af7fdfe01a4e8414046752ee16fb0d29d39/examples/fungible-token/ft/src/lib.rs#L71) como ejemplo de referencia.

## Desventajas

* Cuando este contrato de NFT es creado e incializado, el almacenamiento por token será mayor que una versión NFT Core. Las interfaces pueden dar cuenta de esto agregando depósitos extras mientras se mintea. Esto solo se puede realizar rellenando con una cantida razonable, o con la interface usando la [llamada RPC detallada aquí](https://docs.near.org/docs/develop/front-end/rpc#genesis-config) que obtiene configuraciones génesis y determina precisamente cuanto depósito se necesita.
* El estándar de que `icon` sea una URL de datos en lugar de un enlace endpoint HTTP que podría contener código que viola la privacidad no puede ser desplegado o actualizado desde los metadatos del contrato, y se debe de hacer en el lado del consumidor/aplicación cuando se muestra los datos del token.
* Si un ícono on-chain usa una URL de datos o si no está establecido pero el documento dado por reference contiene una URL de un icon que viola la privacidad, los consumidores y aplicaciones de esta información no deberían ingenuamente mostrar la versión de reference y preferir la versión segura. Esto es técnicamente una violación de la política de "reference estableciendo ganancias" descrita anteriormente.

## Posibilidades futuras

- Estándares detallados podrían ser aplicados para las versiones.
- Un esquema detallado de lo que debe contener el objeto reference.

## Errata

La primera versión (`1.0.0`) tenía un lenguaje confuso en relación a los campos:
- `issued_at`
- `expires_at`
- `starts_at`
- `updated_at`

Se le dió a esos campos el tipo `string|null` pero no estaba claro si debería de ser un epoch Unix en milisegundos o [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html). Al tener que revisitar esto, se determinó que usar milisegundos epoch era lo más eficiente ya que reducirá el cálculo en el contrato inteligente y puede ser derivado trivialmente de la marca de tiempo del bloque.
