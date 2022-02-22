# API Math

```rust
random_seed(register_id: u64)
```

Regresa una semilla aleatoria que puede ser usada para la generación de número pseudo-aleatorio de una manera determinista.

###### Pánicos

- Si el tamaño del registro excede el límite establecido `MemoryAccessViolation`;

---

```rust
sha256(value_len: u64, value_ptr: u64, register_id: u64)
```

Hashea la secuencia de bytes aleatoria usando sha256 y la regresa a `register_id`.

###### Pánicos

- Si `value_len + value_ptr` apunta por fuera de la memoria o los registros usan más memoria que el límite, con `MemoryAccessViolation`.

---

```rust
keccak256(value_len: u64, value_ptr: u64, register_id: u64)
```

Hashea la secuencia aleatoria de bytes usando keccak256 y la regresa a `register_id`.

###### Pánicos

- Si `value_len + value_ptr` apunta por fuera de la memoria o los registros usan más memoria que el límite, con `MemoryAccessViolation`.

---

```rust
keccak512(value_len: u64, value_ptr: u64, register_id: u64)
```

Hashea la secuencia aleatoria de bytes usando keccak512 y los regresa a `register_id`.

###### Panics

- Si `value_len + value_ptr` apunta por fuera de la memoria o los registros usan más memoria que el límite, con `MemoryAccessViolation`.
