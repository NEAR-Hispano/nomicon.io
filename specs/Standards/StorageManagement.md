# Gestión de almacenamiento ([NEP-145](https://github.com/near/NEPs/discussions/145))

Versión `1.0.0`

NEAR usa [stakeo de storage] que significa que una cuenta de contrato debe tener el balance suficiente para cubrir todo el alamacenamiento agregado con el tiempo. Este estándar proporciona una manera uniforma para pasar el costo del almacenamiento a los usuarios. Permite a las cuentas y contratos:

1. Revisar el balance de almacenamiento de una cuenta.
2. Determinar el almacenamiento mínimo para agregar información de la cuenta tal que puede interactuar como se espera con un contrato.
3. Agregar balance de almacenamiento para una cuenta; si bien a sí misma o a otra.
4. Retirar algún depósito de almacenamiento removiendo información de la cuenta asociada de un contrato y después haciendo una llamada para remover el depósito no usado.
5. Cancelar el registro de una cuenta para recuperar el saldo de almacenamiento completo.

  [stakeo de storage]: https://docs.near.org/docs/concepts/storage-staking

Estado de la técnica:

- Un estándar anterior de token fungible ([NEP-21](https://github.com/near/NEPs/pull/21)) resultando como [el almacenamiento se pagó](https://github.com/near/near-sdk-rs/blob/1d3535bd131b68f97a216e643ad1cba19e16dddf/examples/fungible-token/src/lib.rs#L92-L113) para cuando se aumenta el allowance de un sistema escrow.

## Escenarios de ejemplo

Para mostrar la flexibilidad y el poder de este estándar, veamos dos contratos de ejemplo.

1. Un contrato de Token Fungible simple que usa la Gestión de Almacenamiento en un modo de "sólo registro", donde el contrato solo añade almacenamiento en la primera interacción de un usuario.
   1. La cuenta se registra a sí misma
   2. La cuenta registra a otra
   3. Intento innecesario para volver a registrar
   4. Forzar el cierre de la cuenta
   5. La cuenta cierra correctamente el registro
2. Un contrato de redes sociales, donde lo usuarios pueden agregar más información al contrato a lo largo del tiempo.
   1. La cuenta se registra a sí misma con más del mínimo requerido
   2. Intento innecesario para volver a registrar usando el parámetro `registration_only`
   3. Intento de toma de acción que excede el almacenamiento que ha sido pagado; incrementando el depósito de almacenamiento
   4. Remueve almacenamiento y reclama el exceso del depósito

### Ejemplo 1: Contraton de Token Fungible

Imagina un contrato de [token fungible](Tokens/FungibleTokenCore.md) desplegado en `ft`. Digamos que este contrato guarda todos los balances de usuario en una estructura de Mapa internamente, y agregar una llave para un un usuario nuevo requiere 0.00235Ⓝ. Este contrato por lo tanto usa el estándar de Gestión de Almacenamiento para pasar el costo a los usuarios, para que así un usuario nuevo pague una tarifa de registro, para interactuar con este contrato, de 0.00235Ⓝ, o 2350000000000000000000 yoctoⓃ ([yocto](https://www.metricconversion.us/prefixes.htm) = 10<sup>-24</sup>).

Para este contrato, `storage_balance_bounds` será:

```json
{
  "min": "2350000000000000000000",
  "max": "2350000000000000000000"
}
```

Esto significa que un usuario debe depositar 0.00235Ⓝ para interactuar con este contrato, y si intenta depositar más de eso no tendrá efecto alguno (los depósitos agregados serán inmediatamente reembolsados).

Vamos a seguir a dos usuarios, a Alice con la cuenta `alice` y Bob con la cuenta `bob`, y ellos interactúan con `ft` a través de los siguientes escenarios:

1. Alice se registra a sí misma
2. Alice registra a Bob
3. Alice trata de registrar a Bob otra vez
4. Alice forza el cierre de su cuenta
5. Bob cierra su cuenta sin problema

#### 1. La cuenta paga su tarifa de registro

**Explicación de alto nivel**

1. Alice revisa si está registrada con el contrato `ft`.
2. Alice determina la tarifa de registro necesaria para registrarse con el contrato `ft`.
3. Alice emite una transacción para depositar Ⓝ a su cuenta.

**Llamadas técnicas**

1. Alice consulta un método de solo-vista para determinar si ella tiene almacenamiento en este contrato con `ft::storage_balance_of({"account_id": "alice"})`. Usando [NEAR CLI](https://docs.near.org/docs/tools/near-cli) para hacer esta llamada view, el comando sería:

       near view ft storage_balance_of '{"account_id": "alice"}'

   La respuesta:

       null

2. Alice usa [NEAR CLI](https://docs.near.org/docs/tools/near-cli) para hacer la llamada view.

       near view ft storage_balance_bounds

   Como se mencinó anteriormente, esto mostrará que los valores de `min` y `max` son 2350000000000000000000 yoctoⓃ.

3. Alice convierte esta cantidad de yoctoⓃ a 0.00235 Ⓝ, después llama `ft::storage_deposit` con este depósito ligado. Usando NEAR CLI:

       near call ft storage_deposit '' \
         --accountId alice --amount 0.00235

   El resultado:

       {
         total: "2350000000000000000000",
         available: "0"
       }


#### 2. La cuenta paga por el almacenamiento de otra cuenta

Alice desea enviar eventualmente a `ft` tokens para Bob que no está registrado. Ella decide pagar por el almacenamiento de Bob.

**Explicación de alto nivel**

Alice emite una transacción para depostiar Ⓝ para a cuenta de Bob.

**Llamadas técnicas**
Alice llama `ft::storage_deposit({"account_id": "bob"})` con el depósito ligado de '0.00235'. Usando NEAR CLI el comando sería:

    near call ft storage_deposit '{"account_id": "bob"}' \
      --accountId alice --amount 0.00235

El resultado:

    {
      total: "2350000000000000000000",
      available: "0"
    }

#### 3. Intento inncesario de registrar a una cuenta ya registrada

Alice accidentalmente hace la misma llamada, e incluso omite un cero inicial en la cantidad de depósito.

    near call ft storage_deposit '{"account_id": "bob"}' \
      --accountId alice --amount 0.0235

El resultado:

    {
      total: "2350000000000000000000",
      available: "0"
    }

Adicionalmente, se le reembolsarán los 0.0235Ⓝ que ella agregó, porque el `storage_deposit_bounds.max` especifica que la cuenta de Bob no puede tener un balance total mayor a 0.00235Ⓝ.

#### 4. La cuenta forza el cierre del regitro

Alice decide que no importan sus tokens en `ft` y quiere recuperar su tarifa de registro forzosamente. Si el contrato permite esta operación, los tokens `ft` remanentes serán quemados o transferidos a otra cuenta, que ella podría o no podría tener la habilidad de especiicar antes de el cierre forzado.

**Explicación de alto nivel**

Alice emite una transacción para anular el registro de su cuenta y recuperar los Ⓝ de su tarifa de registro. Ella de agregar 1 yoctoⓃ, expresado en Ⓝ como `.000000000000000000000001`.

**Llamadas técnicas**
Alice llama a `ft::storage_unregister({"force": true})` con un depósito de 1 yoctoⓃ. Usando NEAR CLI el comando sería:

    near call ft storage_unregister '{ "force": true }' \
      --accountId alice --depositYocto 1

El resultado:

    true

#### 5. La cuenta cierra correctamente el registro

Bob quiere cerrar su cuenta, pero tiene un balance diferente de cero de tokens `ft`.

**Explicación de alto nivel**

1. Bob trata de cerrar correctamente su cuenta, llamando a `storage_unregister()` sin especificar `force=true`. Esto resulta en un error inteligible que le dice por qué no se puede anular aún el registro de su cuenta.
2. Bob manda todos su tokens `ft` a un amigo.
3. Bob vuelve a tratar de cerrar su cuenta correctamente. Funciona.

**Llamadas técnicas**

1. Bob llama a `ft::storage_unregister()` con un depósito de 1 yoctoⓃ. Usando NEAR CLI el comando sería:

       near call ft storage_unregister '' \
         --accountId bob --depositYocto 1

   Falla con un mensaje como "No se puede cerrar una cuenta correctamente con un balance positivo remanente; bob tiene un balance N"

2. Bob transfiere sus tokens a un amigo usando `ft_transfer` del estándar [Núcleo de token fungible](Tokens/FungibleTokenCore.md).

3. Bob trata de la llamada del paso 1 otra vez. Funciona

### Ejemplo 2: Contrato de redes sociales

Imagina un contrato inteligente de redes sociales que pasa el costo del almacenamiento a los usuarios para la información de publicaciones y seguidores. Digamos que este contrato está desplegado en la cuenta `social`. Como en el ejemplo de contrato de Token Fungible anterior, el `storage_balance_bounds.min` es 0.00235, porque este contrato también añadirá un usuario recientemente registrado a un Mapa interno. Sin embargo, este contrato no establece un `storage_balance_bounds.max`, como los usuarios pueden agregar más información al contrato a lo largo del tiempo y debe de cubrir el costo por este almacenamiento.

Así que para este contrato, `storage_balance_bounds` regresará:

```json
{
  "min": "2350000000000000000000",
  "max": null
}
```
Sigamos a un usuario, Alice con la cuenta `alice`, mientras ella interactúa con `social` a través de los siguientes escenarios:

1. Registro
2. Intento innecesario de volver a registrar usando el parámetro `registration_only`
3. Intento de tomar acción que exceda el almacenamiento ya pagado; incrementando el depósito de almacenamiento
4. Remosión del almacenamiento y reclamo del depósito excedente

#### 1. La cuenta se registra con `social`

**Explicación de alto nivel**

Alice emite una transacción para depositar Ⓝ para su cuenta. Mientras que el `storage_balance_bounds.min` para este contrato es 0.00235Ⓝ, la interfaz que ella usa sugiere agregar 0.1Ⓝ, para que así ella pueda agregar información inmediatamente a la aplicación, en lugar de *solo* registrarse.

**Llamadas técnicas**

Usando NEAR CLI:

    near call social storage_deposit '' \
      --accountId alice --amount 0.1

El resultado:

    {
      total: '100000000000000000000000',
      available: '97650000000000000000000'
    }

Aquí vemos que ella ha depositado 0.1Ⓝ y que 0.00235 de esa cantidad fue usado para registrar su cuenta, y es entonces retenido por el contrato. El resto está disponible para facilitar la interacción con el contrato, pero también puede ser retirado por Alice usando `storage_withdraw`.

#### 2. Intento innecesario para volver a registrar usando el parámetro `registration_only`

**Explicación de alto nivel**
Alice no puede recordar si ella ya se registró y reenvió la llamada, usando el parámetro `registration_only` para asegurarse de que no agregue otros 0.1Ⓝ.

**Llamadas técnicas**

Usando NEAR CLI:

    near call social storage_deposit '{"registration_only": true}' \
      --accountId alice --amount 0.1

El resultado:

    {
      total: '100000000000000000000000',
      available: '97650000000000000000000'
    }

Adicionalmente, Alice se le reembolsarán los 0.1Ⓝ extra que ella agregó. Esto facilita a los otros contratos siempre intentar registrar usuarios mientrar realizan transacciones por lotes sin preocuparse de errores o depósitos perdidos.

Note que Alice no incluyó `registration_only`, ella hubiera terminado con un `total` de 0.2Ⓝ.

#### 3. La cuenta incrementa el depósito del almacenamiento

Suposición: `social` tiene una función `post` que permite crear un post nuevo con un texto sin formato. Alice ha usado casi todo su balance de almacenamiento. Ella trata de llamar a `post` con una cantidad larga de texto, y se aborta la transacción porque ella necesita pagar por más almacenamiento primero.

Tenga en cuenta que, primeramente, las aplicaciones probablemente querrán evitar esta situación solicitando a los usuarios que recarguen los depósitos de almacenamiento antes de que el balance se acabe.

**Explicación de alto nivel**

1. Alice emite una transacción, digamos `social.post`, y falla con un error inteligible que dice que tiene un balance de almacenamientoe insuficiente para cubrir el corto de la operación
2. Alice emite una transacción para incrementar su balance de almacenamiento
3. Alice vuelve a tratar la transacción inicial y funciona

**Llamadas técnicas**

1. Esto está fuera del alcance de esta especificación, pero digamos que Alice llama a `near call social post '{ "text": "very long message" }'`, y que esto falla con un mensaje diciendo algo como "Depósito de almacenamiento insuficiente. Por favor llame a `storage_deposit` y agregue al menos 0.1 NEAR e intente otra vez."

2. Alice deposita la cantidad adecuada en una transacción al llamar a `social::storage_deposit` con el depósito agregado de '0.1'. Usando NEAR CLI:

       near call social storage_deposit '' \
         --accountId alice --amount 0.1

   El resultado:

       {
         total: '200000000000000000000000',
         available: '100100000000000000000000'
       }

3. Alice trata la llamada inicial `near call social post` otra vez. Funciona.

#### 4. Remoción de almacenamiento y relamo del depósito excedente

Suposición: Alice tiene más depositado de lo que usa.

**Explicación de alto nivel**

1. Alice ve su depósito de almacenamiento y ve que tiene un extra.
2. Alice retira su depósito excedente.

**Llamadas técnicas**

1. Alice consulta `social::storage_balance_of({ "account_id": "alice" })`. Con NEAR CLI:

       near view social storage_balance_of '{"account_id": "alice"}'

   Respuesta:

       {
         total: '200000000000000000000000',
         available: '100100000000000000000000'
       }

2. Alice llama a `storage_withdraw` con un depósito de 1 yoctoⓃ. comando de NEAR CLI:

       near call social storage_withdraw \
         '{"amount": "100100000000000000000000"}' \
         --accountId alice --depositYocto 1

   Resultado:

       {
         total: '200000000000000000000000',
         available: '0'
       }

## Explicación a nivel referencia

**NOTAS**:

- Todas las cantidades, balances y allowance son limitadas por `U128` (valor máximo 2<sup>128</sup> - 1).
- Este estándar de almacenamiento usa JSON para la serialización de argumentos y resultados.
- Las cantidades en los argumentso y resultados son serializados con cadenas en Base-10, e.j. `"100"`. Esto se hace para evitar la limitación de entero máximo de JSON de 2<sup>53</sup>.
- Para prevenir que el contrato desplegado sea modificado o eliminado, no debería tener llaves de acceso en su cuenta.

**Interfaz**:

```ts
// La estructura que será regresada por los métodos:
// * `storage_deposit`
// * `storage_withdraw`
// * `storage_balance_of`
// Los valores `total` y `available` son representaciones de cadena de enteros
// no firmados de 128-bits mostrando el balance de una cuenta específica en yoctoⓃ.
type StorageBalance = {
   total: string;
   available: string;
}

// The below structure will be returned for the method `storage_balance_bounds`.
// Both `min` and `max` are string representations of unsigned 128-bit integers.
//
// `min` is the amount of tokens required to start using this contract at all
// (eg to register with the contract). If a new contract user attaches `min`
// NEAR to a `storage_deposit` call, subsequent calls to `storage_balance_of`
// for this user must show their `total` equal to `min` and `available=0` .
//
// A contract may implement `max` equal to `min` if it only charges for initial
// registration, and does not adjust per-user storage over time. A contract
// which implements `max` must refund deposits that would increase a user's
// storage balance beyond this amount.
type StorageBalanceBounds = {
    min: string;
    max: string|null;
}

/************************************/
/* CHANGE METHODS on fungible token */
/************************************/
// Payable method that receives an attached deposit of Ⓝ for a given account.
//
// If `account_id` is omitted, the deposit MUST go toward predecessor account.
// If provided, deposit MUST go toward this account. If invalid, contract MUST
// panic.
//
// If `registration_only=true`, contract MUST refund above the minimum balance
// if the account wasn't registered and refund full deposit if already
// registered.
//
// The `storage_balance_of.total` + `attached_deposit` in excess of
// `storage_balance_bounds.max` must be refunded to predecessor account.
//
// Returns the StorageBalance structure showing updated balances.
function storage_deposit(
    account_id: string|null,
    registration_only: boolean|null
): StorageBalance {}

// Withdraw specified amount of available Ⓝ for predecessor account.
//
// This method is safe to call. It MUST NOT remove data.
//
// `amount` is sent as a string representing an unsigned 128-bit integer. If
// omitted, contract MUST refund full `available` balance. If `amount` exceeds
// predecessor account's available balance, contract MUST panic.
//
// If predecessor account not registered, contract MUST panic.
//
// MUST require exactly 1 yoctoNEAR attached balance to prevent restricted
// function-call access-key call (UX wallet security)
//
// Returns the StorageBalance structure showing updated balances.
function storage_withdraw(amount: string|null): StorageBalance {}

// Unregisters the predecessor account and returns the storage NEAR deposit.
//
// If the predecessor account is not registered, the function MUST return
// `false` without panic.
//
// If `force=true` the function SHOULD ignore existing account data, such as
// non-zero balances on an FT contract (that is, it should burn such balances),
// and close the account. Contract MAY panic if it doesn't support forced
// unregistration, or if it can't force unregister for the particular situation
// (example: too much data to delete at once).
//
// If `force=false` or `force` is omitted, the contract MUST panic if caller
// has existing account data, such as a positive registered balance (eg token
// holdings).
//
// MUST require exactly 1 yoctoNEAR attached balance to prevent restricted
// function-call access-key call (UX wallet security)
//
// Returns `true` iff the account was successfully unregistered.
// Returns `false` iff account was not registered before.
function storage_unregister(force: boolean|null): boolean {}

/****************/
/* VIEW METHODS */
/****************/
// Returns minimum and maximum allowed balance amounts to interact with this
// contract. See StorageBalanceBounds.
function storage_balance_bounds(): StorageBalanceBounds {}

// Returns the StorageBalance structure of the valid `account_id`
// provided. Must panic if `account_id` is invalid.
//
// If `account_id` is not registered, must return `null`.
function storage_balance_of(account_id: string): StorageBalance|null {}
```

## Drawbacks

- The idea may confuse contract developers at first until they understand how a system with storage staking works.
- Some folks in the community would rather see the storage deposit only done for the sender. That is, that no one else should be able to add storage for another user. This stance wasn't adopted in this standard, but others may have similar concerns in the future.

## Future possibilities

- Ideally, contracts will update available balance for all accounts every time the NEAR blockchain's configured storage-cost-per-byte is reduced. That they *must* do so is not enforced by this current standard.
