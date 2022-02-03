# Metadata del Token Fungible ([NEP-148](https://github.com/near/NEPs/discussions/148))

Versión `1.0.0`

## Resumen
[summary]: #summary

Una interface para la metadata de los tokens fungibles. La meta es que la metadata sea a prueba de futuro así como ligera. Esto será importante para las dApps que necesiten información adicional acerca de las propiedades de un TF, y ampliamente compatible con otros estándares de tokens tal que [NEAR Rainbow Bridge](https://near.org/blog/eth-near-rainbow-bridge/) pueda mover tokens entre cadenas.

## Motivación

Los tokens fungibles personalizados juegan un rol importante en las aplicaciones descentralizadas al día de hoy. TF pueden tener propiedades personalizadas para diferenciarse de otros tokens o contratos en el ecosistema. En NEAR, muchas propiedades comunes pueden ser guardadas directamente on-chain. Otras propiedades son mejor guardarlas off-chain o en una plataforma de almacenamiento descentralizado, para así ahorrar en costos de almacenamiento y permitir una experimentación rápida de la comunidad.

Mientras la tecnología blockchain avanza, se vuelve cada vez más importante el proporcionar compatibilidad con versiones anteriores y un concepto de especificación. Este estándar cubre todas estas preocupaciones. 

Estado de la técnica:

- [EIP-1046](https://eips.ethereum.org/EIPS/eip-1046)
- [OpenZeppelin's ERC-721 Metadata standard](https://docs.openzeppelin.com/contracts/2.x/api/token/erc721#ERC721Metadata) también ayudo, aunque es para tokens no-fungibles.

## Explicación a nivel de guía

Un contrato inteligente de token fungible permite propiedades detectables. Algunas propiedades pueden ser determinadas por otros contratos on-chain, o regresar llamadas de método view.
Otros solo pueden ser determinados por un sistema oracle para ser usados on-chain, o por un frontent con la habilidad de aceder a un archivo de referencia ligada.

### Ejemplos de escenarios

### El token proporciona la metadata al momento del despliegue y la inicialización

Alice despliega un contrato de token fungible wBTC.

**Suposiciones**

- El contrato de token wBTC es `wbtc`
- La cuenta de Alice es `alice`.
- La presición ("decimals" en este estándar de metadatos) en el contrato wBTC es `10^8`.

**Explicación de alto nivel**

Alice emite una transacción para desplegar e inicializar el contrato de token fungible, proporcionando argumentos a la función de inicialización que establece los campos de la metadata.

**Llamadas técnivas**

1. `alice` despliega un contrato y llama a `wbtc::new` con toda la metadata. Si este despliegue e inicialización se hicieron usando [NEAR CLI](https://docs.near.org/docs/tools/near-cli) el comando sería:

    near deploy wbtc --wasmFile res/ft.wasm --initFunction new --initArgs '{
      "owner_id": "wbtc",
      "total_supply": "100000000000000",
      "metadata": {
         "spec": "ft-1.0.0",
         "name": "Wrapped Bitcoin",
         "symbol": "WBTC",
         "icon": "data:image/svg+xml,%3C…",
         "reference": "https://example.com/wbtc.json",
         "reference_hash": "AK3YRHqKhCJNmKfV6SrutnlWW/icN5J8NUPtKsNXR1M=",
         "decimals": 8
      }
    }' --accountId alice

## Explicación a nivel de referencia

Un contrato de token fungible que implementa el estándar de la metadata contendrá una función llamada `ft_metadata`.

```ts
function ft_metadata(): FungibleTokenMetadata {}
```

**Interface**:

```ts
type FungibleTokenMetadata = {
    spec: string;
    name: string;
    symbol: string;
    icon: string|null;
    reference: string|null;
    reference_hash: string|null;
    decimals: number;
}
```

**Un contrato de implementación DEBE incluir los siguientes campos on-chain**

- `spec`: una cadena. Debe ser `ft-1.0.0` para indicar que un contrato de Token Fungible se adhiere a las versiones actuales de esta Metadata y a las especificaciones del [Núcleo de Token Fungible](./FungibleTokenCore.md). Esto permitirá a los consumidores del Token Fungible saber si ellos soportan las características de un contrato dado.
- `name`: el nombre legible por humanos del token.
- `symbol`: la abreviación, como wETH o AMPL.
- `decimals`: utilizado en las interfaces para mostrar los dígitos significativos adecuados de un token. Este concepto está bien explicado en esta [publicación de OpenZeppelin](https://docs.openzeppelin.com/contracts/3.x/erc20#a-note-on-decimals).

**Un contrato de implementación puede incluír los siguientes campos on-chain**

- `icon`: una pequeña imagen asociada con este token. Debe ser una [URL de datos](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs), para ayudar a los consumidores a mostrarla rápido mientras se protegen los datos del usuario. Recomendación: use [SVG optimizados](https://codepen.io/tigt/post/optimizing-svgs-in-data-uris), que pueden resultar en imágenes de alta resolución con solo cientos de bytes de [costo de almacenamiento](https://docs.near.org/docs/concepts/storage-staking). (Note que estos costos de almacenamiento son incurridos al dueño/desplegador del token, pero que a la vez la consulta de estos íconos es una operación de lectura muy barata y cacheable para todos los consumidores del contrato en los nodos RPC que sirven los datos). Recomendación: cree íconos que funcionen bien con sitios web en modo claro y en modo oscuro, si bien usando esquemas de colores de tono medio, o [incrustando consultas `media` en el SVG](https://timkadlec.com/2013/04/media-queries-within-svg/).
- `reference`: un enlace a un archivo JSON válido conteniendo varias llaves ofreciendo detalles suplementarios en el token. Ejemplo: "/ipfs/QmdmQXB2mzChmMeKY47C43LxUdg1NDJ5MWcKMKxDu7RgQm", "https://example.com/token.json", etc. Si la información dada en este documento tiene conflictos con los atributos on-chain, los valores en `reference` se considerarán como la fuente de la verdad.
- `reference_hash`: el hash sha256 codificado en base64 del archivo JSON contenido en el campo `reference`. Esto es para protegerse contra la manipulación off-chain.

## Desventajas

- Se puede argumentar que `symbol` e incluso `name` podría pertenecer como clave/valor en el objeto JSON de `referencia`.
- El pedir que `icon` sea una URL de datos en lugar de un enlace a un endpoint HTTP que podría contener código que viola la privacidad no se puede hacer en la implementación o actualización de los metadatos del contrato, y se debe hacer en el lado del consumidor/aplicación cuando se muestren los datos del token.
- Si un ícono on-chain usa una URL de datos o si no está establecido pero el documento dado por `reference` contiene una URL de un `icon` que viola la privacidad, los consumidores y aplicaciones de esta información no deberían ingenuamente mostrar la versión de `reference` y preferir la versión segura. Esto es técnicamente una violación de la política de "`reference` estableciendo ganancias" descrita anteriormente.

## Posibilidades futuras

- Estándares detallados podrían ser aplicados para las versiones.
- Un esquema detallado de lo que debe contener el objeto `reference`.
