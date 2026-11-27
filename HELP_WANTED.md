# Help wanted

attest is open to contributions. Below is the hardening work that would make it stronger, roughly in order of impact. Pick anything, comment on (or open) an issue so others know you are on it, and see [CONTRIBUTING.md](CONTRIBUTING.md) to get started.

Found a vulnerability instead? Please report it privately, see [SECURITY.md](SECURITY.md).


### Reference verifier

**High impact.** A small verifier that a website could run to check a proof of containment, with test vectors for valid, expired and forged proofs.

### Threat model review

**High impact.** Try to break the protocol: replay, supervisor key theft, an escaped agent proxying through a contained one. Open an issue for each attack.
