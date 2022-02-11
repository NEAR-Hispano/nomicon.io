# Mensajes

Todos los mensajes enviados en la red son del tipo `PeerMessage`. Se codifican usando [Borsh](https://borsh.io/) que permite una estructura rica, de tamaño pequeño y un codificado/decodificado rápido. Para más detalles sobre las estructuras de datos usadas como parte de un mensaje vea el [código de referencia](https://github.com/nearprotocol/nearcore).

## Codificado

Un `PeerMessage` es convertido en un arreglo de bytes (`Vec<u8>`) usando la serialización borsh. Un mensaje codificado está conformado por 4 bytes con la longitud del `PeerMessage` serializado concatenado con el `PeerMessage` serializado.

Revise los detalles de la [especificación Borsh](https://github.com/nearprotocol/borsh#specification) para que vea como maneja cada estructura de datos.

## Estructura de datos

### PeerID

El id de un peer en la red es su [PublicKey](https://github.com/nearprotocol/nearcore/blob/master/core/crypto/src/signature.rs).

```rust
struct PeerId(PublicKey);
```

### PeerInfo


```rust
struct PeerInfo {
    id: PeerId,
    addr: Option<SocketAddr>,
    account_id: Option<AccountId>,
}
```

`PeerInfo` contiene información relevante para intentar conectarse a otro peer. [`SocketAddr`](https://doc.rust-lang.org/std/net/enum.SocketAddr.html) es una tupla de la forma: `IP:port`.

### AccountID

```rust
type AccountId = String;
```

### PeerMessage

```rust
enum PeerMessage {
    Handshake(Handshake),
    HandshakeFailure(PeerInfo, HandshakeFailureReason),
    /// Cuando un nonce fallido es usado par algún peer, este mensaje se envía de regreso como evidencia.
    LastEdge(Edge),
    /// Contiene cuentas e información de borde.
    Sync(SyncData),
    RequestUpdateNonce(EdgeInfo),
    ResponseUpdateNonce(Edge),
    PeersRequest,
    PeersResponse(Vec<PeerInfo>),
    BlockHeadersRequest(Vec<CryptoHash>),
    BlockHeaders(Vec<BlockHeader>),
    BlockRequest(CryptoHash),
    Block(Block),
    Transaction(SignedTransaction),
    Routed(RoutedMessage),
    /// Desconectado de forma correcta de otro peer.
    Disconnect,
    Challenge(Challenge),
}
```

### AnnounceAccount

Cada cuenta debe anunciar su cuenta

```rust
struct AnnounceAccount {
    /// ID de cuenta a ser anunciado.
    account_id: AccountId,
    /// PeerId del dueño de la cuenta.
    peer_id: PeerId,
    /// Este anuncio es solo válido para este `epoch`.
    epoch_id: EpochId,
    /// Firma usando la llave secreta asociada con el ID de cuenta.
    signature: Signature,
}
```

### Handshake

```rust
struct Handshake {
    /// Versión del protocolo
    pub version: u32,
    /// Versión de protocolo compatible más antigua.
    pub oldest_supported_version: u32,
    /// ID del peer del que envía.
    pub peer_id: PeerId,
    /// ID del peer del que recibe.
    pub target_peer_id: PeerId,
    /// Dirección de escucha del remitente.
    pub listen_port: Option<u16>,
    /// Cadena de información del Peer.
    pub chain_info: PeerChainInfoV2,
    /// Información para el nuevo borde.
    pub edge_info: EdgeInfo,
}
```

<!-- TODO: Make diagram about handshake process, since it is very complex -->

### Edge

```rust
struct Edge {
    /// Dado que los bordes no están dirigidos, `peer0 < peer1` debería mantenerse.
    peer0: PeerId,
    peer1: PeerId,
    /// Nonce para realizar el seguimiento la última actualización de este borde.
    nonce: u64,
    /// Firma de las partes validando el borde. Estas son la firma del borde agregado.
    signature0: Signature,
    signature1: Signature,
    /// Información necesaria para declarar un borde como removido.
    /// El bool dice que parte está removiendo el borde: false para el Peer0, true para el Peer1
    /// La firma de la parte que está removiendo el borde.
    removal_info: Option<(bool, Signature)>,
}
```

### EdgeInfo

```rust
struct EdgeInfo {
    nonce: u64,
    signature: Signature,
}
```

### RoutedMessage

```rust
struct RoutedMessage {
    /// ID del peer al que este mensaje va dirigido.
    /// Si `target` es un hash, este mensaje debe ser regresado.
    target: PeerIdOrHash,
    /// El remitente original de este mensaje
    author: PeerId,
    /// Firma del autor del mensaje. Si esta firma es inválida debemos banear al último
    /// remitente de este mensaje. Si el mensaje es inválido debemos banear al autor del mensaje.
    signature: Signature,
    /// El tiempo para vivir de este mensaje. Después de pasar algunos saltos este numero debe ser
    /// decrementado por 1. Si este número es 0, descartar este mensaje.
    ttl: u8,
    /// Mensaje
    body: RoutedMessageBody,
}
```

### RoutedMessageBody

```rust
enum RoutedMessageBody {
    BlockApproval(Approval),
    ForwardTx(SignedTransaction),

    TxStatusRequest(AccountId, CryptoHash),
    TxStatusResponse(FinalExecutionOutcomeView),
    QueryRequest {
        query_id: String,
        block_id_or_finality: BlockIdOrFinality,
        request: QueryRequest,
    },
    QueryResponse {
        query_id: String,
        response: Result<QueryResponse, String>,
    },
    ReceiptOutcomeRequest(CryptoHash),
    ReceiptOutComeResponse(ExecutionOutcomeWithIdAndProof),
    StateRequestHeader(ShardId, CryptoHash),
    StateRequestPart(ShardId, CryptoHash, u64),
    StateResponse(StateResponseInfo),
    PartialEncodedChunkRequest(PartialEncodedChunkRequestMsg),
    PartialEncodedChunkResponse(PartialEncodedChunkResponseMsg),
    PartialEncodedChunk(PartialEncodedChunk),
    Ping(Ping),
    Pong(Pong),
}
```

## CryptoHash

`CryptoHash` son objetos con 256 bits de información.

```rust
pub struct Digest(pub [u8; 32]);

pub struct CryptoHash(pub Digest);
```
