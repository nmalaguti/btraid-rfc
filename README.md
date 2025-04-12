# btraid: A Safe, Flexible Full-Stripe Parity Model for Btrfs

This RFC proposes a new design for reliable, full-stripe parity in Btrfs, addressing the long-standing issues with its RAID 5/6 implementation. The design introduces **btraid**, a general, stripe-based redundancy framework that supports pluggable parity algorithms, dynamic pool expansion, and full compatibility with Btrfs's core features like CoW, compression, and snapshots.

## Motivation

Btrfs's current RAID 5/6 implementation is incomplete and unsafe. It lacks full-stripe write protection, parity journaling, and robust recovery mechanisms. This design aims to replace that implementation with a clean, extensible model that aligns with the architecture and goals of Btrfs.

## Design Goals

- **Atomic full-stripe writes** with no write hole
- **Per-stripe redundancy algorithms** (`mirror`, `xor`, `rs`, etc.)
- **Minimal metadata overhead**
- **Dynamic expansion and contraction** of the pool
- **Fine-grained control** over redundancy per file or subvolume
- **Full compatibility** with snapshots, compression, and reflinks
- **Read-path optimizations** using per-block checksums

## The btraid Model

### N/M Redundancy
Each stripe is described by a ratio `N/M`:
- `M`: total number of logical blocks in the stripe
- `N`: number of redundancy blocks
- Each parity block declares its algorithm (e.g., `mirror`, `xor`, `rs`)

Examples:
- `0/4`: RAID 0 (striping only)
- `1/2`: RAID 1 (mirror)
- `1/4`: RAID 5 (single-parity)
- `2/6`: RAID 6 (dual-parity)

### Stripe Metadata
- Stored per-stripe alongside extent metadata
- Records:
  - Block roles (data or parity)
  - Algorithm used for each parity block
  - Logical to physical mapping

### Write Path
- Data blocks are written via CoW as in standard Btrfs
- Parity is computed over uncompressed logical blocks
- Each logical block is compressed individually
- All blocks (data + parity) are committed together atomically

### Read Path
- Normal reads use per-block checksums to validate individual 4K blocks
- Parity is only used if a block is missing or corrupt
- Full-stripe reads occur during scrub or repair

## Benefits

- Works with a single disk (`0/1`) and grows over time
- Rebalancing acts as live data migration
- No need for journaling or ARC-like caches
- RAID modes become special cases of a generalized redundancy engine
- Can start safe with `mirror`, evolve to `xor`, `rs`, etc.

## Implementation Path

1. `btraid1` with `mirror` only (drop-in replacement for RAID1)
2. Add `xor` (RAID 5 equivalent)
3. Add `rs` (RAID 6 equivalent)
4. Stripe-aware rebalance engine
5. CLI support and status tools

## Risks and Challenges

- Needs allocator and rebalance changes
- Stripe metadata must be stable and correct
- Recovery tooling and test surface must be comprehensive
- Requires long-term maintainer commitment

## Why Not Just Fix RAID 5/6?

The current implementation is inherently unsound: it lacks full-stripe semantics and can't be safely repaired incrementally. This model replaces it with something safe, composable, and testable — while aligning with Btrfs's strengths: CoW, extent-based allocation, and rich metadata.

## License

This work is licensed under Creative Commons Attribution 4.0 International 

This RFC and design are provided under the [Creative Commons Attribution 4.0](https://creativecommons.org/licenses/by/4.0/) license.

---

_Repository: https://github.com/nmalaguti/btraid-rfc_

