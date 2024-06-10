# FROST signing for BIP340-compatible Threshold Signatures (BIP draft)

### Abstract

This document proposes a standard for the Flexible Round-Optimized Schnorr Threshold (FROST) signing protocol. The standard is compatible with [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) public keys and signatures. It supports _tweaking_, which allows deriving [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) child keys from the group public key and creating [BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) Taproot outputs with key and script paths.

### Copyright

This document is licensed under the 3-clause BSD license.

## Introduction

This document proposes the FROST signing protocol based on the FROST3 variant (see section 2.3) introduced in ROAST[[RRJSS22](https://eprint.iacr.org/2022/550)], instead of the original FROST[[KG20](https://eprint.iacr.org/2020/852)]. Key generation for FROST signing is out of scope for this document. However, we specify the requirements that a key generation method must satisfy to be compatible with this signing protocol.

Many sections of this document have been directly copied or modified from [BIP327](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki) due to the similarities between the FROST3 and [MuSig2](https://eprint.iacr.org/2020/1261.pdf) signature schemes.

### Motivation

The FROST signature scheme [[KG20](https://eprint.iacr.org/2020/852),[CKM21](https://eprint.iacr.org/2021/1375),[BTZ21](https://eprint.iacr.org/2022/833),[CGRS23](https://eprint.iacr.org/2023/899)] enables _t-of-n_ Schnorr threshold signatures,in which a threshold _t_ of some set of _n_ signers is required to produce a signature.
FROST remains unforgeable as long as at most _t-1_ signers are compromised, and remains functional as long as _t_ honest signers do not lose their secret key material. It supports any choice of _t_ as long as _1 ≤ t ≤ n_[^t-edge-cases].

The primary motivation is to create a standard that allows users of different software projects to jointly control Taproot outputs ([BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)).
Such an output contains a public key which, in this case, would be the group public key derived from the public shares of threshold signers.
It can be spent using FROST to produce a signature for the key-based spending path.

The on-chain footprint of a FROST Taproot output is essentially a single BIP340 public key, and a transaction spending the output only requires a single signature cooperatively produced by _threshold_ signers. This is **more compact** and has **lower verification cost** than signers providing _n_ individual public keys and _t_ signatures, as would be required by an _t-of-n_ policy implemented using <code>OP_CHECKSIGADD</code> as introduced in ([BIP342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)).
As a side effect, the numbers _t_ and _n_ of signers are not limited by any consensus rules when using FROST.

Moreover, FROST offers a **higher level of privacy** than <code>OP_CHECKSIGADD</code>: FROST Taproot outputs are indistinguishable for a blockchain observer from regular, single-signer Taproot outputs even though they are actually controlled by multiple signers. By tweaking a group public key, the shared Taproot output can have script spending paths that are hidden unless used.

There are threshold-signature schemes other than FROST that are fully compatible with Schnorr signatures.
The FROST variant proposed below stands out by combining all the following features:
* **Two Communication Rounds**: FROST is faster in practice than other threshold-signature schemes [[GJKR03](https://link.springer.com/chapter/10.1007/3-540-36563-x_26)] which requires at least three rounds, particularly when signers are connected through high-latency anonymous links. Moreover, the need for fewer communication rounds simplifies the algorithms and reduces the probability that implementations and users make security-relevant mistakes.
* **Efficiency over Robustness**: FROST trades off the robustness property for network efficiency (fewer rounds), requiring the protocol to be aborted in the case of any misbehaving participant.
* **Provable security**: FROST3 with an idealized key generation (i.e., trusted setup) has been [proven existentially unforgeable](https://eprint.iacr.org/2022/550.pdf) under the one-more discrete logarithm (OMDL) assumption (instead of the discrete logarithm assumption required for single-signer Schnorr signatures).

### Design

## Overview

### Optionality of Features

### General Signing Flow
TODO: can we add that the excalidraw image here?

### Key Generation

### Nonce Generation

### Identifying Disruptive Signers

#### Further Remarks

### Tweaking the Group Public Key

- [ ] subsections
	- [ ] optionality of features
	- [ ] key generation compatibility
		TODO give a link to "frost keys" in "algorithms"
	- [ ] general signing flow
		TODO mention various keygen protocols
	- [ ] nonce generation
	- [ ] identifying disruptive signers
	- [ ] tweaking the aggregate public key

## Key Generation

This document does not provide information on the key generation method necessary for FROST signing. To learn about such methods, you can refer to [RFC-frost (Appendix D)](https://datatracker.ietf.org/doc/draft-irtf-cfrg-frost/) and [BIP-DKG](https://github.com/BlockstreamResearch/bip-frost-dkg).

It is important to note that any FROST keys generated using the aforementioned methods must meet the correctness conditions (refer to <link subsection>) to be compatible with the signing protocol. However, it is essential to understand that the correctness conditions do not guarantee the security of these key generation methods. The conditions only ensure that the keys generated are functionally compatible with the signing protocol.

TODO will duplicating the necessary notations here improve readability? rather than mentioning its link

### Key Specification

FROST signatures function as if they were created by an individual signer using a signing key, thus enabling them to be verified with the corresponding public key. In order to achieve this functionality, a key generation protocol divides the group signing key among each participant using Shamir secret sharing.

FROST keys produced by a key generation protocol can be represented by the following parameters:

#### General parameters

|No|Name|Type|Range (Inclusive)|Description|
|---|---|---|---|---|
|1|_max_participants_|integer|2..n|maximum number of signers allowed in the signing process|
|2|_min_participants_|integer|2..max_participants|minimum number of signers required to initiate the signing process|

#### Participant parameters

|No|Name|Type|Range (Inclusive)|Description|
|---|---|---|---|---|
|1|_id_|integer|1..max_participants|used to uniquely identify each participant, must be distinct from the _id_ of every other participant|
|2|_secshare_<sub>id</sub>|integer (scalar)|1..n|signing key of a participant|
|3|_pubshare_<sub>id</sub>|curve point (_plain public key_)|shouldn't be infinity point|public key associated with the above signing key|
|4|_group_pubkey_|curve point (_plain public key_)|shouldn't be infinity point|group public key used to verify the BIP340 Schnorr signature produced by the FROST-signing protocol|

### Correctness Conditions

The notations used in this section can be found in _todo link_

#### Public shares condition

For each participants _i_ in the range [_1..max_participants_], their public share must equal to their secret share scalar multiplied with the generator point, represented as: _pubshare<sub>i</sub> = secshare<sub>i</sub>⋅G_

#### Group public key condition

TODO: For this condt, the ROAST paper forces the |T| = t. Why? Every signer set with t <= |T| <= n, must satisfy this condition, right?

Consider a set of participants, denoted by _T_, chosen from a total pool of participants whose size is _max_participants_. For this set _T_, we can define a special parameter called the "group secret key". It is calculated by summing the secret share and interpolating value for each participant in T:

_group_seckey_ = sum (_derive_interpolating_value<sub>j, T</sub>_._secshare_<sub>j</sub>) mod _n_, for every _j_ in _T_

For all possible values of T, the group public key must equal to their group secret key scalar multiplied by the generator point represented as _group_pubkey<sub>i</sub>_ = _group_seckey<sub>j, T</sub>_⋅_G_.

## Algorithms

The following specification of the algorithms has been written with a focus on clarity. As a result, the specified algorithms are not always optimal in terms of computation and space. In particular, some values are recomputed but can be cached in actual implementations (todo see _mention link here_).

### Notation

The following conventions are used, with constants as defined for [secp256k1](https://www.secg.org/sec2-v2.pdf). We note that adapting this proposal to other elliptic curves is not straightforward and can result in an insecure scheme.

- Lowercase variables represent integers or byte arrays.
    - The constant _p_ refers to the field size, _0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F_.
    - The constant _n_ refers to the curve order, _0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141_.
    - The constant _num_participants_ refers to number of participants involved in the signing process, must be at least _min_participants_ but must not be larger than _max_participants_.
- Uppercase variables refer to points on the curve with equation _y<sup>2</sup> = x<sup>3</sup> + 7_ over the integers modulo _p_.
    - _is_infinite(P)_ returns whether _P_ is the point at infinity.
    - _x(P)_ and _y(P)_ are integers in the range _0..p-1_ and refer to the X and Y coordinates of a point _P_ (assuming it is not infinity).
    - The constant _G_ refers to the base point, for which _x(G) = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798_ and _y(G) = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8_.
    - Addition of points refers to the usual [elliptic curve group operation](https://en.wikipedia.org/wiki/Elliptic_curve#The_group_law).
    - [Multiplication (⋅) of an integer and a point](https://en.wikipedia.org/wiki/Elliptic_curve_point_multiplication) refers to the repeated application of the group operation.
- Functions and operations:
    - _||_ refers to byte array concatenation.
    - The function _x[i:j]_, where _x_ is a byte array and _i, j ≥ 0_, returns a _(j - i)_-byte array with a copy of the _i_th byte (inclusive) to the _j_th byte (exclusive) of _x_.
    - The function _bytes(n, x)_, where _x_ is an integer, returns the n-byte encoding of _x_, most significant byte first.
    - The constant _empty_bytestring_ refers to the empty byte array. It holds that _len(empty_bytestring) = 0_.
    - The function _xbytes(P)_, where _P_ is a point for which _not is_infinite(P)_, returns _bytes(32, x(P))_.
    - The function _len(x)_ where _x_ is a byte array returns the length of the array.
    - The function _has_even_y(P)_, where _P_ is a point for which _not is_infinite(P)_, returns _y(P) mod 2 == 0_.
    - The function _with_even_y(P)_, where _P_ is a point, returns _P_ if _is_infinite(P)_ or _has_even_y(P)_. Otherwise, _with_even_y(P)_ returns _-P_.
    - The function _cbytes(P)_, where _P_ is a point for which _not is_infinite(P)_, returns _a || xbytes(P)_ where _a_ is a byte that is _2_ if _has_even_y(P)_ and _3_ otherwise.
    - The function _cbytes_ext(P)_, where _P_ is a point, returns _bytes(33, 0)_ if _is_infinite(P)_. Otherwise, it returns _cbytes(P)_.
    - The function _int(x)_, where _x_ is a 32-byte array, returns the 256-bit unsigned integer whose most significant byte first encoding is _x_.
    - The function _lift_x(x)_, where _x_ is an integer in range _0..2<sup>256</sup>-1_, returns the point _P_ for which _x(P) = x<sup>todo add ref here</sup>_ and _has_even_y(P)_, or fails if _x_ is greater than _p-1_ or no such point exists. The function _lift_x(x)_ is equivalent to the following pseudocode: TODO: add footnote
		- Fail if _x > p-1_.
		- Let _c = x<sup>3</sup> + 7 mod p_.
		- Let _y' = c<sup>(p+1)/4</sup> mod p_.
		- Fail if _c ≠ y'<sup>2</sup> mod p_.
		 - Let _y = y'_ if _y' mod 2 = 0_, otherwise let _y = p - y'_ .
		- Return the unique point _P_ such that _x(P) = x_ and _y(P) = y_.
    - The function _cpoint(x)_, where _x_ is a 33-byte array (compressed serialization), sets _P = lift_x(int(x[1:33]))_ and fails if that fails. If _x[0] = 2_ it returns _P_ and if _x[0] = 3_ it returns _-P_. Otherwise, it fails.
    - The function _cpoint_ext(x)_, where _x_ is a 33-byte array (compressed serialization), returns the point at infinity if _x = bytes(33, 0)_. Otherwise, it returns _cpoint(x)_ and fails if that fails.
    - The function _hash<sub>tag</sub>(x)_ where _tag_ is a UTF-8 encoded tag name and _x_ is a byte array returns the 32-byte hash _SHA256(SHA256(tag) || SHA256(tag) || x)_.
    - The function _count(lst, x)_, where _lst_ is a list of integers containing _x_, returns the number of times _x_ appears in _lst_.
    - The function _has_unique_elements(lst)_, where _lst_ is a list of integers, returns True if _count(lst, x)_ returns 1 for all _x_ in _lst_. Otherwise returns False. The function _has_unique_elements(lst)_ is equivalent to the following pseudocode:
        - For _x_ in _lst_:
          - if _count(lst, x)_ > 1:
            - Return False
        - Return True
    - The function _int_ids(lst)_, where _lst_ is a list of 32-byte array, returns a list of integers. The function _int_ids(lst)_ is equivalent to the following pseudocode:
	    - _res_ = []
	    - For _x_ in _lst_:
		    - Fail if _int(x)_ ≥ n or _int(x)_ < 1
		    - _res.append(int(x))_
		- Return _res_
    - The function _concat_bytearrays(lst)_, where _lst_ is a list of byte array elements, returns a single byte array. The function _concat_bytearrays(lst)_ is equivalent to the following pseudocode:
	    - _res_ = _empty_bytestring_
	    - For _x_ in _lst_:
		    - _res = res || x_
		- Return _res_
    - The function _sorted(lst)_, where _lst_ is a list of byte array elements, returns a list of byte array sorted based on the numerical values of its elements in ascending order, preserving the relative order of equal elements.
- Other:
    - Tuples are written by listing the elements within parentheses and separated by commas. For example, _(2, 3, 1)_ is a tuple.

TODO remove unused functions above

### Tweaking Group Public Key

#### Tweak Context

The Tweak Context is a data structure consisting of the following elements:
- The point _Q_ representing the potentially tweaked group public key: an elliptic curve point
- The accumulated tweak _tacc_: an integer with _0 ≤ tacc < n_
- The value _gacc_: 1 or -1 mod n

We write "Let _(Q, gacc, tacc) = tweak_ctx_" to assign names to the elements of a Tweak Context.

Algorithm _TweakCtxInit(group_pk):_
- Input:
    - The group public key _group_pk_: a 33-byte array
- Let _Q = cpoint(group_pk)_
- Fail if _is_infinite(Q)_.
- Let _gacc = 1_
- Let _tacc = 0_
- Return _tweak_ctx = (Q, gacc, tacc)_.

Algorithm _GetXonlyPubkey(tweak_ctx)_:
- Let _(Q, _, _) = tweak_ctx_
- Return _xbytes(Q)_

Algorithm _GetPlainPubkey(tweak_ctx)_:
- Let _(Q, _, _) = tweak_ctx_
- Return _cbytes(Q)_

#### Applying Tweaks

Algorithm _ApplyTweak(tweak_ctx, tweak, is_xonly_t)_:
- Inputs:
    - The _tweak_ctx_: a Tweak Context (todo link the defn) data structure
    - The _tweak_: a 32-byte array
    - The tweak mode _is_xonly_t_: a boolean
- Let _(Q, gacc, tacc) = tweak_ctx_
- If _is_xonly_t_ and _not has_even_y(Q)_:
    - Let _g = -1 mod n_
- Else:
    - Let _g = 1_
- Let _t = int(tweak)_; fail if _t ≥ n_
- Let _Q' = g⋅Q + t⋅G_
    - Fail if _is_infinite(Q')_
- Let _gacc' = g⋅gacc mod n_
- Let _tacc' = t + g⋅tacc mod n_
- Return _tweak_ctx' = (Q', gacc', tacc')_

### Nonce Generation

TODO include participant identifier in the input args? Commiting to a signer set will prevent using the preprocessed nonce with another signer set. But commiting only the signer's pariticipant identifier should be fine.
think: what the max msg len? we use 8 bytes while hashing it

Algorithm _NonceGen(secshare, pubshare, group_pk, m, extra_in)_:
- Inputs:
    - The participant’s secret signing key _secshare_: a 32-byte array (optional argument)
    - The corresponding public key _pubshare_: a 33-byte array (optional argument)
    - The x-only group public key _group_pk_: a 32-byte array (optional argument)
    - The message _m_: a byte array (optional argument)
    - The auxiliary input _extra_in_: a byte array with _0 ≤ len(extra_in) ≤ 2<sup>32</sup>-1_ (optional argument)
- Let _rand'_ be a 32-byte array freshly drawn uniformly at random
- If the optional argument _secshare_ is present:
    - Let _rand_ be the byte-wise xor of _secshare_ and _hash<sub>FROST/aux</sub>(rand')_
- Else:
    - Let _rand = rand'_
- If the optional argument _pubshare_ is not present:
    - Let _pubshare_ = _empty_bytestring_
- If the optional argument _group_pk_ is not present:
    - Let _group_pk_ = _empty_bytestring_
- If the optional argument _m_ is not present:
    - Let _m_prefixed = bytes(1, 0)_
- Else:
    - Let _m_prefixed = bytes(1, 1) || bytes(8, len(m)) || m_
- If the optional argument _extra_in_ is not present:
    - Let _extra_in = empty_bytestring_
- Let _k<sub>i</sub> = int(hash<sub>FROST/nonce</sub>(rand || bytes(1, len(pubshare)) || pubshare || bytes(1, len(group_pk)) || group_pk || m_prefixed || bytes(4, len(extra_in)) || extra_in || bytes(1, i - 1))) mod n_ for _i = 1,2_
- Fail if _k<sub>1</sub> = 0_ or _k<sub>2</sub> = 0_
- Let _R<sub>⁎,1</sub> = k<sub>1</sub>⋅G, R<sub>⁎,2</sub> = k<sub>2</sub>⋅G_
- Let _pubnonce = cbytes(R<sub>,1</sub>) || cbytes(R<sub>⁎,2</sub>)_
- Let _secnonce = bytes(32, k<sub>1</sub>) || bytes(32, k<sub>2</sub>)_
- Return _(secnonce, pubnonce)_

### Nonce Aggregation

Algorithm _NonceAgg(pubnonce<sub>1..u</sub>, id<sub>1..u</sub>)_:
- Inputs:
    - The number of signers _u_: an integer with _min_participants ≤ u ≤ max_participants_
    - The public nonces _pubnonce<sub>1..u</sub>_: _u_ 66-byte arrays
    - The participant identifiers _id<sub>1..u</sub>_: _u_ 32-byte arrays with _1 ≤ int(id<sub>i</sub>) ≤ max_participants_
- For _j = 1 .. 2_:
    - For _i = 1 .. u_:
        - Let _R<sub>i,j</sub> = cpoint(pubnonce<sub>i</sub>[(j-1)_33:j_33])_; fail if that fails and blame signer _id<sub>i</sub>_ for invalid _pubnonce_.
    - Let _R<sub>j</sub> = R<sub>1,j</sub> + R<sub>2,j</sub> + ... + R<sub>u,j</sub>_
- Return _aggnonce = cbytes_ext(R<sub>1</sub>) || cbytes_ext(R<sub>2</sub>)_

TODO: change identifiers of signers to _u_ 32-byte arrays? (currently we use _u_ ints)
TODO: then, have a function that converts ids byte-array to integer array, must check for 1 <= ids[i] <= max_participant

### Session Context

The Session Context is a data structure consisting of the following elements:
- The number _u_ of participants available in the signing session with _min_participants ≤ u ≤ max_participants_
- The participant identifiers of the signers _id<sub>1..u</sub>: _u_ 32-byte arrays with 1 _≤ int(id<sub>i</sub>) ≤ max_participants_ < n
- The individual public shares _pubshare<sub>1..u</sub>_: _u_ 33-byte arrays
- The aggregate public nonce of signers _aggnonce_: a 66-byte array
- The number _v_ of tweaks with _0 ≤ v < 2^32_
- The tweaks _tweak<sub>1..v</sub>_: _v_ 32-byte arrays
- The tweak modes _is_xonly_t<sub>1..v</sub>_ : _v_ booleans
- The message _m_: a byte array

We write "Let _(u, id<sub>1..u</sub>, pubshare<sub>1..u</sub>, aggnonce, v, tweak<sub>1..v</sub>, is_xonly_t<sub>1..v</sub>, m) = session_ctx_" to assign names to the elements of a Session Context.

Algorithm _GetSessionValues(session_ctx)_:
- Let _(u, id<sub>1..u</sub>, pubshare<sub>1..u</sub>, aggnonce, v, tweak<sub>1..v</sub>, is_xonly_t<sub>1..v</sub>, m) = session_ctx_
- _group_pk_ = _DeriveGroupPubkey(id<sub>1..u</sub>, pubshare<sub>1..u</sub>)_
- Let _tweak_ctx<sub>0</sub> = TweakCtxInit(group_pk)_; fail if that fails
- For _i = 1 .. v_:
    - Let _tweak_ctx<sub>i</sub> = ApplyTweak(tweak_ctx<sub>i-1</sub>, tweak<sub>i</sub>, is_xonly_t<sub>i</sub>)_; fail if that fails
- Let _(Q, gacc, tacc) = tweak_ctx<sub>v</sub>_
- Let _ser_ids_ = _concat_bytearrays(sorted(id<sub>1..u</sub>))_
- Let b = _int(hash<sub>FROST/noncecoef</sub>(ser_ids || aggnonce || xbytes(Q) || m)) mod n_
- Let _R<sub>1</sub> = cpoint_ext(aggnonce[0:33]), R<sub>2</sub> = cpoint_ext(aggnonce[33:66])_; fail if that fails and blame nonce aggregator for invalid _aggnonce_.
- Let _R' = R<sub>1</sub> + b⋅R<sub>2</sub>_
- If _is_infinite(R'):_
    - _Let final nonce_ R = G _(see dealing with inf nonce agg link)_
- _Else:_
    - _Let final nonce_ R = R'
- Let _e = int(hash<sub>BIP0340/challenge</sub>((xbytes(R) || xbytes(Q) || m))) mod n_
- _Return_ (Q, gacc, tacc, b, R, e)

Algorithm _GetSessionInterpolatingValue(session_ctx, my_id)_:
- Let _(u, id<sub>1..u</sub>, _, _, _, _, _) = session_ctx_
- Return _DeriveInterpolatingValue(id<sub>1..u</sub>, my_id)_; fail if that fails

Internal Algorithm _DeriveInterpolatingValue(id<sub>1..u</sub>, my_id):_
- Fail if _my_id_ not in _id<sub>1..u</sub>_
- Fail if not _has_unique_elements(id<sub>1..u</sub>)
- _integer_id<sub>1..u</sub> = int_ids(id<sub>1..u</sub>)_; Fail if that fails
- Return _DeriveInterpolatingValueInternal(_integer_id<sub>1..u</sub>_, int(my_id))_

Internal Algorithm _DeriveInterpolatingValueInternal(id<sub>1..u</sub>, my_id):_
- Let _num = 1_
- Let _denom = 1_
- For _i = 1..u_:
    - If _id<sub>i</sub>_ ≠ _my_id_:
	    - Let _num_ = _num⋅id<sub>i</sub>_
	    - Let _denom_ = _denom⋅(id<sub>i</sub> - my_id)_
- _lambda_ = _num⋅denom<sup>-1</sup> mod n_
- Return _lambda_

Algorithm _GetSessionGroupPubkey(session_ctx)_:
- Let _(u, id<sub>1..u</sub>, pubshare<sub>1..u</sub>, _, _, _, _) = session_ctx_
- Return _DeriveGroupPubkey(id<sub>1..u</sub>, pubshare<sub>1..u</sub>)_; fail if that fails

Algorithm _DeriveGroupPubkey(id<sub>1..u</sub>, pubshare<sub>1..u</sub>)_
- _inf_point = bytes(33, 0)_
- _Q_ = _cpoint_ext(inf_point)_
- For _i_ = _1..u_:
    - _P_ = _cpoint(pubshare<sub>i</sub>)_; fail if that fails
    - _lambda_ = _DeriveInterpolatingValue(id<sub>1..u</sub>, id<sub>i</sub>)_
    - _Q_ = _Q_ + _lambda⋅P_
- Return _Q_

Algorithm _SessionHasSignerPubshare(session_ctx, signer_pubshare)_:
- Let _(u, _, pubshare<sub>1..u</sub>, _, _, _, _) = session_ctx_
- If _signer_pubshare in pubshare<sub>1..u</sub>_
	- Return True
- Otherwise Return False

### Signing

TODO: should we add 1 <= _my_id_ <= max_pariticiapants, check here?
Algorithm _Sign(secnonce, secshare, my_id, session_ctx)_:
- Inputs:
    - The secret nonce _secnonce_ that has never been used as input to _Sign_ before: a 64-byte array
    - The secret signing key _secshare_: a 32-byte array
    - The identifier of the signing participant _my_id_: a 32-byte array with 1 _≤ int(my_id) ≤ max_participants_
    - The _session_ctx_: a Session Context (todo _link to defn_) data structure
- Let _(Q, gacc, _, b, R, e) = GetSessionValues(session_ctx)_; fail if that fails
- Let _k<sub>1</sub>' = int(secnonce[0:32]), k<sub>2</sub>' = int(secnonce[32:64])_
- Fail if _k<sub>i</sub>' = 0_ or _k<sub>i</sub>' ≥ n_ for _i = 1..2_
- Let _k<sub>1</sub> = k<sub>1</sub>', k<sub>2</sub> = k<sub>2</sub>'_ if _has_even_y(R)_, otherwise let _k<sub>1</sub> = n - k<sub>1</sub>', k<sub>2</sub> = n - k<sub>2</sub>'_
- Let _d' = int(secshare)_
- Fail if _d' = 0_ or _d' ≥ n_
- Let _P = d'⋅G_
- Let _pubshare = cbytes(P)_
- Fail if _SessionHasSignerPubshare(session_ctx, pubshare) = False_
- Let _&lambda; = GetSessionInterpolatingValue(session_ctx, my_id)_; fail if that fails
- Let _g = 1_ if _has_even_y(Q)_, otherwise let _g = -1 mod n_
- Let _d = g⋅gacc⋅d' mod n_ (See _negating seckey when signing_)
- Let _s = (k<sub>1</sub> + b⋅k<sub>2</sub> + e⋅&lambda;⋅d) mod n_
- Let _psig = bytes(32, s)_
- Let _pubnonce = cbytes(k<sub>1</sub>'⋅G) || cbytes(k<sub>2</sub>'⋅G)_
- If _PartialSigVerifyInternal(psig, my_id, pubnonce, pubshare, session_ctx)_ (see below) returns failure, fail
- Return partial signature _psig_

### Partial Signature Verification

Algorithm _PartialSigVerify(psig, id<sub>1..u</sub>, pubnonce<sub>1..u</sub>, pubshare<sub>1..u</sub>, tweak<sub>1..v</sub>, is_xonly_t<sub>1..v</sub>, m, i)_:
- Inputs:
    - The partial signature _psig_: a 32-byte array
    - The number _u_ of identifiers, public nonces, and individual public shares with _min_participants ≤ u ≤ max_participants_
    - The participant identifiers _id<sub>1..u</sub>_: _u_ 32-byte array with _1 ≤ int(id<sub>i</sub>) ≤ max_participants_
    - The public nonces _pubnonce<sub>1..u</sub>_: _u_ 66-byte arrays
    - The individual public shares _pubshare<sub>1..u</sub>_: _u_ 33-byte arrays
    - The number _v_ of tweaks with _0 ≤ v < 2^32_
    - The tweaks _tweak<sub>1..v</sub>_: _v_ 32-byte arrays
    - The tweak modes _is_xonly_t<sub>1..v</sub>_ : _v_ booleans
    - The message _m_: a byte array
    - The index _i_ of the signer in the list of identifiers, public nonces, and individual public shares where _0 < i ≤ u_
- Let _aggnonce = NonceAgg(pubnonce<sub>1..u</sub>)_; fail if that fails
- Let _session_ctx = (u, id<sub>1..u</sub>, pubshare<sub>1..u</sub>, aggnonce, v, tweak<sub>1..v</sub>, is_xonly_t<sub>1..v</sub>, m)_
- Run _PartialSigVerifyInternal(psig, id<sub>i</sub>, pubnonce<sub>i</sub>, pubshare<sub>i</sub>, session_ctx)_
- Return success iff no failure occurred before reaching this point.

Internal Algorithm _PartialSigVerifyInternal(psig, my_id, pubnonce, pubshare, session_ctx)_:
- Let _(Q, gacc, _, b, R, e) = GetSessionValues(session_ctx)_; fail if that fails
- Let _s = int(psig)_; fail if _s ≥ n_
- Fail if _SessionHasSignerPubshare(session_ctx, pubshare) = False_
- Let _R<sub>⁎,1</sub> = cpoint(pubnonce[0:33]), R<sub>⁎,2</sub> = cpoint(pubnonce[33:66])_
- Let _Re<sub>⁎</sub>' = R<sub>⁎,1</sub> + b⋅R<sub>⁎,2</sub>_
- Let effective nonce _Re<sub>⁎</sub> = Re<sub>⁎</sub>'_ if _has_even_y(R)_, otherwise let _Re<sub>⁎</sub> = -Re<sub>⁎</sub>'_
- Let _P = cpoint(pubshare)_; fail if that fails
- Let _&lambda; = GetSessionInterpolatingValue(session_ctx, my_id)_
- Let _g = 1_ if _has_even_y(Q)_, otherwise let _g = -1 mod n_
- Let _g' = g⋅gacc mod n_ (See _link to neg of seckey when signing_)
- Fail if _s⋅G ≠ Re<sub>⁎</sub> + e⋅&lambda;⋅g'⋅P_
- Return success iff no failure occurred before reaching this point.

### Partial Signature Aggregation

Algorithm _PartialSigAgg(psig<sub>1..u</sub>, id<sub>1..u</sub>, session_ctx)_:
- Inputs:
    - The number _u_ of signatures with _min_participants ≤ u ≤ max_participants_
    - The partial signatures _psig<sub>1..u</sub>_: _u_ 32-byte arrays
    - The participant identifiers _id<sub>1..u</sub>_: _u_ 32-byte arrays with _1 ≤ int(id<sub>i</sub>) ≤ max_participants_
    - The _session_ctx_: a Session Context (todo _link to defn_) data structure
- Let _(Q, _, tacc, _, _, R, e) = GetSessionValues(session_ctx)_; fail if that fails
- For _i = 1 .. u_:
    - Let _s<sub>i</sub> = int(psig<sub>i</sub>)_; fail if _s<sub>i</sub> ≥ n_ and blame signer _id<sub>i</sub>_ for invalid partial signature.
- Let _g = 1_ if _has_even_y(Q)_, otherwise let _g = -1 mod n_
- Let _s = s<sub>1</sub> + ... + s<sub>u</sub> + e⋅g⋅tacc mod n_
- Return _sig =_ xbytes(R) || bytes(32, s)

### Test Vectors & Reference Code

We provide a naive, highly inefficient, and non-constant time [pure Python 3 reference implementation of the group public key tweaking, nonce generation, partial signing, and partial signature verification algorithms](./reference/reference.py).

Standalone JSON test vectors are also available in the [same directory](./reference/vectors/), to facilitate porting the test vectors into other implementations.

The reference implementation is for demonstration purposes only and not to be used in production environments.

## Remarks on Security and Correctness

### Tweaking Definition

Two modes of tweaking the group public key are supported. They correspond to the following algorithms:

Algorithm _ApplyPlainTweak(P, t)_:
- Inputs:
    - _P_: a point
    - The tweak _t_: an integer with _0 ≤ t < n_
- Return _P + t⋅G_

Algorithm _ApplyXonlyTweak(P, t)_:
- Return _with_even_y(P) + t⋅G_

### Negation of the Secret Share when Signing

During the signing process, the *[Sign](./README.md#signing)* algorithm might have to negate the secret share in order to produce a partial signature for an X-only group public key. This public key is derived from *u* public shares and *u* participant identifiers (denoted by the signer set *U*) and then tweaked *v* times (X-only or plain).

The following elliptic curve points arise as intermediate steps when creating a signature:  
• _P<sub>i</sub>_ as computed in any compatible key generation method is the point corresponding to the *i*-th signer's public share. Defining *d<sub>i</sub>'* to be the *i*-th signer's secret share as an integer, i.e., the *d’* value as computed in the *Sign* algorithm of the *i*-th signer, we have:  
&emsp;&ensp;*P<sub>i</sub> = d<sub>i</sub>'⋅G*  
• *Q<sub>0</sub>* is the group public key derived from the signer’s public shares. It is identical to the value *Q* computed in *DeriveGroupPubkey* and therefore defined as:  
&emsp;&ensp;_Q<sub>0</sub> = &lambda;<sub>1, U</sub>⋅P<sub>1</sub> + &lambda;<sub>2, U</sub>⋅P<sub>2</sub> + ... + &lambda;<sub>u, U</sub>⋅P<sub>u</sub>_  
• *Q<sub>i</sub>* is the tweaked group public key after the *i*-th execution of *ApplyTweak* for *1 ≤ i ≤ v*. It holds that  
&emsp;&ensp;*Q<sub>i</sub> = f(i-1) + t<sub>i</sub>⋅G* for *i = 1, ..., v* where  
&emsp;&ensp;&emsp;&ensp;*f(i-1) := with_even_y(Q<sub>i-1</sub>)* if *is_xonly_t<sub>i</sub>* and  
&emsp;&ensp;&emsp;&ensp;*f(i-1) := Q<sub>i-1</sub>* otherwise.  
• *with_even_y(Q*<sub>v</sub>*)* is the final result of the group public key derivation and tweaking operations. It corresponds to the output of *GetXonlyPubkey* applied on the final Tweak Context.

The signer's goal is to produce a partial signature corresponding to the final result of group pubkey derivation and tweaking, i.e., the X-only public key *with_even_y(Q<sub>v</sub>)*.

For _1 ≤ i ≤ v_, we denote the value _g_ computed in the _i_-th execution of _ApplyTweak_ by _g<sub>i-1</sub>_. Therefore, _g<sub>i-1</sub>_ is _-1 mod n_ if and only if _is_xonly_t<sub>i</sub>_ is true and _Q<sub>i-1</sub>_ has an odd Y coordinate. In other words, _g<sub>i-1</sub>_ indicates whether _Q<sub>i-1</sub>_ needed to be negated to apply an X-only tweak:  
&emsp;&ensp;_f(i-1) = g<sub>i-1</sub>⋅Q<sub>i-1</sub>_ for _1 ≤ i ≤ v_.  
Furthermore, the _Sign_ and _PartialSigVerify_ algorithms set value _g_ depending on whether Q<sub>v</sub> needed to be negated to produce the (X-only) final output. For consistency, this value _g_ is referred to as _g<sub>v</sub>_ in this section.  
&emsp;&ensp;_with_even_y(Q<sub>v</sub>) = g<sub>v</sub>⋅Q<sub>v</sub>_.

So, the (X-only) final public key is  
&emsp;&ensp;_with_even_y(Q<sub>v</sub>)_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅Q<sub>v</sub>_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅(f(v-1)_ + _t<sub>v</sub>⋅G)_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅(g<sub>v-1</sub>⋅(f(v-2)_ + _t<sub>v-1</sub>⋅G)_ + _t<sub>v</sub>⋅G)_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅g<sub>v-1</sub>⋅f(v-2)_ + _g<sub>v</sub>⋅(t<sub>v</sub>_ + _g<sub>v-1</sub>⋅t<sub>v-1</sub>)⋅G_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅g<sub>v-1</sub>⋅f(v-2)_ + _(sum<sub>i=v-1..v</sub> t<sub>i</sub>⋅prod<sub>j=i..v</sub> g<sub>j</sub>)⋅G_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅g<sub>v-1</sub>⋅...⋅g<sub>1</sub>⋅f(0)_ + _(sum<sub>i=1..v</sub> t<sub>i</sub>⋅prod<sub>j=i..v</sub> g<sub>j</sub>)⋅G_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅...⋅g<sub>0</sub>⋅Q<sub>0</sub>_ + _g<sub>v</sub>⋅tacc<sub>v</sub>⋅G_   
&emsp;&ensp;where _tacc<sub>i</sub>_ is computed by _TweakCtxInit_ and _ApplyTweak_ as follows:  
&emsp;&ensp;&emsp;&ensp;_tacc<sub>0</sub>_ = _0_  
&emsp;&ensp;&emsp;&ensp;_tacc<sub>i</sub>_ = _t<sub>i</sub>_ + _g<sub>i-1</sub>⋅tacc<sub>i-1</sub> for i=1..v mod n_  
&emsp;&ensp;for which it holds that _g<sub>v</sub>⋅tacc<sub>v</sub>_ = _sum<sub>i=1..v</sub> t<sub>i</sub>⋅prod<sub>j=i..v</sub> g<sub>j</sub>_.

_TweakCtxInit_ and _ApplyTweak_ compute  
&emsp;&ensp;_gacc<sub>0</sub>_ = 1  
&emsp;&ensp;_gacc<sub>i</sub>_ = _g<sub>i-1</sub>⋅gacc<sub>i-1</sub> for i=1..v mod n_  
So we can rewrite above equation for the final public key as  
&emsp;&ensp;_with_even_y(Q<sub>v</sub>)_ = _g<sub>v</sub>⋅gacc<sub>v</sub>⋅Q<sub>0</sub>_ + _g<sub>v</sub>⋅tacc<sub>v</sub>⋅G._

Then we have  
&emsp;&ensp;_with_even_y(Q<sub>v</sub>)_ - _g<sub>v</sub>⋅tacc<sub>v</sub>⋅G_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅gacc<sub>v</sub>⋅Q<sub>0</sub>_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅gacc<sub>v</sub>⋅(&lambda;<sub>1, U</sub>⋅P<sub>1</sub> + ... + &lambda;<sub>u, U</sub>⋅P<sub>u</sub>)_  
&emsp;&ensp;&emsp;&ensp;= _g<sub>v</sub>⋅gacc<sub>v</sub>⋅(&lambda;<sub>1, U</sub>⋅d<sub>1</sub>'⋅G + ... + &lambda;<sub>u, U</sub>⋅d<sub>u</sub>'⋅G)_  
&emsp;&ensp;&emsp;&ensp;= _sum<sub>i=1..u</sub>(g<sub>v</sub>⋅gacc<sub>v</sub>⋅&lambda;<sub>i, U</sub>⋅d<sub>i</sub>')*G._  

Intuitively, _gacc<sub>i</sub>_ tracks accumulated sign flipping and _tacc<sub>i</sub>_ tracks the accumulated tweak value after applying the first _i_ individual tweaks. Additionally, _g<sub>v</sub>_ indicates whether _Q<sub>v</sub>_ needed to be negated to produce the final X-only result. Thus, signer _i_ multiplies its secret share _d<sub>i</sub>'_ with _g<sub>v</sub>⋅gacc<sub>v</sub>_ in the [_Sign_](./README.md#signing) algorithm.

#### Negation of the Pubshare when Partially Verifying

As explained in [Negation Of The Secret Share When Signing](./README.md#negation-of-the-secret-share-when-signing) the signer uses a possibly negated secret share  
&emsp;&ensp;_d = g<sub>v</sub>⋅gacc<sub>v</sub>⋅d' mod n_  
when producing a partial signature to ensure that the aggregate signature will correspond to a group public key with even Y coordinate.

The [_PartialSigVerifyInternal_](./README.md#partial-signature-verification) algorithm is supposed to check  
&emsp;&ensp;_s⋅G = Re<sub>⁎</sub> + e⋅&lambda;⋅d⋅G_.

The verifier doesn't have access to _d⋅G_ but can construct it using the participant public share _pubshare_ as follows:  
_d⋅G  
&emsp;&ensp;= g<sub>v</sub>⋅gacc<sub>v</sub>⋅d'⋅G  
&emsp;&ensp;= g<sub>v</sub>⋅gacc<sub>v</sub>⋅cpoint(pubshare)_  
Note that the group public key and list of tweaks are inputs to partial signature verification, so the verifier can also construct _g<sub>v</sub>_ and _gacc<sub>v</sub>_.


### Dealing with Infinity in Nonce Aggregation

### Choosing the Size of the Nonce

## Backwards Compatibility

This document proposes a standard for the FROST threshold signature scheme that is compatible with [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki). FROST is _not_ compatible with ECDSA signatures traditionally used in Bitcoin.

## Acknowledgments

<!-- Footnotes -->

[^t-edge-cases]: While `t = n` and `t = 1` are in principle supported, simpler alternatives are available in these cases.
In the case `t = n`, using a dedicated `n`-of-`n` multi-signature scheme such as MuSig2 (see [BIP327](bip-0327.mediawiki)) instead of FROST avoids the need for an interactive DKG.
The case `t = 1` can be realized by letting one signer generate an ordinary [BIP340](bip-0340.mediawiki) key pair and transmitting the key pair to every other signer, who can check its consistency and then simply use the ordinary [BIP340](bip-0340.mediawiki) signing algorithm.
Signers still need to ensure that they agree on key pair. A detailed specification is not in scope of this document.
