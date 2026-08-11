# Identity

A user's identity is a set of keys only they control. The vault holds
custody, answers capability requests, and owns consent, rotation and
recovery. It is not an account and no server issues it.

**Contract:** [`identity/UI-Identity.md`](https://github.com/onymchat/onym-system/blob/main/identity/UI-Identity.md)
· profile: [BIP-39](https://github.com/onymchat/onym-system/blob/main/identity/UI-Identity-BIP39.md)
**Code:** [`onym-sdk-swift`](https://github.com/onymchat/onym-sdk-swift),
[`onym-sdk-kotlin`](https://github.com/onymchat/onym-sdk-kotlin), consumed by
[`onym-ios`](https://github.com/onymchat/onym-ios) /
[`onym-android`](https://github.com/onymchat/onym-android)

There is no identity server. This seat is a device-side library plus the
client's `IdentityRepository` and recovery-phrase flow.

## The SDKs

Both wrap the same per-type PLONK FFI staticlibs built from
`onym-contracts/plonk/sep-*-ffi`, so proofs and hashes are byte-identical
across platforms.

| Namespace | Functions |
|---|---|
| `Common` | `leafHash`, `publicKey`, `merkleRoot`, `sha256Commitment`, `poseidonCommitment`, `parsePlonkProof`, `nostrDerivePublicKey`, `nostrSignEventId`, `nostrVerifyEventSignature` |
| `Anarchy` | `bakeMembershipVK`, `bakeUpdateVK`, `pinnedMembershipVKSha256Hex`, `pinnedUpdateVKSha256Hex`, `proveMembership`, `proveUpdate` |
| `OneOnOne` | `bakeCreateVK`, `proveCreate` |
| `Tyranny` | `bakeCreateVK`, `bakeUpdateVK`, `pinnedCreateVKSha256Hex`, `pinnedUpdateVKSha256Hex`, `proveCreate`, `proveUpdate` |

Swift: 25 functions over a stable C ABI (`COnymFFI`), `throws -> Data`.
Kotlin: 23 JNI entry points (`chat.onym.sdk.internal.OnymJni`), throwing
`OnymException`. LTO strips the Kotlin `.so` to ~340 KB from ~110 MB of
input staticlibs.

The `pinned*VKSha256Hex` values anchor the client to the same verifying
keys the on-chain verifier was built from — a prover-shape drift fails
here and in `onym-contracts` CI simultaneously.

## Consuming

**Swift** — add the `onym-sdk-swift` package; `import OnymSDK`.

**Kotlin** — the AAR is served from the static Maven repo on the
[`releases`](https://github.com/onymchat/onym-sdk-kotlin/tree/releases)
branch, wired in `settings.gradle.kts`. No `publishToMavenLocal` step.

```sh
# onym-ios
brew install xcodegen && ./generate-xcodeproj.sh && open OnymIOS.xcodeproj

# onym-android — JDK 17, min SDK 26, target 35
./gradlew :app:assembleDebug
```

`project.yml` is the source of truth for the Xcode project; `*.xcodeproj/`
is gitignored, so re-run the generator after pulling.

## Status

The clients are being grown in small hand-reviewable chunks. Today that
means the persistent reactive `IdentityRepository` and the recovery-phrase
backup flow exist; chat, invites and rotation do not. Identity rotation and
recovery are named in the whitepaper as open work — treat the vault as
create-and-back-up only.
