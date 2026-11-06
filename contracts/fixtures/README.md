# Fixtures: 20 example blocks

Generated chain of 20 blocks (heights 0 to 19, chain id 1) in the JSON view of
[node-api.md](../node-api.md). `blocks/block-NN.json` holds one block with its
`hash`. `chain.json` holds all blocks and the keys used (seeds are test-only).

**These are proof-of-work blocks**, the format the node produces today: a header
with `parent`, `target`, `nonce` and `system_id`, and mock proofs. Blocks of the
PoUW design in [block.md](../block.md) (`difficulty`, buckets) will look somewhat
different; that format is still being researched, so there are no PoUW fixtures yet.

- Block 1 registers a circuit, most blocks hold a transfer, every third block
  holds a proof request with a mock proof.
- Built with the node code (`crates/pouw-api/fixtures.rs` in pouw-node), so ids,
  Merkle roots, eta, mask and signatures are the real ones.
- Not a chain a node can sync: the target is constant and the senders have no
  genesis balance. Use it to mock the API and to test parsers.

Regenerate:

    POUW_FIXTURES_DIR=<this folder> cargo test -p pouw-api write_fixtures -- --ignored
