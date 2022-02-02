# Transacción financiera

Supongamos que Alice quiere transferir 100 tokens a Bob.
En este caso estamos hablando de tokens de Near Protocol nativos, opuesto a los tokens definidos por el usuario implementeados a través de contratos inteligentes.
Hay varias formas en las que esto se puede hacer:

- Transferencia directa a través de una transacción que contenga un acción de transferencia;
- Alice llamando a un contrato inteligente que a su vez crea una transacción financiera hacia Bob.

En esta sección hablaremos acerca del escenario anterior más simple.

## Pre-requisitos

Para que esto funcione Alice y Bob necesitan tener _cuentas_ y un acceso a ellas a través de
_las llaves de acceso completo_.

Supongamos que Alices tiene la cuenta `alice_near` y Bob tiene la cuenta `bob_near`. También, hace ya tiempo,
cada uno de ellos creó una public-secret key-pair (par de llaves públicas-privadas), guardó la llave secreta en algún lugar (e.j. en una aplicación de billetera)
y creó una llave de acceso completo con la llave pública de la cuenta.

También necesitamos asumir que Alice y Bob tienen algún número de tokens en sus cuentas. Alice necesita >100 tokens en su cuenta
para que así ella pueda transferir 100 tokens a Bob, pero también Alice y Bob necesitan tener algunos tokens para pagar por la _renta_ de sus cuentas --
que escencialmente es el costo del almacenamiento ocupado por la cuenta en la red de Near Protocol.

## Creando un transacción

Para enviar la transacción Alice o Bob necesitan ejecutar un nodo.
Sin embargo, Alice necesita una manera de crear y firmar una estructura de transacción.
Supongamos que alice usa near-shell o alguna otra herramienta de terceros para eso.
La herramiento entonces crea la siguiente estructura:

```
Transaction {
    signer_id: "alice_near",
    public_key: "ed25519:32zVgoqtuyRuDvSMZjWQ774kK36UTwuGRZMmPsS6xpMy",
    nonce: 57,
    receiver_id: "bob_near",
    block_hash: "CjNSmWXTWhC3EhRVtqLhRmWMTkRbU96wUACqxMtV1uGf",
    actions: vec![
        Action::Transfer(TransferAction {deposit: 100} )
    ],
}
```

Que contiene una acción de transferencia de token, el ide de la cuenta que firma la transacción (`alice_near`)
la cuenta hacia la que va la transacción (`bob_near`). Alice también usa la llave pública
asociada con una de las llaves de acceso completo de la cuenta `alice_near`.

Adicionalmente, Alice usa el _nonce_ que es un valor único que permite a Near Protocol diferenciar transacciones (en caso de que haya varias transacciones entrando rápidamente) 
que debería estar incrementando estrictamente con cada transacción. A diferencia de Ethereum, donde los nonces son asociados con las llaves de acceso, opuesto a
las cuentas, con esto varios usuarios usando las mismas cuentas a través de diferentes llaves de acceso no necesitan preocuparse de reusar accidentalmente
los nonces de las demás personas.

El hash del bloque es usado para calcular la transacción "freshness". Es usado para asegurarse de que la transacción no
se pierda (en algún lugar de la red por ejemplo) y despues llegar horas, días o años después cuando dejó de ser relevante
o indeseable de ejecutar. La transacción no necesita llegar a un bloque en específico, por el contrario se requiere que llegue
a un número determinado de bloques desde el bloque identificado por el `block_hash` (desde 2019-10-27 el valor constante es 10 bloques).
Cualquier transacción llegando por debajo de este límite es considerada inválida.

near-shell u otra herramienta que Alice usa firma la transacción: calculando el hash de la transacción y firmándola
con la llave secreta, resultando en un objeto de tipo `SignedTransaction`.

## Enviando la transacción

Para enviar la transacción, near-shell se conecta a través de RPC a cualquier node de Near Protocol y lo envía.
Si los usuarios quieren esperar hasta que la transacción sea procesada, ellos pueden usar el método JSONRPC `send_tx_commit` que espera por
que la transacción aparezca en un bloque. De otra manera el usuario puede usar `send_tx_async`.

## De transacción a recibo

Nos saltamos los detalles acerca de como la transacción llega para ser procesada por el tiempo de ejecución, porque es parte de la discusión acerca de la capa de blockchain.
Consideramos el momento en donde `SignedTransaction` está siendo trasladada a `Runtime::apply` del crate `runtime`.
`Runtime::apply` inmediatamente pasa la transacción a `Runtime::process_transaction`
que a su vez hace lo siguiente:

- Verifica que la transacción sea válida;
- Aplica los cargos reversibles e irreversibles a la cuenta `alice_near`;
- Crea un recibo con el mismo conjunto de acciones dirigidas hacia `bob_near`.

Los primeros dos pasos se realizan dentro del método `Runtime::verify_and_charge_transaction`.
Específicamente hace las siguientes revisiones:

- Verifica que `alice_near` y `bob_near` son ids de cuenta sintácticamente válidas;
- Verifica que la firma de la transacción es correcta basándose en el hash de la transacción ligada a la llave pública;
- Recupera el último estado de la cuenta `alice_near` y simúltaneamente revisa que exista;
- Recupera el estado de las llave de acceso que `alice_near` usó para firmar la transacción;
- Revisa que el nonce de la transacción es mayor al nonce de la última transacción ejecutada con esa llave de acceso;
- Checa si la cuenta que firmó la transacción es la misma que cuenta que la recibe. En nuestro caso el remitente (`alice_near`) y el receptor
(`bob_near`) no son los mismos. Aplicamos diferentes tarifas si el receptor y el remitente son la misma cuenta;
- Aplica la renta del almacenamiento a la cuenta `alice_near`;
- Calcula cuanto gas necesitamos gastar para convertir esta transacción a un recibo;
- Calcula cuanto balance necesitamos substraer de `alice_near`, en este caso son 100 tokens;
- Deduce los tokens y el gas del balance de `alice_near`, usando el precio actual del gas;
- Revisa si después de todas estas operaciones la cuenta tiene el balance suficienta para pagar por la renta de los siguientes bloques
  (una constante enconómica definida por el Protocolo Near). De lo contrario la cuenta estará libre para una eliminación inmediata, cosa que no queremos;
- Actualiza la cuenta `alice_near` con el balance nuevo y usa las llaves de acceso usadoc con un nonce nuevo;
- Calcula la recompensa que debe ser pagada a los validadores por el gas quemado.

Si alguna de las operaciones falla, se revertirán todos los cambios.

## Procesando el recibo

El recibo creado en la sección pasada eventualmente llegará al tiempo de ejecución en el fragmento que aloja a la cuenta `bob_near`.
Nuevamente, será procesada por `Runtime::apply` que inmediatamente llamará a `Runtime::process_receipt`.
Revisará que este recibo no tiene dependencias de datos (que solo es el caso para las llamadas de función) y después llamara a `Runtime::apply_action_receipt` en `TransferAction`.
`Runtime::apply_action_receipt` realizará las siguientes revisiones:

- Recupera el estado de la cuenta `bob_near` si todavía existe (es posible que Bob haya borrado su cuenta al mismo tiempo que la transacción de transferencia);
- Aplica la renta de la cuenta de Bob;
- Calcula el costo de procesar un recibo y una acción de transferencia;
- Revisa si `bob_near` todavía existe y si deposita los tokens transferidos; 
- Calcula la recompensa que debe ser pagada a los validadores por el gas quemado.
