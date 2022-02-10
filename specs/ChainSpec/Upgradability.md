# Capacidad de actualización

Esta parte de la especificación describe los detalles de la actualización del protocolo, y toca algunas partes diferentes del sistema.

Los tres diferentes niveles para la capacidad de actualización son:
1. Actualizar sin algún cambio a las estructuras de datos subyacentes o al protocolo;
2. Actualizar cuando las estructuras de datos subyacentes cambiaron (configuración, base de datos o algo más interno al nodo y probablemente específico del cliente);
3. Actualizar con cambios en el protocolo a los cuales todos los nodos validadores deben de ajustarse.

## Versionado

Hay dos versiones importantes diferentes:
- Versión de binario que define sus estructuras de datos internas / base de datos y configuraciones. Esta versión es específica del cliente y no necesita empatar entre los nodos.
- Versión del protocolo, definiendo el "lenguaje" que los nodos están hablando.

```rust
/// Última versión del protocolo con la que este binario puede trabajar.
type ProtocolVersion = u32;
```

## Control de versiones del cliente

Los clientes deben seguir el [versionado semántico](https://semver.org/lang/es/).
Específicamente:
 - La versión PRINCIPAL define las versiones del protocolo.
 - La versión MENOR define los cambios que son específicos del cliente pero requieren migración de base de datos, cambio de configuración o algo similar. Esto incluye características específicas del cliente. El cliente debería ejecutar las migraciones al arrancar al detectar que la información en el disco fue producida por una versión anterior y automigrarla a la nueva.
  - La versión PATCH define cuándo se corrigen errores, lo que no debería requerir migraciones o cambios de protocolo.

Los clientes pueden definir que tan actual es la versión de los datos almacenados y las migraciones aplicadas.
La recomendación genera es almacenar la versión en la base de datos y al inicio del binario, revisar la versión de la base de datos y realizar las migraciones requeridas.

## Actualización del protocolo

Generalmente, manejamos la capacidad de actualización de las estructuras de datos a través de un contenedor de enumeración. Vea la estructura `BlockHeader` por ejemplo.

### Estructuras de datos versionadas

Dado que esperamos que muchas estructuras de datos cambien o sean actualizadas a la par de que el protocolo evoluciona, algunos cambios son requeridos para soportar eso.

La principal es agregar estructuras de datos `Versioned` compatibles con versiones anteriores como esta:

```rust
enum VersionedBlockHeader {
    BlockHeaderV1(BlockHeaderV1),
    /// Versión actual, donde `BlockHeader` es usado internamente para todas las operaciones.
    BlockHeaderV2(BlockHeader),
}
```

Donde `VersionedBlockHeader` será almacenado en el disco y se enviará por cable.
Esto permite codificar y decodificar versiones anteriores (hasta 256 según la especificación https://borsh.io). Si algunas estructuras de datos tienen más de 256 versiones, las versiones anteriores probablemente pueden ser retiradas y reusadas.

Internamente la versión actual es usada. Las versiones anteriores 
Internally current version is used. Las versiones anteriores tienen muchas interfaces / características que están definidas por diferentes componentes o se actualizan a la próxima versión (guardar para la validación de hash).

### Consenso

| Name | Value |
| - | - |
| `PROTOCOL_UPGRADE_BLOCK_THRESHOLD` | `80%` |
| `PROTOCOL_UPGRADE_NUM_EPOCHS` | `2` |

La manera en la que la versión será indicada por los validadores, será a través de

```rust
/// Agregar `version` al header del bloque.
struct BlockHeaderInnerRest {
    ...
    /// Última versión en la que se ejecuta el binario de nodo productor actual.
    version: ProtocolVersion,
}
```

La condición para cambiar a la siguiente versión del protocolo se basa en el % de stake `PROTOCOL_UPGRADE_NUM_EPOCHS` epochs antes indicadas sobre el cambio a la siguiente versión:

```python
def next_epoch_protocol_version(last_block):
    """Determina la siguiente versión del protocolo epoch dado el último bloque"""
    epoch_info = epoch_manager.get_epoch_info(last_block)
    # Encuentra epoch que decide si la versión debería cambiar caminando hacia atrás.
    for _ in PROTOCOL_UPGRADE_NUM_EPOCHS:
        epoch_info = epoch_manager.prev_epoch(epoch_info)
        # Stop if this is the first epoch.
        if epoch_info.prev_epoch_id == GENESIS_EPOCH_ID:
            break
    versions = collections.defaultdict(0)
    # Iterar sobre todos los bloques en el epoch anterior y recolectar la última versión para cada validador.
    authors = {}
    for block in epoch_info:
        author_id = epoch_manager.get_block_producer(block.header.height)
        if author_id not in authors:
            authors[author_id] = block.header.rest.version
    # Versiones de peso con el stake de cada validador.
    for author in authors:
        versions[authors[author] += epoch_manager.validators[author].stake
    (version, stake) = max(versions.items(), key=lambda x: x[1])
    if stake > PROTOCOL_UPGRADE_BLOCK_THRESHOLD * epoch_info.total_block_producer_stake:
        return version
    # De otra manera regresar la versión que fue usada en ese epoch decisivo.
    return epoch_info.version
```
