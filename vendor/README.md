# Asset dependencies

The auction uses Splice's [TestTokenV2 implementation](https://github.com/canton-network/splice/tree/69b43eb761e38695052c983715aa855c8cb207fc/token-standard/examples/splice-test-token-v2)
at commit `69b43eb761e38695052c983715aa855c8cb207fc`.
Its six Daml source files are unchanged under `splice-test-token-v2/`.
[SOURCE.json](splice-test-token-v2/SOURCE.json) records their upstream Git blob
hashes and build adjustments. The Apache-2.0 license is in
[splice-test-token-v2/LICENSE](splice-test-token-v2/LICENSE).

The local `daml.yaml` uses this workspace's SDK 3.4.11, retains Daml-LF 2.1,
and points to the existing pinned Token Standard DARs. Upstream uses SDK 3.5.2.

`dars/vendor/` contains the 13 interface and utility DARs from the same Splice
commit. [The manifest](dars/manifest.yaml) records their versions, package IDs,
checksums, build adjustments and licenses. Its paths are relative to `vendor/`.

TestTokenV2 provides funded and unfunded iterated allocations, account-provider
approval instructions, settlement, withdrawal and event logging. It is an
upstream test implementation. These scripts validate the auction against this
pin, rather than claiming compatibility with every registry or a Canton Coin
deployment. The registry source has no OpenZeppelin D1/D2 extensions.
