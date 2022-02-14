- Nombre de la propuesta: Nuevo algoritmo de intercambio de tablas de enrutamiento
- Fecha inicial: 6/21/2021
- NEP PR: [nearprotocol/neps#0000](https://github.com/near/nearcore/pull/4112)
- Problema(s): https://github.com/near/nearcore/issues/3838.
- GitHub PR: https://github.com/near/NEPs/pull/220

# Resumen
[resumen]: #summary

Actualmente, cada nodo en la red almacens su propia copia del [Grafo de Enrutamiento](NetworkSpec.md#Routing Table).
El proceso de intercambio y verificación de bordes requiere enviar una copia completa de la tabla de enrutamiento en cada sincronización.
Enviar la compia completa de la table de enrutamiento es un desperdicio, en esta especificación proponemos una solución, que perimite hacer sincronización parcial.
Esto debería reducir tanto la banda de ancha usada y la cantidad de tiempo de CPU.

Típicamente, solo una sincronización completa sería requerida cuando un nodo se conecta a la red por primera vez.
Para dos nodos dados que realizan una sincronización, proponemos un método, que requiere un ancho de banda/tiempo de procesamiento proporcional al tamaño de los datos que deben intercambiarse para los nodos dados.

# Algoritmo
En esta especificación describimos la implementación del algoritmo `Reconciliación de gráficos de enrutamiento` usando Filtros Bloom Inversos.
Una visión general de diferentes algoritmos `Reconciliación de gráficos de enrutamiento` pueden ser encontrados en [ROUTING_GRAPH_RECONCILIATION.pdf](https://github.com/pmnoxx/docs/blob/piotr-test-markdown/near/ROUTING_GRAPH_RECONCILIATION.pdf)

El proceso de intercambio de `tablas de enrutamiento` implica intercambiar bordes entre un par de nodos.
Para los nodos `A`, `B`, que implica agregar bordes que `A` tiene, pero `B` no y viceversa.
Para cada conexión creamos una estructura de datos [IbfSet](#IbfSet).
Almacena `Filtros Bloom Inversos` [Ibf](#Ibf) de potencias de `2^k` para `k` en el rango `10..17`.
Esto nos permite recuperar `2^17 / 1.3` bordes con el `99%` de probabilidad acorde con [Reconciliación de conjuntos eficiente sin contexto previo](https://www.ics.uci.edu/~eppstein/pubs/EppGooUye-SIGCOMM-11.pdf).
De otra manera, hacemos un intercambio completo de tabla de enrutamiento.

Hemos escogido un tamaño mínimo de `Ibf` el cual es `2^10`, porque la sobrecarga de procesamiento de `IBF` es menor que insignificante, y para reducir el número de mensajes requeridos en el intercambio.
Por el otro lado, limitamos el tamaño de `Ibf` a `2^17` para reducir el uso de memoria requerido para cada estructura de datos.


# Tabla de enrutamiento
Estamos heredando [Tabla de Enrutamiento](NetworkSpec.md#Routing Table) con el campo adicional `peer_ibf_set`

```rust
pub struct RoutingTable {
   /// Otros campos
   ///    ..
   /// Estructura de datos usada para el intercambio de tablas de enrutamiento.
   pub peer_ibf_set: IbfPeerSet,
}
```
- `peer_ibf_set` - una estructura de datos auxiliar, que permite hacer sincronización parcial.

# IbfPeerSet
`IbfPeerSet` contiene un mapeo entre `PeerId` y `IbfSet`.
Se usa para almacenar metadatos para cada peero, que puede ser usado para calcular la diferencia establecida entre las tablas de enrutamiento del peer actual y el peer contenido en el mapeo.
```rust
pub struct SimpleEdge {
  pub key: (PeerId, PeerId),
  pub nonce: u64,
}

pub struct IbfPeerSet {
    peer2seed: HashMap<PeerId, u64>,
    unused_seeds: Vec<u64>,
    seed2ibf: HashMap<u64, IbfSet<SimpleEdge>>,
    slot_map: SlotMap,
    edges: u64,
}
```
- `peer2seed` - contiene un mapeo de `PeerId` a `seed` usado para generar `IbfSet`
- `unused_seeds` - lista de las semillas sin usar actualmente, que no serán usadas para conexiones entrantes nuevas
- `seed2ibf` - contiene un mapeo de `seed` a `IbfSet`
- `slot_map` - una estructura de datos auxilar, que es usada para almacenar `Edges`, esto nos permite ahorrar memoria al no almacenar
  duplicados.
- `edges` - número total de bordes en la estructura de datos

# Messages

```rust
pub struct RoutingSyncV2 {
    pub version: u64,
    pub known_edges: u64,
    pub ibf_level: u64,
    pub ibf: Vec<IbfElem>,
    pub request_all_edges: bool,
    pub seed: u64,
    pub edges: Vec<Edge>,
    pub requested_edges: Vec<u64>,
    pub done: bool,
}
```
- `version` - versión de la tabla de enrutamiento, actualmente 0. Para el uso futuro
- `known_edges` - número de bordes que el peer conoce, esto nos permite decidir cuando solicitar todos los bordes
  inmediatamente o no.
- `ibf_level` - nivel de filtros inversos bloom solicitados de `10` a `17`.
- `request_all_edges` - true si el peer decide solicitar todos los bordes de otro peer
- `seed` - similla usada para generar el ibf
- `edges` - lista de bordes enviados al otro peer
- `requested_edges` - lista de hashes de bordes que el peer quiere obtener
- `done` - true si es el último mensaje en la sincronización


# IbfSet
Estructura usada para representar el conjunto de `Filtros Bloom Inversos` para el peer dado.
```rust
pub struct IbfSet<T: Hash + Clone> {
    seed: u64,
    ibf: Vec<Ibf>,
    h2e: HashMap<u64, SlotMapId>,
    hasher: DefaultHasher,
    pd: PhantomData<T>,
}
```
- `seed` - semilla usada para generar el IbfSet
- `ibf` - lista de `Filtros Bloom Inversos` con tamaños de `10` a `17`.
- `h2e` - mapeo de hash del borde dado al id del borde. Esto es usado para ahorrar memoria, para evitar guardar múltiples
  copies del borde dado.
- `hasher` - función de hashing

# Ibf
Estructura que representa los Filtros Bloom Inversos.
```rust
pub struct Ibf {
    k: i32,
    pub data: Vec<IbfElem>,
    hasher: IbfHasher,
    pub seed: u64,
}
```
- `k` - número de elementos en el vector
- `data` - vector de elementos del filtro bloom
- `semilla` - semilla de cada filtro bloom inverso
- `hasher` - función de hashing

# IbfElem
Elemento del filtro bloom
```rust
pub struct IbfElem {
    xor_elem: u64,
    xor_hash: u64,
}
```
- `xor_element` - xor de los hashes de los bordes almacenados en la caja dada
- `xor_hash` - xor de los hashes, de los hashes de los bordes almacenados en la caja dada

# Intercambio de la tabla de enrutamiento

El proceso de intercambiar tablas de enrutamiento involucra múltiples pasos.
Puede ser reducido si es necesario agregando un estimador de cuantos bordes difieren, pero no está implementado para mantener simplicidad.

## Paso 1
Peer `A` se conecta a peer `B`.

## Paso 2
Peer `B` escoje una `seed` única, y genera `IbfPeerSet` basado en la semilla y todos los bordes conocidos en la
[Tabla de Enrutamiento](NetworkSpec.md#Routing Table).

## Paso 3 - 10
En pasos impares peer `B` le envía mensajes a peer `A`, con un Ibf de tamaño `2^(step+7)`.
En pasos pares, peer `A` lo hace.
- Caso 1 - si no podemos encontrar todos los bordes continuamos con el siguiente paso
- Caso 2 - de lo contrario, sabemos cuál es la diferencia entre conjuntos.
Calculamos el conjunto de hashes de los bordes dados los cuales el nodo conoce, y también de los que no conoce.
El peer actual envía la respuesta, y el intercambio de tabla de enrutamiento finaliza.

## Paso 11

En caso de que la sincronización no esté realizada aún, cada lado envía una lista de hashes de bordes que conoce.

## Paso 12

Cada lado envía la lista de bordes que el otro lado solicitó.

# Consideraciones de seguridad


### Evitar calculación extra si otro peer se vuelve a conectar al servidor
Para evitar la calculación extra cuando un peer se vuelve a conectar a un servidor, podemos mantener un pool de estructuras `IbfPeerSet`.
El número de dichas estructuras que necesitamos mantener va a ser igual al número máximo de conexiones entrantes, que los servidores han tenido más el número de conexiones salientes.

### Produciendo de bordes, donde hay colisión de hashes.
Usamos una estructura `IbfPeerSet` única, para cada conexión, esto previene que se adivinen las semillas.
Por lo tanto, somos resistentes a ese tipo de ataque.


# Sobrecarga de memoria
Para cada conexión necesitamos aproximadamente `(2^10 + ... 2^17) * sizeof(IbfElem) bytes = 2^18 * 16 bytes = 4 MiB`.
Asumiendo que mantenemos `40` extra de dicha estructura de datos, necesitaríamos `160 MiB` extra.

# Sobrecarga de rendimiento
En cada actualización necesitamos actualizar cada estructura `IbfSet` `3*8 = 24` veces.
Asumiento que mantenemos `40` de dichas estructuras de datos, eso requiere hasta `960` actualizaciones.

# Enfoques alternativos

### Solo usar una estructura `IbfPeerSet` para todos
Esto reduciría el número de actualizaciones que necesitamos hacer, y el uso de memoria.
Sin embargo, sería posible adivinar el hash de la función usada, que nos podría exponer a una vulnerabilidad de seguridad.
Podría ser simple producir dos bordes, de modo que haya una colisión de hash, que causaría que el intercambio de tabla de enrutamiento fallara.

### Incrementar el número de estructuras `IbfSet` por cada `IbfPeerSet`
En teoría, podríamos incrementar los tamaños de las estructuras `IBF` usadas de `2^10..2^17` a `2^10..2^20`.
Esto nos permitiría recuperar la diferencia de conjunto si llega hasta `2^20/3` en lugar de `2^17/3` a costa de aumentar la sobrecarga de memoria de `160 MiB a 640 Mib`.

### Siplificar el algoritmo para solo intercambiar la listas de hashes de hashes conocidos de cada peer
Cada borde tiene un tamaño de aproximadamente 400 bytes.
Asumamos que necesitamos enviar 1 millón de bordes.
Al enviar una lista de hashes de 4 bytes de todos los bordes conocidos en cada sincronización solo necesitaríamos enviar `4 MiB` de metadatos más el tamaño de los bordes que difieren.
Este enfoque sería más simple, pero no tan eficiente en términos de banda ancha.
Eso seguiría siendo una mejora de tener que enviar solo `4 MiB` sobre `400 MiB` con la implementación existente.

# Mejoras futuras

## Reducir uso de memoria
### Teóricamente, usando un número ajustado de estructuras `IbfPeerSet` para cada nodo, un número menor de conjuntos podría ser usado. Por ejemplo 10, esto requeriría la estimación de qué tan probable es producir bordes que puedan llegar a producir colisiones.
Todavía podría ser posible generar fuera de línea un conjunto de bordes que puede causar que la recuperación de bordes de `IBF` falle para un nodo.

