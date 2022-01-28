# Tiempo de ejecución

El tiempo de ejecución es usado para ejecutar contratos inteligentes y otras acciones creadas por los usuarios y preservar el estado entre ejecuciones.
Puede ser descrito desde tres ángulos diferentes: ir paso a paso por varios escenarios, describiendo los componentes del tiempo de ejecución,
y describiendo las funciones que el tiempo de ejecución realiza.

## Escenarios

- Transacciones financieras -- examinamos qué pasa cuando el tiempo de ejecución necesita procesar una simple transacción financiero;
- Función cross-contrato -- el escenario cuando el usuario llama a un contrato que a su vez llama a otro contrato.

## Componentes

Los componentes del tiempo de ejecución pueden ser descritos a través de los crates:

- `near-vm-logic` -- describe la interface que los contratos inteligentes usan para interactuar con la blockchain.
  Encapsula el comportamiento de la blockchain que es visible al contrato inteligente, ej. libre de reglas, reglas de acceso al almacenamiento, reglas de promesa;
- `near-vm-runner` crate -- envuelve a "Wasmer" que se encarga de la ejecucó del código del contrato inteligente. Expone la interface
  proporcionada por `near-vm-logic` al contrato inteligente;
- `runtime` crate -- encapsula la lógica de cómo las transacciones y recibos deben de ser manejados. Si encuentra
  una llamada al contrato inteligente dentro de una transacción o recibo llama a `near-vm-runner`, para las demás acciones, como
  la creación de una cuenta, las procesa en el momento.

Los crates de utilidades son:

- `near-runtime-fees` -- un crate conveniente que encapsula la configuración de las tarifas. Tal vez nos deshagamos de el después;
- `near-vm-errors` -- contiene la jerarquía de errores que pueden ocurrir durante una transacción o procesamiento de recibo;
- `near-vm-runner-standalone` -- una herramiento ejecutable que permite la ejecución del tiempo de ejecución sin la necesidad de la blockchain, ej. para
  pruebas de integración de proyectos de L2;
- `runtime-params-estimator` -- genera los tiempos de ejecución y genera la configuración de las tarifas.

Aparte de eso, desde los componentes que describimos [la Especificación de Enlaces](BindingsSpec/BindingsSpec.md) que es una
parte importante del tiempo de ejecución que especifica las funciones que el contrato inteligente puede llamar desde su host -- el tiempo de ejecución.
La especificación está definida en `near-vm-logic`, pero está expuesta al contrato inteligente en `near-vm-runner`.

## Funciones

- Consumo y producción de recibos
- Tarifas
- Máquina virtual
- Verificación
