# wolfMKA Update: Interop Testing, an Automotive Profile, and Stronger Gates

When we first previewed **wolfMKA**, wolfSSL's implementation of the **MACsec
Key Agreement (MKA)** protocol (IEEE Std 802.1X, Clause 9), it ran the full
protocol: it discovered peers, elected a Key Server, distributed SAKs and
rekeyed make-before-break. It was built to MISRA C:2012 with 100% MC/DC
coverage. Since then we've made three kinds of change:

- run wolfMKA against independent MKA implementations over a real wire;
- add the OPEN Alliance automotive profile;
- tighten the gates every change has to pass.

Here's what's new.

## Interop against independent implementations

Unit tests show that wolfMKA agrees with itself. They can't show that it
agrees with anyone else. The interop harness runs wolfMKA and a peer in
separate Linux network namespaces joined by a veth pair, so every MKPDU crosses
a real Ethernet interface.

Testing against more than one implementation matters. Where the standard is
ambiguous, matching one reference implementation only shows that you share its
reading. Agreeing with several unrelated implementations is much stronger
evidence of conformance.

### Products we've tested against

**wpa_supplicant (2.11), from the hostap project.** This is the MKA
implementation most Linux systems ship, and the reference most people measure
against. We test it in two ways:

- *Control plane.* Peer discovery, liveness, Key Server election and SAK
  agreement, run once with wolfMKA as Key Server and once with wpa_supplicant as
  Key Server. That covers both halves of the protocol: wrapping and distributing
  a SAK, and unwrapping and installing a peer's SAK. Both GCM-AES-128 and
  GCM-AES-256 are tested.
- *Data plane.* Both ends program real kernel MACsec devices. A `ping` must flow
  protected, and a packet capture must show MACsec frames (EtherType 0x88E5)
  rather than cleartext. The Key Server then rekeys twice. Traffic must survive
  each handover, every SecY call must be accepted by the kernel, and old SAs
  must be retired instead of piling up.

**hostapd + wpa_supplicant, over EAP.** Most deployments use a pre-shared CAK,
but IEEE 802.1X also allows the CAK to be derived from an EAP exchange. hostapd
runs as the 802.1X authenticator with its built-in EAP server, and
wpa_supplicant authenticates against it with EAP-GPSK. wolfMKA derives a CAK and
CKN from the resulting MSK, and they must match wpa_supplicant's byte for byte.

**MKAdaemon, from Technica Engineering.** MKAdaemon is an independent MKA
implementation with an automotive lineage, and it speaks MKA Version 3 as the
OPEN Alliance profile requires. It is our only XPN-capable peer, because
wpa_supplicant 2.11 supports GCM-AES-128/256 only. The tier runs eight rounds:

- both Key Server roles, across **GCM-AES-128, GCM-AES-256, GCM-AES-XPN-128 and
  GCM-AES-XPN-256**, with protected traffic over kernel MACsec devices;
- a check that **both ends are actually encrypting**, not just
  integrity-protecting;
- Key Server-driven rekeys on the XPN rounds, where the Salt changes with each
  new key;
- MKAdaemon restarting: wolfMKA must expire it and agree a fresh SAK with the
  returning participant;
- wolfMKA restarting and rejoining a CA that is already running;
- a negative round with the same CKN but a different CAK, which must associate
  nothing.

**libmka, from Intrepid Control Systems.** libmka is a third unrelated
implementation, built on mbedTLS. It hands its SAK to a callback rather than
driving a MACsec device, so this tier tests the control plane only, in both Key
Server roles. Two things make the result stronger than it looks:

- libmka *accepting* a frame proves both ends derived the same ICK.
- libmka *installing* the AES-wrapped SAK proves the KEK matched.

The round then compares the Key Identifier each side reports.

**The Linux kernel `macsec` driver.** The kernel's MACsec driver is the data
plane in the wpa_supplicant and MKAdaemon data-plane tiers. wolfMKA programs it
over netlink through the SecY binding in `examples/mka_macsec`. That binding
now supports XPN: 64-bit packet numbers, per-channel Short SCIs and the
12-octet Salt.

### Summary

| Product | Vendor / project | Plane | Cipher suites | Key Server roles | In CI |
|---|---|---|---|---|---|
| wpa_supplicant 2.11 | hostap project | Control + data | GCM-AES-128, -256 | Both | Yes |
| hostapd + wpa_supplicant (EAP-GPSK) | hostap project | Key derivation | n/a | n/a | Yes |
| MKAdaemon | Technica Engineering | Control + data | GCM-AES-128, -256, XPN-128, XPN-256 | Both | No (built from source) |
| libmka | Intrepid Control Systems | Control | GCM-AES-128 | Both | No (built from source) |
| Linux `macsec` | Linux kernel | Data | All of the above | n/a | Partly |

The control-plane and EAP tiers run in CI with `INTEROP_STRICT=1`. If a runner
can't provide the prerequisites, the job fails instead of quietly reporting
`SKIP`. MKAdaemon and libmka aren't packaged, so those tiers need a source
build and run outside CI. The build recipes are documented in the repository.

Every tier runs a two-member CA. Behaviour that only appears with three or more
members, such as the full XPN SSCI mapping, is covered by the unit suite but
hasn't yet been tested against another implementation.

## Newer revisions of the standard

- **MKA Version 3 support.** `WOLFMKA_MKA_VERSION` selects the MKA version
  announced in the Basic Parameter Set: 1 (802.1X-2010, the default), 2
  (802.1Xbx-2014) or 3 (802.1X-2020). Version 3 builds meet that version's
  obligations: an ordered Live Peer List, and the Key Server's SSCI in the
  MKPDUs that distribute an XPN SAK.
- **XPN SSCI assignment per 802.1X-2020 §9.10.** A member takes its Short SCI
  from the mapping the Key Server publishes with the SAK, rather than computing
  it from its own view of the membership. The two views can differ when an
  MKPDU is lost or delayed. When a member joins while an XPN key is in force,
  the Key Server issues a fresh SAK so every member's SSCI comes from the same
  membership list.

## OPEN Alliance TC17 automotive profile

`WOLFMKA_AUTOMOTIVE_PROFILE` (opt-in) implements the OPEN Alliance TC17
Automotive MKA profile. It sends Version 3 MKPDUs, removes Announcements and
cipher-suite negotiation, and `wm_Participant_AutomotiveDefaults()` supplies the
profile's startup timing. Along with it come:

- **Fast response to new information** (`immediate_response`): MACsec comes up
  about 3 ms into the in-process exchange instead of waiting for the next Hello
  Time.
- **Distribute before install**: a peer behind a slow data plane starts
  installing at the same time as the Key Server.
- **Rekey on lost SAK**, with a configurable `sak_loss_threshold` that counts
  *consecutive* misses. The default is 1 under the profile, for the fastest
  recovery, and 3 elsewhere.
- **A Message Number recency window for liveness**: a peer must echo a Message
  Number this participant actually sent recently.
- **Peer-reset recovery**: a device that returns with a fresh Member Identifier
  is admitted immediately. Its stale entry is retired only once the new one
  proves liveness, so a forged or replayed MI can't evict a working peer.
- **Key Server priority extremes**: 0 means always Key Server and 255 means
  never.
- **MACsec statistics and PAE status** accessors, plus **CA quiescing** across
  the participants on a port.

[`OPEN-ALLIANCE.md`](wolfMKA/OPEN-ALLIANCE.md) maps the profile requirement by
requirement, including the gaps that remain.

Outside the profile, `MkaEventOps.key_server_lost` adds the missing
counterpart to `key_server_elected`. An integrator gating a port or a status
indicator on the election can now see it lost as well as made.

## AUTOSAR binding

- **Det development-error reporting** (`MKA_DEV_ERROR_DETECT`). It is off by
  default, so the binding links without a Det. When it's on, every rejection is
  reported with its exact service ID and error ID. Errors are detected and
  rejected the same way whether reporting is on or off.
- **The same gates as the core.** `src/autosar/Mka.c` is now held to **100%
  line, decision and MC/DC coverage** (`make coverage-autosar`, run with Det off
  and on) and is included in the **MISRA C:2012** scan.

## Build and verification

- **The RNG workspace is mandatory**, like the CMAC workspace. Every
  participant draws from a fixed, caller-owned pool, so no DRBG state lands on
  the stack or heap in the `WOLFMKA_SMALL_STACK` / `WOLFMKA_NO_MALLOC` builds.
- **`make check-abi`** verifies that wolfMKA and the installed wolfSSL agree on
  the sizes of the shared crypto structures. If they don't, it fails the build
  and names the fix. `WC_RNG`'s size depends on the C standard the library was
  compiled under, so this mismatch would otherwise go unnoticed.
- **`make docs` is a real documentation gate.** It fails the build if any public
  type, field or enumerator is missing documentation.
- **1,408 unit checks** (up from 1,106).
- **100% line, decision and MC/DC coverage**, enforced across **eleven** build
  configurations plus the AUTOSAR binding. Every `.c` file in `src/` is now
  gated.

## Get in touch

wolfMKA targets automotive ECUs, industrial Ethernet and network equipment. Its
data plane plugs into a Linux `macsec` device, a hardware offload, or your own
SecY. If MACsec is on your roadmap, we'd like to hear from you and set you up
for early access.

Contact us at **facts@wolfssl.com** or call **+1 425 245 8247**.

Love it? Star us on [GitHub](https://github.com/wolfSSL/wolfssl)!
