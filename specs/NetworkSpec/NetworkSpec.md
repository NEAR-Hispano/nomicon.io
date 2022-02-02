# Red

La capa de red constituye el nivel bajo del Protocolo Near y es la última instancia responsable de transportar mensajes entre pares. Para proveer un enrutamiento eficiente mantiene una tabla de enrutamiento entre todos los pares activamente conectados a la red, y manda mensajes entre ellos usando los mejores caminos. Hay un mecanismo implementado, que permite a los nuevos pares que entran a la red, descubrir a otros pares, y rebalancea las conexiones de una manera en la que la latencia sea mínima. Las firmas criptográficas son usadas para verificar identidades de los pares participantes del protocolo debido a que es un sistema que no necesita de autorización.

Este documento debe servir como referencia para que todos los clientes implementen la capa de red.

## Mensajes

Estructuras de datos usados para los mensajes entre pares están enumeradas en [Mensajes](Messages.md).

## Descubriendo la red

Cuando un nodo se inicia por primera vez, trata de conectarse a una lista de nodos de arranque especificados en un archivo de configuración. La dirección de cada nodo.

Se espera que un nodo pida periócamente una lista de pares a sus nodos vecinos para aprender de los demás nodos en la red. Esto permitirá a cada nodo descubrirse entre ellos mismos, además de tener información relevante para tratar de establecer una conexión nueva con esto. Cuando un nodo recibe un mensaje de tipo [`PeersRequest`](Messages.md#peermessage) se espera que conteste con un mensaje de tipo [`PeersResponse`](Message.md#peermessage) con la información de pares sanos que este nodo conozca.

### Handshakes

Para establecer una conexión nueva entre un par de nodos, estos seguirán el siguiente protocolo. El nodo A abre una conexión con el nodo B y le envía un [Handshake](Messages.md#Handshake) (apretón de manos por su traducción al español). Si el handshake es válido (vea las razones para [declinar un handshake](#Decline-handshake)) el nodo B procederá a enviar un [Handshake](Messages.md#Handshake) al nodo A. Después de que los dos nodos acepten el handshake, marcará al otro como una conexión activa, hasta que uno de ellos pare la conexión.

El [Handshake](Messages.md#Handshake) contiene información relevante del nodo, la cadena actual y la información para crear un nuevo edge entre ambos nodos.

#### Declinar handshake

Cuando un nodo recibe un handshake de otro nodo, este declinará la conexión si una de las siguientes situaciones pasa:

1. El otro nodo tiene genésis diferente.
2. El nonce del edge es demasiado bajo.

#### Edge

Los edges son usados para hacerle saber a los demás nodos en la red que hay una conexión activa entre un par de nodos actualmente. Vea la definición de [esta estructura de datos](Messages.md#Edge).

Si el nonce del edge es impar, denota un edge `Added`, de lo contrario denota un edge `Removed`. Cada nodo debe de monitorear el nonce usado para los edges entre cada par de nodos. El par C cree que el par A y B actualmente están conectados si y solo si el edge con el nonce más alto que C conoce para ellos, tiene un nonce impar.

Cuando dos nodos se conectan exitosamente entre ellos, transmiten el edge nuevo para hacerles saber a los otros pares sobre esta conexión. Cuando un nodo es desconectado de otro nodo, debe de incrementar el nonce por 1, firmar el edge nuevo y transmitirlo para hacerle saber a los otros nodos que la conexión fue removida.

Una conexión eliminada será válida, si contiene información válida del edge agregado que está invalidando. Esto previene a los pares incrementar el nonce por más de uno cuando eliminan un edge.

Cuando el nodo A propone un edge al B con el nonce X, lo aceptará y lo firmará si:

- X = 1 y B no sabe de algún otro edge anterior entre A y B
- X es impar y X > Y donde Y es el nonce del edge con el nonce más alto entre A y B que B conoce.

<!-- TODO: What is a valid edge: pseudo code -->

## Tabla de enrutamiento

Cada nodo mantiene una tabla de enrutamiento con todas las conexiones existentes e información relevante para enrutar mensajes. El grafo explícito con todas las conexiones activas se almacena todo el tiempo.

```rust
struct RoutingTable {
    /// PeerId asociado para cada id de cuenta conocido
    account_peers: HashMap<AccountId, AnnounceAccount>,
    /// PeerId activo que son parte del camino más corto a cada PeerId.
    peer_forwarding: HashMap<PeerId, HashSet<PeerId>>,
    /// Almacenar la última actualización para los edges conocidos.
    edges_info: HashMap<(PeerId, PeerId), Edge>,
    /// Hash de los mensajes que requiere enrutamiento de regreso al salto anterior respectivo.
    route_back: HashMap<CryptoHash, PeerId>,
    /// Vista actual de la red. Los nodos son los pares y los edges son las conexiones activas.
    raw_graph: Graph,
}

```

- `account_peers` es un mapeo de cada cuenta conocida al corresponsal [anuncio](Messages.md#announceaccount). Dado que los validadores son conocidos por sus [AccountId](Messages.md#accountid), cuando un nodo necesita enviar un mensaje a un validador, encuentra el [PeerId](Messages.md#peerid) asociado con el [AccountId](Messages.md#accountid) en esta tabla.

- `peer_forwarding`: Para el nodo `S`, `peer_forwarding` constituye un mapeo de cada [PeerId](Messages.md#peerid) `T`, para el conjunto de pares que están directamente conectados a `S` y pertenecen a la ruta más corta, en términos de número de edges, entre `S` y `T`. Cuando el nodo `S` necesita de enviar un mensaje al nodo `T` y no están directamente conectados, `S` escoje un par entre el conjunto `peer_forwarding[S]` y le envía un mensaje enrutado con destino hacia `T`.

<!-- TODO: Add example. Draw a graph. Show routing table for each node. -->
<!-- TODO: Notice when two nodes are totally disconnected, and when two nodes are directly connected -->

- `edges_info` es un mapeo entre cada par desordenado de pares `A` y `B` con el edge con el nonce más alto conocido entre esos pares.

- `route_back` es usado para calvular la ruta para ciertos mensajes. Lea más acerca de esto en la sección [Enrutamiento de regreso](#routing-back).

- `raw_graph` es la representación explícita del [grafo](https://es.wikipedia.org/wiki/Grafo) de la red. Cada vértica de este grado es un par, y cada edge (borde) es una conexión activa entre un par de pares, e.j. el edge con el nonce más alto entre este par de pares es de tipo `Added`. Es usado para calcular el camino más corto entre la fuente de todos los demás pares.

### Actualizaciones

`RoutingTable` debe de ser actualizada acorde a cuando el nodo recibe actualizaciones de la red:

- Edges nuevos: el mapeo `edges_info` es actualizado con edges nuevos si su nonce es más alto. Esto es relevante para saber si una conexión nueva fue creada, o si alguna conexión fue detenida.

```python
def on_edges_received(self, edges):
    for edge in edges:
        # Verifica si el edge es válido
        if edge.is_valid():
            peer0 = edge.peer0
            peer1 = edge.peer1

            # Encuentra el nonce más alto que se conoce en este punto.
            current_edge = self.routing_table.edges_info.get((peer0, peer1))

            # Si no hay un edge conocido, o si el edge conocido tiene un nonce más chico
            if current_edge is None or current_edge.nonce < edge.nonce:
                # Actualiza con el nuevo edge
                self.routing_table.edges_info[(peer0, peer1)] = edge
```

- Cuenta anunciante: el mapeo `account_peers` es actualizado con los anuncios de los epochs más recientes.

```python
def on_announce_accounts_received(self, announcements):
    for announce_account in announcements:
        # Verifica que el anuncio es válido
        if announce_account.is_valid():
            account_id = announce_account.account_id

            # Encuentra el anuncio más reciente para el account_id que se está anunciando
            current_announce_account = self.routing_table.account_peers.get(account_id)

            # Si el epoch nuevo pasa después del epoch actual.
            if announce_account.epoch > current_announce_account.epoch:
                # Actualiza con el anuncio nuevo
                self.routing_table.account_peers[announce_id] = announce_account
```

- Mensajes de enrutamiento de regreso: Lea sobre esto en la sección [Enrutamiento de regreso](#routing-back)

### Enrutamiento

Cuando un nodo necesita enviar un mensaje a otro par, verifica en la tabla de enrutamiento si está conectado con ese par, posiblemente no lo haga directamente sino a través de saltos. Después selecciona uno de los caminos más cortos al par objetivo y envía un [`RoutedMessage`](Messages.md#routedmessage) al primer peer en el camino.

Cuando recibe un [`RoutedMessage`](Messages.md#routedmessage), verifica si es el objetivo, en ese caso consume el cuerpo del mensaje, de lo contrario encuentra una ruta al objetivo siguiendo el enfoque descrito y envía el mensaje de nuevo. Es importante que antes de enrutar un mensaje cada par verifica la firma del autor original del mensaje, pasar un mensaje con firma inválida podría resultar en un baneo del remitente. Sin embargo, no es requerido revisar el contenido del mensaje en sí.

Cada [`RoutedMessage`](Messages.md#routedmessage) está equipado con un entero "tiempo-para-vivir". Si este mensage no es para el nodo que lo está procesando, decrementa el campo por uno antes de enrutarlo; si el valor es 0, el nodo descarta el mensaje en lugar de enrutarlo.

#### Enrutamiento de regreso

Es posible que el nodo `A` sea conocido por `B` pero no al revés. En el caso de que el nodo `A` envíe una petición que requiere una respuesta de `B`, la respuesta se enruta de regreso a través del mismo camino usado para enviar el mensaje de `A` a `B`. Cuando un nodo recibe un [RoutedMessage](Messages.md#routedmessage) que requiere una respuesta, lo alamacena en el mapeo `route_back`, el hash del mensaje mapeado al [PeerId](Messages.md#peerid) del remitente. Después de que el mensaje llega al destino final y la respuesta es calculada, se enruta de regreso usando el hash del mensaje original como objetivo. Cuando un nodo recibe un Routed Message en el cual el objetivo es un hash, el nodo verifica el remitente anterior en el mapeo `route_back`, y envía el mensaje a el si existe, de otra manera descarta el mensaje.

El hash de un `RoutedMessage` que será guardado en el mapeo `route_back` calculado como:

```python
def route_back_hash(routed_message):
    return sha256(concat(
        borsh(routed_message.target),
        borsh(routed_message.author),
        borsh(routed_message.body)
    ))
```

```python
def send_routed_message(self, routed_message, sender):
    """
    sender: PeerId del nodo a través del cual el mensaje fue recibido.

    No confundir sender con el routed_message.author:
    routed_message.author es el PeerId del creador original del mensaje
    
    La única situación en la cual el sender == routed_message.author es cuando el mensaje no
    se recibió de la red, pero fue creado por el nodo y debe ser enrutado.
    """
    if routed_message.requires_response():
        crypto_hash = route_back_hash(routed_message)
        self.routing_table.route_back[crypto_hash] = sender

    next_peer = self.find_closest_peer_to(routed_message.target)
    self.send_message(next_peer, routed_message)

def on_routed_message_received(self, routed_message, sender):
    # routed_message.target is of type CryptoHash or PeerId
    if isinstance(routed_message.target, CryptoHash):
        # Esta es la respuesta para un mensaje enrutado.
        # `targer` es el PeerId que envió este mensaje.
        target = self.routing_table.route_back.get(routed_message.target)

        if target is None:
            # Descartar mensaje si no hay una ruta de regreso conocida
            return
        else:
            del self.routing_table.route_back[routed_message.target]
    else:
        target = routed_message.target

    if target == self.peer_id:
        self.handle_message(routed_message.body)
    else:
        self.send_routed_message(routed_message, sender)
```

### Sincronización

Cuando dos nodos se conectan entre sí, estos intercambian todos los edges conocidos (de `RoutingTable::edges_info`) y los anuncios de cuenta (de `RoutingTable::account_peers`).
También transmiten los edges creados recientemente a todos los nodos directamente conectados hacia ellos. Después de que un nodo se entera de un nuevo `AnnounceAccount` o un `Edge` nuevo, estos transmiten automáticamente esta información al resto de los nodos, para que así todos se mantengan actualizados.

## Seguridad

Los mensajes intercambiados entre pares (directos o enrutados) son criptográficamente firmados con la llave de acceso privado del remitente. El ID del nodo contiene la llave pública de cada nodo que permite a otros pares verificar los mensajes. Para mantener una comunicación segura la mayoría del mensaje requiere algún nonce/marca de tiempo que prohiba a un actor malicioso reutilizar un mensaje firmado fuera de contexto.

En el caso del enrutamiento de mensajes cada salto intermedio debe verificar que el hash del mensaje y las firmas son válidas antes de enrutarlas al siguiente salto.

### Comportamiento abusivo

Cuando un nodo A manda más de `MAX_PEER_MSG_PER_MIN` mensajes por minuto al nodo B, será baneado y no podrá seguir enviando mensajes hacia el. Esta es un mecanismo de protección en contra de los nodos abusivos para evitar el spameo por otros pares.

## Detalles de implementación

Hay algunos problemas que deben de ser manejados por la capa de red, pero los detalles acerca de como implementarlos no se aplican en el protocolo, sin embargo, aquí proponemos como abordarlos.

### Balanceando la red

[Github comment](https://github.com/nearprotocol/nearcore/issues/2395#issuecomment-610077017)
