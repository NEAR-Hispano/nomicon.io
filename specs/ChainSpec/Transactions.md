# Transacciones en la capa de Blockchain

Un cliente crea una transacción, calcula el hash de la transacción y firma este hash para obtener una transacción firmada.
Ahora esta transacción firmada puede sern enviada a un nodo.

Cuando un nodo recibe una transacción firmada nueva, valida la transacción (si el nodo monitorea el fragmento) e informa a los peers. Eventualmente, la transacción válida es agregada al pool de transacciones.

Cada nodo validador tiene su pool de transacciones. La pool de transacciones mantiene transacciones que no son aún descartadas y aún no incluídas en la cadena.

Antes de producir un fragmento, las transacciones son ordenadas y validadas otra vez. Esto se hace para producir fragmentos con solo una transacción válida.

## Ordenamiento de transacciones

El pool de transacciones agrupa por un par de `(signer_id, signer_public_key)`.
El `signer_id` es el ID de la cuenta del usuario que firmó la transacción, el `signer_public_key` es la llave pública de la llave de acceso de la cuenta que fue usada para firmar las transacciones.
Las transacciones dentro de un grupo no están ordenadas.

El orden válido de la transacción en un fragmento es el siguiente:

- las transacciones están ordenadas en lotes.
- dentro de un lote todas las llaves de las transacciones deben ser diferentes.
- un conjunto de llaves de transacción en cada lote subsecuente debe ser un subconjunto de llaves del lote anterior.
- las transacciones con la misma llave deben ser ordenadas en orden estrictamente creciente de sus nonces correspondientes.

Nota:

- el orden dentro de un conjunto es indefinido. Cada nodo debe usar una semilla secreta única para ordenar a los usuarios que encuentren las claves más bajas para aprovechar cada nodo.

El pool de transacciones proporciona una estructura de drenaje que permite jalar transacciones en el orden apropiado.

## Validación de transacción

La validación de la transacción pasa dos veces, una vez antes de agregar a la pool de transacciones, la siguiente antes de agregar al fragmento.

### Antes de agregar a la pool de transacciones

Esto se hace para filtrar rápidamente transacciones que tienen una firma inválida o son inválidas en el estado más reciente.

### Antes de agregar a un fragmento

Un productor de fragmentos tiene que crear un fragmento con transacciones válidas y ordenados hasta ciertos límites.
Un límite es el número máximo de transacciones, otro es el total de gas quemado para las transacciones.

Para ordenar y filtrar transacciones, el productor de fragmentso obtiene un interador de pool y lo pasa al adaptador del tiempo de ejecución.
El adaptador del tiempo de ejecución jala transacciones una por una.
Las transacciones válidas son añadidas al resultado, las transacciones inválidas son descartadas.
Una vez que se llega al límite, todas las transacciones restantes del iterador son regresadas a la pool de transacciones.

## Iterador de la pool

El iterador de la pool es una característica que itera sobre los grupos de transacciones hasta que todos los grupos de transacciones quedan vacíos.
El iterador de la pool regresa una referencia mutable a un grupo de transacciones que implementa un iterador de drenaje.
El iterador de drenaje es como un iterador normal, pero remueve la entidad regresada del grupo.
Extrae transacciones del grupo en orden desde el nonce más pequeño hasta el más grande.

El iterador de la pool y los iteradores de drenaje para grupos de transacciones permiten al adaptador del tiempo de ejecución crear un orden apropiado.
Para cada grupo de transacción, el adaptador del tiempo de ejecución se queda extrayendo transacciones hasta que se encuentra la transacción válida.
Si el grupo de transaccion se queda vacío, entonces se salta.

El adaptador del tiempo de ejecución implementa el código siguiente para extraer todas las transacciones válidas:

```rust
let mut valid_transactions = vec![];
let mut pool_iter = pool.pool_iterator();
while let Some(group_iter) = pool_iter.next() {
    while let Some(tx) = group_iter.next() {
        if is_valid(tx) {
            valid_transactions.push(tx);
            break;
        }
    }
}
valid_transactions
```

### Ejemplo de ordenamiento de transacción usando el iterador de pool.

Digamos que:

- los IDs de cuenta son letras mayúsculas (`"A"`, `"B"`, `"C"` ...)
- las llaves públicas son letras minúsculas (`"a"`, `"b"`, `"c"` ...)
- los nonces son números (`1`, `2`, `3` ...)

Una pool podría tener un grupo de transacciones en el hashmap:

```
transactions: {
  ("A", "a") -> [1, 3, 2, 1, 2]
  ("B", "b") -> [13, 14]
  ("C", "d") -> [7]
  ("A", "c") -> [5, 2, 3]
}
```

Hay 3 cuentas (`"A"`, `"B"`, `"C"`). La cuenta `"A"` usó 2 llaves públicas (`"a"`, `"c"`). Otras cuentas usaron 1 llave pública cada una.
Las transacciones dentro de cada grupo pueden tener nonces repetidos mientras están en la pool.
Eso se debe a que la pool no filtra transacciones con el mismo nonce, solo transacciones con el mismo hash.

Para este ejemplo, digamos que las transacciones son válidas si el nonce es par y estrictamente mayor que el nonce anterior para la misma llave.

##### Inicialización

Cuando se llama a `.pool_iterator()`, un `PoolIteratorWrapper` nuevo se crea y retiene la referencia mutable a la pool,
para que así la pool no pueda ser modificada fuera de este iterador. El envoltorio se así:

```
pool: {
    transactions: {
      ("A", "a") -> [1, 3, 2, 1, 2]
      ("B", "b") -> [13, 14]
      ("C", "d") -> [7]
      ("A", "c") -> [5, 2, 3]
    }
}
sorted_groups: [],
```

`sorted_groups` es una fila de grupos transacciones ordenadas que ya fueron ordenadas y extraídas de la pool.

##### Transacción #1

El primer grupo a ser seleccionado es para la llave `("A", "a")`, el iterador de la pool ordena las transacciones por sus nonces y regresa la referencia mutable para el grupo. Los nonces ordenados son:
`[1, 1, 2, 2, 3]`. El tiempo de ejecución extrae `1`, después a `1`, y después a `2`. Las transacciones con el nonce `1` son inválidas por el nonce impar.

La transacción con el nonce `2` se agrega a la lista de las transacciones válidas.

El grupo de la transacción se descarta y el envoltorio del iterador de la pool se convierte en el siguiente:

```
pool: {
    transactions: {
      ("B", "b") -> [13, 14]
      ("C", "d") -> [7]
      ("A", "c") -> [5, 2, 3]
    }
}
sorted_groups: [
  ("A", "a") -> [2, 3]
],
```

##### Transacción #2

El siguiente grupo es para la llave `("B", "b")`, el iterador de la pool ordena las transacciones por sus nonces y regresa la referencia mutable para el grupo. Los nonces ordenados son: `[13, 14]`. El adaptador del tiempo de ejecución extrae `13`, después `14`. La transacción con el nonce `13` es inválida por el nonce impar.

La transacción con el nonce `14` se agrega a la lista de transacciones váldas.

El grupo de la transacción se descarta, pero está vacío, por lo que el iterador de la pool lo descarta por completo:

```
pool: {
    transactions: {
      ("C", "d") -> [7]
      ("A", "c") -> [5, 2, 3]
    }
}
sorted_groups: [
  ("A", "a") -> [2, 3]
],
```

##### Transacción #3

El siguiente grupo es para la llave `("C", "d")`, el iterador de la pool ordena las transacciones por sus nonces y regresa la referencia mutable para el grupo. Los nonces ordenados son: `[7]`. El adaptador del tiempo de ejecución extrae `7`. La transacción con el nonce `7` es inválida por el nonce impar.

Ninguna transacción válida se agrega para este grupo.

El grupo de la transacción se descarta, está vacío, por lo que el iterado de la pool lo descarta por completo:

```
pool: {
    transactions: {
      ("A", "c") -> [5, 2, 3]
    }
}
sorted_groups: [
  ("A", "a") -> [2, 3]
],
```

El siguiente grupo es para la llave `("A", "c")`, el iterador de la pool ordena las transacciones por sus nonces y regresa la referencia mutable para el grupo. Los nonces ordenados son: `[2, 3, 5]`. El adaptador del tiempo de ejecución extrae `2`.

Es una transacción válida, así que se agrega a la lista de transacciones válidas.

El grupo de la transacción se descarta, para que así el iterador de la pool lo descarte por completo:

```
pool: {
    transactions: { }
}
sorted_groups: [
  ("A", "a") -> [2, 3]
  ("A", "c") -> [3, 5]
],
```

##### Transacción #4

El siguiente grupo no se extrae de la pool, sino de sorted_groups. La llave es `("A", "a")`.
Ya está ordenado, por lo que el iterador regresa la referencia mutable. Los nonces son:
`[2, 3]`. El adaptador del tiempo de ejecución extrae `2`, después extrae `3`.

La transacción con nonce `2` es inválida, porque ya extraímos la transacción #1 de este grupo y tenía en nonce `2`.
El nonce nuevo tiene que ser más largo que el nonce anterior, así que está transacción queda invalidada.

La transacción con el nonce `3` es inválida porque el odd es impar.

Ninguna transacción válida es agregada para este grupo.

El grupo de la transacción se descarta, está vacío, por lo tanto el iterador de la pool lo descarta completamente:

```
pool: {
    transactions: { }
}
sorted_groups: [
  ("A", "c") -> [3, 5]
],
```

El siguiente grupo es para la llave `("A", "c")`, con los nonces `[3, 5]`.
El adaptador del teimpo de ejecución extrae `3`, después extrae `5`. Las dos transacciones son inválidas porque el nonce es impar.

Ninguna transacción fue agregada.

El grupo de la transacción se descarte, el envoltorio del iterador de la pool se queda vacío:

```
pool: {
    transactions: { }
}
sorted_groups: [ ],
```

Cuando el adaptador del tiempo de ejecución trata de extraer el grupo siguiente, el iterador de la pool regresa `None`, por lo que el adaptador del tiempo de ejecución descarta el iterador.

##### Descartando el iterador

Si el iterador no se vació por completo, pero aún quedan algunas transacciones. Se reinsertan de regreso a la pool.

##### Transacciones por fragmentos

Las transacciones que fueron extraídas de la pool:

```
// First batch
("A", "a", 1),
("A", "a", 1),
("A", "a", 2),
("B", "b", 13),
("B", "b", 14),
("C", "d", 7),
("A", "c", 2),

// Next batch
("A", "a", 2),
("A", "a", 3),
("A", "c", 3),
("A", "c", 5),
```

Las transacciones válidas son:

```
("A", "a", 2),
("B", "b", 14),
("A", "c", 2),
```

En total solo hubieron 3 transacciones válidas, que resultaron en un lote.

### Validación de orden

Otros validadores necesitan revisar el orden de las transacciones en el fragemto producido.
Puede hacerse en tiempo lineal, usando un algoritmo voraz.

Para seleccionar un primer lote necesitamos iterar sobre las transacciones una por una hasta que veamos una transacción
con la llave que ya incluímos en el primer lote.
Esta transacción pertenece al siguiente lote.

Ahora todas las transacciones en el lote N+1 deben tener una transacción correspondiente con la misma llave en el lote N.
Si no hay tranascciones con la misma llave en el lote N, entonces la orden es inválida.

También hacemos cumplir el orden de la secuencia de transacciones para la misma llave, sus nonces deben estar en orden estrictamente creciente.

Aquí está el algoritmo que valida el orden:

```rust
fn validate_order(txs: &Vec<Transaction>) -> bool {
    let mut nonces: HashMap<Key, Nonce> = HashMap::new();
    let mut batches: HashMap<Key, usize> = HashMap::new();
    let mut current_batch = 1;

    for tx in txs {
        let key = tx.key();

        // Verificando el nonce
        let nonce = tx.nonce();
        if let Some(last_nonce) = nonces.get(key) {
            if nonce <= last_nonce {
                // Nonce debe incrementar
                return false;
            }
        }
        nonces.insert(key, nonce);

        // Verificando el lote
        if let Some(last_batch) = batches.get(key) {
            if last_batch == current_batch {
                current_batch += 1;
            } else if last_batch < current_batch - 1 {
                // Se omitió esta clave en el lote anterior
                return false;
            }
        } else {
            if current_batch > 1 {
                // Not in first batch
                return false;
            }
        }
        batches.insert(key, batch);
    }
    true
}
```
