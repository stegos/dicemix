# Dicemix Light

## Primitives

#### Pseudorandom Generator
 * `new_prg(seed)` initializes a PRG with seed `seed` and tweak `tweak`.
 * randomness can be obtained using calls such as `prg.get_bytes(n)` or `prg.get_field_element()`.

#### Deterministic One-time Signatures
We need a key-recoverable signature scheme which is weakly unforgeable under one-time chosen
message attacks.
 * `(otsk, otvk) := new_sig_keypair(rand)` generates a signing key and verification key.
 * `sign(otsk, msg)` creates a deterministic signature of message `msg` with signing key `otsk`.
 * `verify_recover(sig, msg)` outputs the verification key if `sig` is a valid signature on
 `msg` and nothing otherwise.

#### Non-interactive Key Exchange
We need a non-interactive key exchange protocol secure in the CRS model.
 * `(kesk, kepk) := new_ke_keypair(rand)` generates a secret key and a public key.
 We assume that for every valid public key there is a unique corresponding secret key.
 (Note: For ECDH, this implies that setting `kepk` to be just the x-coordinate without an
 additional bit is not sufficient; `kepk` must determine the full curve point.)
 * `validate_kepk(kepk)` outputs `true` iff `kepk` is a valid public key.
 * `shared_secret(kesk, kepk, my_id, their_id, tweak)` derives the shared secret between party
 `my_id` with secret key `kesk` and `their_id` with public key `kepk`, using the tweak `tweak`

#### Hash Function
 * `hash` is a cryptographic hash function (modeled as a random oracle).

## Protocol

### Pseudocode Conventions
 * The (non-excluded) peers are stored in set `P`.
 * `sgn(x)` is the signum function.
 * `**` denotes exponentiation.
 * `^` denotes bitwise XOR.
 * `(o)` denotes the arithmetic operator `o` in the finite field, e.g., `(+)` is addition in
 the finite field.
 * String constants such as `"KE"` are symbolic, their actual representation as bytes is
 defined below.

### Setup Assumptions
TODO: Write

### Authentication
All protocol messages are assumed to be authenticated. Unauthenticated messages must be ignored.
We note that authentication is only required for termination but not for anonymity.

### Pseudocode
TODO: Extend to multiple messages per peer
TODO: Parallel runs
```
run := -1
P_exclude := {}
loop
    run := run + 1

    // If this is not the first run, exclude offline or malicious peers
    if run > 0 then
        if P_exclude != {} then
            P := P \ P_exclude
        else
            // Publish ephemeral secret and determine malicious peers
            broadcast "KESK" || my_kesk
            receive "KESK" || p.kesk from all p in P
            missing P_missing

            P := P \ P_missing

            for all p in P do
                replay all protocol messages of p using p.kesk
                if p has sent an incorrect message then
                    P := P \ {p}

            if there is p in P with p.kepk = my_kepk then
                fail "No honest peers left."

            for all (p1, p2) in P^2 with p1 != p2 and p1.kepk = p2.kepk do
                P := P \ {p1, p2}

    if |P| = 0 then
        fail "No peers left."
        break

    P_exclude := {}

    // Build session ID
    ids[] := sort({p.id | p in P} U {my_id})

    if there are duplicate value in ids[] then
        fail "Duplicate peer IDs."

    sid := version || options || nonce || run || ids[0] || ... || ids[|P|]
    sid_hash := hash("SID" || sid)
    // FIXME more SIDs later?

    // Key exchange
    (my_kesk, my_kepk) := new_sig_keypair()

    // FIXME sign the kepk with the long-term key
    broadcast "KE" || my_kepk
    receive "KE" || p.kepk from all p in P
        where validate_kepk(p.kepk)
        missing P_missing

    P := P \ P_missing

    // Derive shared keys
    for all p in P do
        p.seed_dcexp := shared_secret(my_kesk, p.kepk, my_id, p.id, sid_hash || "DCEXP")
        p.prg_dcexp := new_prg(seed_dcexp)
        p.seed_dcsimple := shared_secret(my_kesk, p.kepk, my_id, p.id, sid_hash || "DCSIMPLE")
        p.prg_dcsimple := new_prg(seed_dcsimple)

    // Generate signature key pair
    otsk_seed := hash("OTSK_SEED" || my_kesk)
    (otsk, my_otvk) := new_ke_keypair(otsk_seed)

    // Run a DC-net with exponential encoding
    my_dc[] := array of |P| finite field elements
    otvk_hash = hash("OTVK" || my_otvk)
    for i := 0 to |P| do
        my_dc[i] := otvk_hash ** (i + 1)

    for all p in P do
        for i := 0 to |P| do
            my_dc[i] := my_dc[i] (+) p.prg_dcexp.get_field_element()

    broadcast "DCEXP" || my_dc[0] || ... || my_dc[|P|]
    receive "DCEXP" || p.dc[0] || ... || p.dc[|P|] from all p in P
        missing P_exclude

    if P_exclude != {} then
        continue

    dc_combined[] := my_dc[]
    for p in P do
        for i := 0 to |P| do
            dc_combined[i] := dc_combined[i] (+) (sgn(my_id - p.id) (*) p.dc[i])

    solve the equation system
        "for all 0 <= i <= |P|,
         dc_combined[i] = (sum)(j := 0 to |P|, roots[j] ** (i + 1))"
         for the array roots[]

    otvk_hashes[] := sort(roots[])

    slot := undef
    if there is exactly one i with otvk_hashes[i] = otvk_hash then  // implement in constant time
        slot := i

    // Run a ordinary DC-net with slot reservations
    msg := fresh_msg()
    sig := sign(otsk, msg)

    slot_size := |sig| + |msg|

    my_dc[] := array of |P| arrays of slot_size bytes, all initalized with 0
    if slot != undef then
        my_dc[slot] := sig || msg  // implement in constant time

    for all p in P do
        for i := 0 to |P| do
            my_dc[i] := my_dc[i] ^ p.prg_dcsimple.get_bytes(slot_size)
        if padding_size > 0 then
            my_padding := my_padding ^ p.prg_dcsimple.get_bytes(slot_size)

    broadcast "DCSIMPLE" || my_dc[0] || ... || my_dc[|P|] || my_padding
    receive "DCSIMPLE" || p.dc[0] || ... || p.dc[|P|] || my_padding from all p in P
        missing P_exclude

    if P_exclude != {} then
        continue

    dc_combined[] := my_dc[]
    for p in P do
        for i := 0 to |P| do
            dc_combined[i] := dc_combined[i] ^ p.dc[i]

    // Check signatures
    msgs[] := array of |P| messages

    found := false
    for i := 0 to |P| do
        sigi || msgs[i] := dc_combined[i]
        otvki := verify_recover(sigi, msg[i])
        if not otvki then
            continue
        if hash("OTVK", otvki) != otvk_hashes[i] then
            continue

    sort(msgs[])

    if msg is not a value in msgs[] then  // implement in constant time
        fail "This is probably a bug."

    // Confirmation
    my_confirmation := validate_and_confirm(msgs[])
    if my_confirmation != undef then
        continue

    broadcast "CF" || my_confirmation
    receive "CF" || p.confirmation from all p in P
        missing P_exclude

    if P_exclude != {} then
        continue

    // Check confirmation
    P_exclude := check_confirmations(msgs[], {(p.id, p.confirmation) | p in P})

    if P_exclude != {} then
        continue

    return successfully
```