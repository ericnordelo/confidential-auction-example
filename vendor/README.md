# Asset dependencies

The auction uses the experimental Token Standard V2 registry from
[OpenZeppelin/canton-contracts at `7696749`](https://github.com/OpenZeppelin/canton-contracts/tree/7696749737885e25cd88422847105f890f03b00d/experiments/token/tokenCIP112-v1).
Its source and package configuration are copied unchanged under
`experiments/token/tokenCIP112-v1/`. Its MIT license is in [LICENSE](LICENSE).

`dars/vendor/` contains the 13 Token Standard interface and utility DARs used by
that registry. [The manifest](dars/manifest.yaml) records their versions, package
IDs, SHA-256 checksums, source revisions, build adjustments and licenses. Its
`file` paths are relative to this `vendor/` directory.

The DARs come from Splice commit
[`69b43eb`](https://github.com/hyperledger-labs/splice/tree/69b43eb761e38695052c983715aa855c8cb207fc/token-standard).
They are the devnet-stage V2 dependencies used by this example, not a claim of
compatibility with every Token Standard registry. The build targets Daml-LF 2.1.
