# Arquitectura

Un nodo Near consta aproximadamente de una capa de blockchain y una capa de tiempo de ejecución.
Estas capas fueron diseñadas para ser independientes entre sí: la capa de blockchain en teoría puede soportar el tiempo de ejecución que procesa transacciones de una manera diferente, tiene una máquina virtual diferente (e.g. RISC-V), tiene cuotas diferentes; por otro lado el tiempo de ejecución es ajeno al origen de la transacción. No es consciente de si la blockchain en la que se ejecuta está fragmentada, el consenso que ésta usa e incluso si corre o no corre como parte de una blockchain en sí.

La capa de blockchain y la capa de tiempo de ejecución comparten los siguientes componentes e invariantes:

## Transacciones y Recibos

Las transacciones y los recibos son un concepto fundamental dentro del Protocolo Near. Las transacciones representan acciones requeridas por el usuario de la blockchain, ej. enviar activos, crear una cuenta, ejecutar un método, etc. Los recibos, por otra parte, son una estructura interna; piensa en el recibo como un mensaje que es usado dentro de un sistema de paso de mensajes.

Las transacciones son creadas por fuera del nodo de Protocolo Near, hechas por el usuario que las envía por medio de RPC o por la red de comunicación.
Los recibos son creados por el tiempo de ejecución desde las transacciones o como resultado del procesamiento de otros recibos.

La capa de blockchain no puede crear procesos o procesar transacciones y recibos, solo puede manipularlos moviéndolos y alimentando así a un tiempo de ejecución.

## Sistema basado en cuentas

Similar a Ethereum, el Protocolo Near es un sistema basado en cuentas. Esto significa que cada usuario de blockchain es asociado aproximadamente por una o varias cuentas (aunque hay excepciones, cuando los usuarios comparten una cuenta y después son separados a través de las llaves de acceso por ejemplo).

El tiempo de ejecución esencialmente es un conjunto de reglas complejo que dictan qué hacer con las cuentas basado en la información de las transacciones y recibos. Por lo tanto, es profundamente consciente del concepto de cuenta.

Sin embargo la capa de blockchain es consciente mayormente de las cuentas que están dentro del (véase abajo) y los validadores (véase abajo).
Aparte de estos dos, esta capa no opera directamente en las cuentas.

### Asume que toda cuenta pertenece a su propio fragmento

Toda cuenta en NEAR pertenece a un fragmento.
Toda la información relacionada a la cuenta pertenece también al mismo fragmento.  La información incluye:

- Balance
- Balance bloqueado (para staking)
- Código del contrato
- Almacenamiento llave-valor del contrato
- Todas las llaves de acceso

El tiempo de ejecución asume, es la única información disponible para la ejecución del contrato.
Mientras que otras cuentas pueden pertenecer al mismo fragmento, el tiempo de ejecución nunca las usa o las provee durante la ejecución del contrato.
Entonces podemos asumir que cada cuenta pertenece a su propio fragmento. Así que no hay razón para intencionalmente tratar de colocar cuentas.

## Trie

El Protocolo Near es una blockchain de estado -- hay un estado asociado a cada cuenta y las acciones que los usuarios realizan mutan el estado de esta. El estado, entonces, es almacenado como un trie, y ambos, la capa de blockchain y la capa de tiempo de ejecución son conscientes de este detalle técnico.

La capa de blockchain manipula directamente el trie. Particiona el trie entre los fragmentos para distribuir la carga de trabajo.
Sincroniza el trie entre los nodos, y eventualmente, será responsable de mantener la consistencia del trie entre los nodos a través de su mecanismo de consenso y otros métodos de teoría de juegos.

La capa de tiempo de ejecución también es consciente de que el almacenamiento que usa para realizar operaciones es un trie. Generalmente no tiene que saber este detalle técnico y en teoría podríamos tener tener el trie abstraído como un valor “llave-valor” genérico.
Sin embargo, permitimos algunas operaciones específicas al trie las cuales están expuestas a los desarrolladores de contratos inteligentes para que así utilicen el Protocolo Near a su máxima eficiencia.

## Tokens y gas

Aunque los tokens son un concepto fundamental de la blockchain, están perfectamente encapsulados dentro de la capa de tiempo de ejecución junto con el gas, las cuotas y las recompensas.

La única forma por la cual la capa de blockchain puede estar consciente de los tokens y el gas es a través de la computación de la tasa de cambio y la inflación que están basados estrictamente en el mecanismo de producción de bloques.

## Validadores

La capa de blockchain y la capa de tiempo de ejecución tienen conciencia o están al tanto de un grupo especial de participantes los cuales son los responsables de mantener la integridad del Protocolo Near. Estos participantes tienen cuentas asociadas y son recompensadas respectivamente. La capa de tiempo de ejecución está al tanto de las recompensas, mientras que todo lo que conlleva un validador está dentro de la capa de blockchain.

## Conceptos de la capa de blockchain

Curiosamente, los siguientes conceptos sólo son para la capa de blockchain, la capa de tiempo de ejecución no sabe de ellos:

- Fragmentación -- la capa de tiempo de ejecución no sabe que está siendo usada dentro de una blockchain fragmentada, ej. no sabe que en el trie en la que está       funcionando es solo una parte del estado completo de la blockchain;
- Bloques o fragmentos -- la capa de tiempo de ejecución no sabe que los recibos que procesa constituyen un fragmento y que los recibos resultantes serán usados en   otros fragmentos. Desde la perspectiva del tiempo de ejecución, solo consume y regresa lotes de transacciones y recibos;
- Consenso -- la capa de tiempo de ejecución no sabe cómo se mantiene la consistencia del estado;
- Comunicación -- la capa de tiempo de ejecución no sabe nada de la topología de red actual. El recibo tiene un “receiver_id” (una cuenta receptora), pero no sabe nada acerca del fragmento al que llegará, así que el destino al que irá cada recibo en particular es la responsabilidad de la capa de blockchain.

## Conceptos de la capa de tiempo de ejecución

- Cuotas y recompensas -- las cuotas y recompensas están perfectamente encapsuladas dentro de la capa de tiempo de ejecución. La capa de blockchain, por el contrario, tiene un conocimiento indirecto de estas a través de la computación de la tasa token-gas y la inflación.
