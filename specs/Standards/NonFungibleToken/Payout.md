# Estándar para una mecánica de pago de múltiples destinatarios en contratos NFT (NEP-199)
Version `2.0.0`.

Este estándar asume que el contrato NFT ha implementado
[NEP-171](https://github.com/near/NEPs/blob/master/specs/Standards/NonFungibleToken/Core.md) (Core) y [NEP-178](https://github.com/near/NEPs/blob/master/specs/Standards/NonFungibleToken/ApprovalManagement.md) (Gestión de Aprobación).

## Resumen
Una interface que permite a los contratos de token no fungible pedir que los contratos financieros paguen a múltiples receptores, haciendo posible implementaciones regalías flexibles.

## Motivación
Actualmente, los NFTs en NEAR soportan el campo `owner_id`, pero carecen de flexibilidad para la propiedad y mecánicas de pago más complejas, incluyendo pero no limitado a las regalías. Los contratos financieros, como los marketplaces, casas de subasta, y los contratos de préstamo NFT se beneficiarían de una interfaz estándar en los contratos productores de NFT para consultar a quién pagar, y cuanto pagar.

Por lo tanto, el objetivo central de este estándar es de definir un conjunto de métodos que los contratos financieros llamen, sin especificar como los contratos NFT definen la división de las mecánicas de pago, y una estructura de respuesta `Payout` estándar.

## Explicación a nivel guía

Esta extensión de pago agrega dos métodos a los contratos NFT:
- un método view: `nft_payout`, aceptando un `token_id` y algún `balance`, regresando el mapeo de `Payout` para el token dado.
- un método de llamada: `nft_transfer_payout`, aceptando todos los argumentos de `nft_transfer`, más un campo para algún `Balance` que calcula el `Payout`, llama a `nft_transfer`, y regresa el mapeo de `Payout`.

Los contratos financieros DEBEN validar varias invariantes en en el valor regresado
`Payout`:
1. El `Payout` regresado NO DEBE ser más largo que la longitud máxima dada (parámetro `max_len_payout`) si se proporciona. Pagos con longitudes excesivas pueden convertirse prohibitivamente costosas de gas. Los contratos financieros pueden especificar la longitud máxima del pago que el contrato va a respetar con el campo `max_len_payout` en `nft_transfer_payout`.
2. Los balances DEBEN sumar menos o igual al argumento `balance` en `nft_transfer_payout`. Si el balance suma menos que el argumento `balance`, el contrato financiero PODRÍA reclamar el restante para sí mismo.
3. La suma de los balances NO DEBE desbordarse. Esto es técnicamente igual al punto 2, pero se espera que los contratos financeros manejen esta posibilidad.

Los contratos financieros PUEDEN especificar su longitud máxima de pago a respetar.
Como mínimo, los contratos financieros NO DEBEN establecer su longitud máxima por debajo de 10.

Si el pago contiene direcciones que no existen, el contrato financiero PUEDE quedarse con esos fondos de pago desperdiciados.

Los contratos financieros PUEDEN obtener una parte del precio de vente del NFT como comisión, restando su parte del total del precio de venta, y llamando a `nft_transfer_payout` con el restante.

## Flujo de ejemplo
```
 ┌─────────────────────────────────────────────────────────────────┐
 │El propietario del token aprueba al marketplace para token_id "0"│
 ├─────────────────────────────────────────────────────────────────┘
 │  nft_approve("0",market.near,<SaleArgs>)
 ▼
 ┌───────────────────────────────────────────────┐
 │Marketplace vende token a user.near por 10N    │
 ├───────────────────────────────────────────────┘
 │  nft_transfer_payout(user.near,"0",0,"10000000",5)
 ▼
 ┌───────────────────────────────────────────────┐
 │El contrato NFT devuelve datos de pago         │
 ├───────────────────────────────────────────────┘
 │  Payout(<who_gets_paid_and_how_much)
 ▼
 ┌───────────────────────────────────────────────┐
 │El mercado valida y paga direcciones           │
 └───────────────────────────────────────────────┘
```

## Reference-level explanation
```rust
/// Un mapeo de cuentas NEAR a la cantidad que se le debe pagar a cada una, en
/// el evento de una venta de token. El mapeo del pago DEBE ser más corto que
/// la longitud mácima especificada por el contrato financiero que obtiene estos
/// datos de pago. Cualquier mapeo de longitud 10 o menos DEBE ser aceptado por
/// los contratos financieros, así que 10 es un límite superior seguro.
#[derive(Serialize, Deserialize)]
#[serde(crate = "near_sdk::serde")]
pub struct Payout {
  pub payout: HashMap<AccountId, U128>,
}

pub trait Payouts{
  /// Dado un `token_id` y un saldo denominado NEAR, devuelva la estructura `Payout`
  /// para el token dado. Entrar en pánico si la longitud del pago excede
  /// `max_len_payout.`
  fn nft_payout(&self, token_id: String, balance: U128, max_len_payout: u32) -> Payout;
  /// Dado un `token_id` y un saldo denominado NEAR, transferir el token
  /// y devuelva la estructura `Payout` para el token dado. Entrar en pánico si
  /// la longitud del pago excede `max_len_payout.`
  #[payable]
  fn nft_transfer_payout(
    &mut self,
    receiver_id: AccountId,
    token_id: String,
    approval_id: u64,
    balance: U128,
    max_len_payout: u32,
  ) -> Payout{
    assert_one_yocto();
    let payout = self.nft_payout(token_id, balance);
    self.nft_transfer(receiver_id, token_id, approval_id);
    payout
  }
}
```

Note que un NFT y un contrato financiero variarán según su implementación. Esto significa que algunos ciclos del CPU extras podrán ocurrir en uno que otro contrato NFT.
Además, un contrato financiero puede aceptar tokens fungibles, NEARs nativos, u otras entidades como pago. Transferir tokens NEAR nativos es menos costoso en gas que enviar tokens fungibles. Por estas razones, la longitud máxima de los pagos puede variar acorde a la personalización del contrato inteligente.

## Desventajas

Hay una introducción de confianza que el contrato llamando a `nft_transfer_payout` va a pagar a todas las partes previstas. Sin embargo, ya que el contrato que llama típicamente es algo como un marketplace usado por usuarios finales, actores maliciosos pueden ser encontrados más facilmente y podrían tener menos incentivos.
Hay una suposición que los contratos NFT entenderán los límites del gas y no permitirán un número de pagos que no pueden ser completados.

## Posibilidades futuras

En el futuro, el contrato NFT en sí, podría ser capáz de poner una transferencia NFT como un estado que sea "transferencia pendiente" hasta que todos los pagos sean otorgados. Esto mantendría tods la información dentro del NFT y removería la confianza.

## Errata

La versión `2.0.0` contiene el `approval_id` de `u64` intencionado en vez de la versión cadena de `U64`. Esto fue un descuido, pero ya que el estándar fue publicado meses antes de notarlo, el equipo pensó que subir la versión era lo mejor.
