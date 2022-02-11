# Pruebas Merkle

Muchos componentes del Protocolo NEAR dependen de la raíz Merkle y las pruebas Merkle. Para un arreglo de hashes sha256, definimos su
raíz merkle como:
```python
CRYPTOHASH_DEFAULT = [0] * 32
def combine_hash(hash1, hash2):
    return sha256(hash1 + hash2)

def merkle_root(hashes):
    if len(hashes) == 0:
        return CRYPTOHASH_DEFAULT
    elif len(hashes) == 1:
        return hashes[0]
    else:
        l = hashes.len();
        subtree_len = l.next_power_of_two() // 2;
        left_root = merkle_root(hashes[0:subtree_len])
        right_root = merkle_root(hashes[subtree_len:l])
        return combine_hash(left_root, right_root)
```

Generalmente, para un arreglo de objetos serializables borsh, su root merkle es definida como
```python
def arr_merkle_root(arr):
    return merkle_root(list(map(lambda x: sha256(borsh(x)), arr)))
```

Una prueba merkle es definida por:
```rust
pub struct MerklePathItem {
    pub hash: MerkleHash,
    pub direction: Direction,
}

pub enum Direction {
    Left,
    Right,
}

pub type MerkleProof = Vec<MerklePathItem>;
```

La verificación de un hash `h` contra una raíz merkle proclamada `r` con prueba `p` se define por:
```python
def compute_root(h, p):
    res = h
    for item in p:
        if item.direction is Left:
            res = combine_hash(item.hash, res)
        else:
            res = combine_hash(res, item.hash)
    return res

assert compute_root(h, p) == r
```
