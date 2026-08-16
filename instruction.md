The kvdex library (`@olli/kvdex`) exposes a rich set of "by secondary index" query helpers on its indexable collections. The "by secondary order" side of the API in `src/collection.ts` is inconsistent with the rest of the library: it has `findBySecondaryOrder`, `deleteBySecondaryOrder`, and `updateBySecondaryOrder`, but these don't follow the `many`/`one` naming used everywhere else and lack `getMany`/`deleteMany`/`updateMany` variants. Separately, the map-backed KV extension currently resides under `src/ext/map_kv/`; a cleaner `src/ext/kv/` module layout houses this extension instead, with imports and the public documentation reflecting the new location.

Concretely, align the many/one secondary-order methods on the collection so a caller can operate over documents sorted by a secondary index rather than by id. Rename the existing helpers to the `many` variants: `findBySecondaryOrder` becomes `getManyBySecondaryOrder(order, options?)`, retrieving multiple documents in the specified secondary order (no options means "all documents", with support for a `filter` predicate over each `doc`); `deleteBySecondaryOrder` becomes `deleteManyBySecondaryOrder(order, options?)`, deleting documents in that order (honoring optional filtering and pagination via `limit`, deleting all when no options are given); and `updateBySecondaryOrder` becomes `updateManyBySecondaryOrder(order, data, options?)`, updating documents in that order (e.g. update the first 10 users ordered by `age`, setting `username = "anon"`). The `one`/iteration helpers `getOneBySecondaryOrder(order, options?)`, `updateOneBySecondaryOrder(order, data, options?)`, `forEachBySecondaryOrder`, `mapBySecondaryOrder`, and `countBySecondaryOrder` already exist and should keep the same ordering semantics as their `BySecondaryIndex` counterparts. All of these must interoperate with both the plain indexable collection and the serialized indexable collection.

**CRITICAL CONSTRAINTS FOR SECONDARY ORDER METHODS:**
- All secondary order methods must respect the ordering direction semantics: ascending vs descending order must be consistently applied across `getManyBySecondaryOrder`, `deleteManyBySecondaryOrder`, and `updateManyBySecondaryOrder`
- Pagination with `limit` and `offset` must work correctly when applied AFTER filtering, not before; filters reduce the result set before pagination is applied
- Filtering with `filter` predicate must process documents in the secondary index order, not randomly
- Methods must properly handle concurrent mutations: if documents are deleted/updated during iteration, the cursor state must remain valid
- For `updateManyBySecondaryOrder`, complex nested updates (using partial types and type preservation) must maintain document structure integrity
- Edge cases: empty collections, single-document collections, operations on filtered-to-empty result sets, cursor resumption after batch operations, and operations with identical secondary index values must all be handled correctly
- All operations must maintain atomic consistency within transaction boundaries
- Secondary index key types (strings, numbers, dates, etc.) must sort correctly according to their natural ordering
- Methods must prevent index corruption when handling rapid concurrent operations across multiple secondary indexes on the same collection
- History tracking (if `keepsHistory` is enabled) must correctly record secondary order operations with their ordering context

The map-backed KV extension lives under `src/ext/kv/` rather than `src/ext/map_kv/`. In the new layout, the module previously at `src/ext/map_kv/kv.ts` corresponds to `src/ext/kv/map_kv.ts`, and `atomic.ts`, `utils.ts`, and `watcher.ts` reside under `src/ext/kv/`. The module additionally includes `src/ext/kv/mod.ts`, `src/ext/kv/storage_adapter.ts`, and `src/ext/kv/types.ts`, while `src/ext/map_kv/storage_adapter.ts` no longer exists. `MapKv` is exported from `src/ext/kv/map_kv.ts`, and `StorageAdapter` is exported from `src/ext/kv/mod.ts`.

While relocating the extension, tighten the backing-map contract. `StorageAdapter` gains a `clear()` method that empties the underlying store, and its `get()` returns `undefined` (rather than `null`) when a key is absent. `MapKv`'s constructor takes a single options object `{ map, entries, clearOnClose }`, all optional: `map` defaults to a `new Map()` backend, `entries` seeds initial KV entries, and `clearOnClose` (default `false`) controls whether the backing map is cleared when the store is closed.

**CRITICAL CONSTRAINTS FOR KV EXTENSION:**
- `StorageAdapter.clear()` must be idempotent (can be called multiple times safely)
- `StorageAdapter.get()` must return `undefined` (not `null`, not empty strings) for missing keys
- `MapKv` with `clearOnClose=true` must clear the backing map ONLY after all pending operations complete
- When `entries` are provided during construction, they must be applied BEFORE any user operations
- `MapKv` must support concurrent read and write operations without deadlocks
- Transaction operations across the KV store must maintain ACID properties even with the Map backend
- The backing map must support all KvValue types correctly (Uint8Array, strings, objects, etc.)
- Memory usage must not grow unbounded with `clearOnClose=false` (implement proper cleanup for intermediate data)
- Watchers must continue working correctly after `clear()` is called
- `MapKv` must handle rapid open/close cycles without resource leaks

Everything continues to work offline under Deno with the existing test task (`deno test --allow-write --allow-read --allow-ffi --allow-sys --unstable-kv --trace-leaks`). `README.md` documents the new secondary-order methods and includes a `KV` section, and `deno.json` reflects any necessary changes. The relocated tests under `tests/ext/kv.test.ts` import `MapKv` from `../../src/ext/kv/map_kv.ts` and `StorageAdapter` from `../../src/ext/kv/mod.ts`, so those paths must resolve.