::

  ZIP: XXXX
  Title: ZSA Transaction Format and Digests
  Owners: Daira-Emma Hopwood <daira@jacaranda.org>
          Pablo Kogan <pablo@qed-it.com>
          Vivek Arte <vivek@qed-it.com>
  Credits:
  Status: Draft
  Category: Consensus
  Created: 2026-10-22
  License: MIT
  Discussions-To: <https://github.com/zcash/zips/issues/618>


===========
Terminology
===========

The key word "MUST" in this document is to be interpreted as described in BCP 14
[#BCP14]_ when, and only when, it appears in all capitals.

The terms "Zcash Shielded Asset", "OrchardZSA" and "ZSA" in this document are to be interpreted
as described in ZIP 226 [#zip-0226]_.

The terms "Asset", "Custom Asset", "Issuance Action" and "Issuance Bundle" in this document are
to be interpreted as described in ZIP 227 [#zip-0227]_.

The term "Action Group" in this document is to be interpreted as described in ZIP 228 [#zip-0228]_.

The term "Ironwood pool" in this document is to be interpreted as described in ZIP 229 [#zip-0229]_.

"vZSA" denotes the transaction version defined in this ZIP (provisionally 7). It is distinct from
version 6, defined in ZIP 229 [#zip-0229]_.

The character § is used when referring to sections of the Zcash Protocol Specification
[#protocol]_.


========
Abstract
========

This proposal defines a new Zcash peer-to-peer transaction format, which supports the
changes for the inclusion of Zcash Shielded Assets [#zip-0226]_ [#zip-0227]_ in a future network upgrade.
It follows the same design pattern as the v6 transaction format [#zip-0229]_.

This proposal also adopts "sighash algorithm versioning" from ZIP 248 [#zip-0248]_:
where previously each transaction version had a single associated sighash
algorithm, going forward it will be possible for signers to use any sighash
algorithm within the closed set specified for a given transaction version (and
made available in consensus via network upgrades).

This ZIP defines the sighash algorithms associated with this new transaction format.

Issuance and burns are public, as in ZIP 227 [#zip-0227]_ and ZIP 226 [#zip-0226]_, while
shielded outputs look the same for every Asset, including ZEC (see `Privacy Implications`_).


==========
Motivation
==========

The OrchardZSA protocol requires serialized data elements that are distinct from
any previous Zcash transaction. Since ZIP 244 was activated in NU5, the
v5 and later serialized transaction formats are not consensus-critical.
Thus, this ZIP defines a format that can easily accommodate future extensions,
where elements of a given pool are kept separate.


====================
Privacy Implications
====================

ZIP 226 [#zip-0226-privacy-implications]_ covers the privacy of the OrchardZSA protocol. This
section covers only what the encoding defined here makes visible.

* Outputs of every Asset, including ZEC, have the same note plaintext lead byte and the same
  ciphertext size, so the encoding does not show which Asset an Action carries.
* Each Action Group shows whether ZSAs are enabled for it. With ZSAs disabled it can only carry
  ZEC [#zip-0226-enablezsa]_, so a wallet that disables them for ZEC-only transfers reveals that.
* Issuance is public: the issuer, each Asset description hash, whether the Asset is finalized,
  and every issued note in full, including its recipient and value [#zip-0227]_.
* Burns are public: each burnt Asset and amount [#zip-0226-burn-mechanism]_.
* The sighash version and the Action Group expiry height are fixed by consensus, and a transaction
  has at most one Action Group, so none of these can be used to tell wallets apart.
* If version 6 transactions stay valid alongside this format, which of the two versions a
  transaction uses to reach the Ironwood pool is visible.


============
Requirements
============

The new format must fully support the OrchardZSA protocol.

The new format should lend itself to future extension or pruning to add or remove
value pools.

The computation of the non-malleable transaction identifier hash must include all
newly incorporated elements except those that attest to transaction validity.

The computation of the commitment to authorizing data for a transaction must include
all newly incorporated elements that attest to transaction validity.


================
Non-requirements
================

More general forms of extensibility, such as defining a key/value format that
allows for parsers that are unaware of some components, are not required.


=============
Specification
=============

All fields in this specification are encoded as little-endian.

This section specifies the vZSA transaction format, its note plaintext encoding, and how its
txid, auth digest and signature digest are computed.

Sighash versioning
==================

As in ZIP 248 [#zip-0248]_, the sighash version is encoded as a single byte alongside any associated data
that the sighash algorithm version requires (for deterministically computing the
digest):

``sighashInfo = [sighashVersion] || associatedData``

where ``associatedData`` is specific to the bundle it appears in.

In a vZSA transaction, every ``sighashInfo`` MUST be ``[0x00]`` (``sighashVersion`` 0, empty
``associatedData``), which selects the digest defined in `Signature Digest`_.


Transaction Format
==================

+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                       | Description                                                         |
+=============================+==============================+=================================================+=====================================================================+
| **Common Transaction Fields**                                                                                                                                                      |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``header``                    |``uint32``                                       |Contains:                                                            |
|                             |                              |                                                 |                                                                     |
|                             |                              |                                                 |* ``fOverwintered`` flag (bit 31, always set)                        |
|                             |                              |                                                 |* ``version`` (bits 30 .. 0) – transaction version, MUST be ``7``.   |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nVersionGroupId``           |``uint32``                                       |Version group ID, MUST be ``0x77777777``.                            |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nConsensusBranchId``        |``uint32``                                       |Consensus branch ID (nonzero).                                       |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``lock_time``                 |``uint32``                                       |Unix-epoch UTC time or block height, encoded as in Bitcoin.          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nExpiryHeight``             |``uint32``                                       |A block height in the range {1 .. 499999999} after which             |
|                             |                              |                                                 |the transaction will expire, or 0 to disable expiry. [#zip-0203]_    |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| **Transparent Transaction Fields**                                                                                                                                                 |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``tx_in_count``               |``compactSize``                                  |Number of transparent inputs in ``tx_in``.                           |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``tx_in``                     |``tx_in``                                        |Transparent inputs, encoded as in Bitcoin.                           |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``tx_out_count``              |``compactSize``                                  |Number of transparent outputs in ``tx_out``.                         |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``tx_out``                    |``tx_out``                                       |Transparent outputs, encoded as in Bitcoin.                          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vSighashInfo``              |``TransparentSighashInfo[tx_in_count]``          |Sighash info for each transparent input.                             |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| **Sapling Transaction Fields**                                                                                                                                                     |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nSpendsSapling``            |``compactSize``                                  |Number of Sapling Spend descriptions in ``vSpendsSapling``.          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 96 × ``nSpendsSapling``     |``vSpendsSapling``            |``SpendDescriptionV5[nSpendsSapling]``           |A sequence of Sapling Spend descriptions, encoded per                |
|                             |                              |                                                 |§7.3 ‘Spend Description Encoding and Consensus’                      |
|                             |                              |                                                 |[#protocol-spendencodingandconsensus]_ (unchanged from V5).          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nOutputsSapling``           |``compactSize``                                  |Number of Sapling Output Descriptions in ``vOutputsSapling``.        |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 756 × ``nOutputsSapling``   |``vOutputsSapling``           |``OutputDescriptionV5[nOutputsSapling]``         |A sequence of Sapling Output descriptions, encoded per               |
|                             |                              |                                                 |§7.4 ‘Output Description Encoding and Consensus’                     |
|                             |                              |                                                 |[#protocol-outputencodingandconsensus]_ (unchanged from V5).         |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``valueBalanceSapling``       |``int64``                                        |The net value of Sapling Spends minus Outputs                        |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``anchorSapling``             |``byte[32]``                                     |A root of the Sapling note commitment tree                           |
|                             |                              |                                                 |at some block height in the past.                                    |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 192 × ``nSpendsSapling``    |``vSpendProofsSapling``       |``byte[192 * nSpendsSapling]``                   |Encodings of the zk-SNARK proofs for each Sapling Spend.             |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vSpendAuthSigsSapling``     |``SaplingSignature[nSpendsSapling]``             |Authorizing signatures for each Sapling Spend.                       |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 192 × ``nOutputsSapling``   |``vOutputProofsSapling``      |``byte[192 * nOutputsSapling]``                  |Encodings of the zk-SNARK proofs for each Sapling Output.            |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``bindingSigSapling``         |``SaplingSignature``                             |A Sapling binding signature on the SIGHASH transaction hash.         |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| **Orchard Transaction Fields**                                                                                                                                                     |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nActionsOrchard``           |``compactSize``                                  |The number of Orchard Action descriptions in                         |
|                             |                              |                                                 |``vActionsOrchard``.                                                 |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 820 × ``nActionsOrchard``   |``vActionsOrchard``           |``OrchardAction[nActionsOrchard]``               |A sequence of Orchard Action descriptions, encoded per               |
|                             |                              |                                                 |§7.5 ‘Action Description Encoding and Consensus’                     |
|                             |                              |                                                 |[#protocol-actionencodingandconsensus]_.                             |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 1                           |``flagsOrchard``              |``byte``                                         |An 8-bit value representing a set of flags. Ordered from LSB to MSB: |
|                             |                              |                                                 | * ``enableSpends``                                                  |
|                             |                              |                                                 | * ``enableOutputs``                                                 |
|                             |                              |                                                 | * ``enableCrossAddress`` (new at NU6.3)                             |
|                             |                              |                                                 | * The remaining bits MUST be :math:`0\!`.                           |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``valueBalanceOrchard``       |``int64``                                        |The net value of Orchard spends minus outputs.                       |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``anchorOrchard``             |``byte[32]``                                     |A root of the Orchard note commitment tree at some block             |
|                             |                              |                                                 |height in the past.                                                  |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``sizeProofsOrchard``         |``compactSize``                                  |Length in bytes of ``proofsOrchard``. Value is                       |
|                             |                              |                                                 |:math:`2720 + 2272 \cdot \mathtt{nActionsOrchard}\!`.                |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| ``sizeProofsOrchard``       |``proofsOrchard``             |``byte[sizeProofsOrchard]``                      |Encoding of aggregated zk-SNARK proofs for Orchard Actions.          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vSpendAuthSigsOrchard``     |``OrchardSignature[nActionsOrchard]``            |Authorizing signatures for each Orchard Action.                      |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``bindingSigOrchard``         |``OrchardSignature``                             |An Orchard binding signature on the SIGHASH transaction hash.        |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| **IronwoodZSA Transaction Fields**                                                                                                                                                 |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nActionGroupsIronwood``     |``compactSize``                                  |The number of Action Group descriptions in ``vActionGroupsIronwood``.|
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vActionGroupsIronwood``     |``ActionGroupDescription[nActionGroupsIronwood]``|A sequence of Action Group descriptions, encoded as per the          |
|                             |                              |                                                 |`OrchardZSA Action Group Description`_.                              |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``valueBalanceIronwood``      |``int64``                                        |The net value of Ironwood spends minus outputs.                      |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``bindingSigIronwood``        |``OrchardSignature``                             |An OrchardZSA binding signature on the SIGHASH transaction hash.     |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| **ZSA Issuance Bundle Fields**                                                                                                                                                     |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``issuerLength``              |``compactSize``                                  |The length of the issuer identifier.                                 |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``issuer``                    |``byte[issuerLength]``                           |The issuer identifier as defined in [#zip-0227-issuer-identifier]_.  |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nIssueActions``             |``compactSize``                                  |The number of issuance actions in the bundle.                        |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vIssueActions``             |``IssueAction[nIssueActions]``                   |A sequence of issuance action descriptions.                          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``issueAuthSig``              |``IssueAuthSignature``                           |The issuer's signature on the SIGHASH transaction hash, encoded per  |
|                             |                              |                                                 |`Issuance Authorization Signature (IssueAuthSignature)`_.            |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+


* The fields ``valueBalanceSapling`` and ``bindingSigSapling`` are present if and only if
  $\mathtt{nSpendsSapling} + \mathtt{nOutputsSapling} > 0$. If ``valueBalanceSapling``
  is not present, then $\mathsf{v^{balanceSapling}}$ is defined to be $0$.

* The field ``anchorSapling`` is present if and only if $\mathtt{nSpendsSapling} > 0$.

* The elements of ``vSpendProofsSapling`` and ``vSpendAuthSigsSapling`` have a 1:1
  correspondence to the elements of ``vSpendsSapling`` and MUST be ordered such that the
  proof or signature at a given index corresponds to the ``SpendDescriptionV5`` at the
  same index.

* The elements of ``vOutputProofsSapling`` have a 1:1 correspondence to the elements of
  ``vOutputsSapling`` and MUST be ordered such that the proof at a given index corresponds
  to the ``OutputDescriptionV5`` at the same index.

* The fields ``flagsOrchard``, ``valueBalanceOrchard``, ``anchorOrchard``,
  ``sizeProofsOrchard``, ``proofsOrchard``, and ``bindingSigOrchard`` are present if and
  only if $\mathtt{nActionsOrchard} > 0$. If ``valueBalanceOrchard`` is not present,
  then $\mathsf{v^{balanceOrchard}}$ is defined to be $0$.

* The fields ``valueBalanceIronwood`` and ``bindingSigIronwood`` are present if and
  only if $\mathtt{nActionGroupsIronwood} > 0$. If ``valueBalanceIronwood`` is not present,
  then $\mathsf{v^{balanceIronwood}}$ is defined to be $0$.

* The field ``issueAuthSig`` is present if and only if $\mathtt{nIssueActions} > 0$. The fields
  ``issuerLength``, ``issuer``, ``nIssueActions``, and ``vIssueActions`` are always present. If
  $\mathtt{nIssueActions} = 0$ then ``issuerLength`` MUST be $0$ (so ``issuer`` and ``vIssueActions``
  are empty).

* The ``version`` and ``nVersionGroupId`` values are provisional and subject to change.

* $\mathtt{nActionGroupsIronwood}$ MUST be $0$ or $1$.

* ``enableCrossAddress`` in ``flagsOrchard`` MUST be $0$, as in version 6 [#zip-0229]_.

* At least one of these MUST hold: $\mathtt{tx\_in\_count} > 0$, $\mathtt{nSpendsSapling} > 0$, or
  ``enableSpends`` is set in ``flagsOrchard`` or in any ``flagsIronwood``. Likewise for outputs,
  with $\mathtt{tx\_out\_count}$, $\mathtt{nOutputsSapling}$ and ``enableOutputs``.

* In coinbase transactions, ``enableSpends`` and ``enableZSA`` MUST be $0$ in every ``flagsIronwood``.

The encodings of ``tx_in`` and ``tx_out`` are unchanged since Zcash launch [#protocol-txnencoding]_
[#zip-0225-transaction-format]_. The types introduced by this format are encoded as described below.

OrchardZSA Action Group Description
-----------------------------------

The OrchardZSA Action Group Description is encoded in a transaction as an instance of an ``ActionGroupDescription`` type:

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``nActionsIronwood``          |``compactSize``                                 |The number of Action descriptions in ``vActionsIronwood``.           |
|                             |                              |                                                |This MUST be greater than ``0`` and less than :math:`2^{16}\!`.      |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 852 × ``nActionsIronwood``  |``vActionsIronwood``          |``OrchardZSAAction[nActionsIronwood]``          |A sequence of OrchardZSA Action descriptions in the Action Group.    |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 1                           |``flagsIronwood``             |``byte``                                        |An 8-bit value representing a set of flags. Ordered from LSB to MSB: |
|                             |                              |                                                | * ``enableSpends``                                                  |
|                             |                              |                                                | * ``enableOutputs``                                                 |
|                             |                              |                                                | * ``enableCrossAddress``                                            |
|                             |                              |                                                | * ``enableZSA``                                                     |
|                             |                              |                                                | * The remaining bits MUST be :math:`0\!`.                           |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``anchorIronwood``            |``byte[32]``                                    |A root of the Ironwood note commitment tree at some block height in  |
|                             |                              |                                                |the past.                                                            |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nAGExpiryHeight``           |``uint32``                                      |A block height in the range {1 .. 499999999} after which any         |
|                             |                              |                                                |transaction including this Action Group cannot be mined, or 0 if     |
|                             |                              |                                                |this Action Group places no constraint on transaction expiry.        |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      | ``nAssetBurn``               | ``compactSize``                                |The number of Assets burnt.                                          |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 40 × ``nAssetBurn``         | ``vAssetBurn``               | ``AssetBurn[nAssetBurn]``                      |A sequence of Asset Burn descriptions, encoded per                   |
|                             |                              |                                                |`OrchardZSA Asset Burn Description`_.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``sizeProofsIronwood``        |``compactSize``                                 |Length in bytes of ``proofsIronwood``. This MUST have the value      |
|                             |                              |                                                |:math:`2848 + 2272 \cdot \mathtt{nActionsIronwood}\!`.               |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeProofsIronwood``      |``proofsIronwood``            |``byte[sizeProofsIronwood]``                    |The aggregated zk-SNARK proof for all Actions in the Action Group.   |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vSpendAuthSigsIronwood``    |``OrchardSignature[nActionsIronwood]``          |Authorizing signatures for each Action of the Action Group in a      |
|                             |                              |                                                |transaction.                                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

* The proofs aggregated in ``proofsIronwood``, and the elements of
  ``vSpendAuthSigsIronwood``, each have a 1:1 correspondence to the elements of
  ``vActionsIronwood`` and MUST be ordered such that the proof or signature at a given
  index corresponds to the ``OrchardZSAAction`` at the same index.
* At least one of ``enableSpends`` and ``enableOutputs`` MUST be set.
* ``nAGExpiryHeight`` MUST be $0$.

Rationale for nAGExpiryHeight
`````````````````````````````

The ``nAGExpiryHeight`` field is included for forward compatibility with Swaps over ZSAs, as
proposed in ZIP 228 [#zip-0228]_. The OrchardZSA protocol does not use an Action Group expiry, so
consensus fixes the field at ``0``, the no-expiry value also used for ``nExpiryHeight`` in ZIP 203
[#zip-0203]_.

Rationale for including Burn fields inside OrchardZSA Action Groups
```````````````````````````````````````````````````````````````````

The burn fields are part of the Action Group rather than of the transaction so that, when Action
Groups are generated by different parties and later bundled into one transaction, each party can
burn Assets in its own Action Group. Transaction-level burn fields would reserve burning to the
party that bundles the Action Groups.


OrchardZSA Action Description (``OrchardZSAAction``)
----------------------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| 32                          |``cv``                        |``byte[32]``                                    |A value commitment to the net value of the input note minus the      |
|                             |                              |                                                |output note.                                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``nullifier``                 |``byte[32]``                                    |The nullifier of the input note.                                     |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``rk``                        |``byte[32]``                                    |The randomized validating key for the element of                     |
|                             |                              |                                                |``vSpendAuthSigsIronwood`` corresponding to this Action.             |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``cmx``                       |``byte[32]``                                    |The :math:`x\!`-coordinate of the note commitment for the output     |
|                             |                              |                                                |note.                                                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``ephemeralKey``              |``byte[32]``                                    |An encoding of an ephemeral Pallas public key.                       |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 612                         |``encCiphertext``             |``byte[612]``                                   |The encrypted contents of the note plaintext.                        |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 80                          |``outCiphertext``             |``byte[80]``                                    |The encrypted contents of the byte string created by concatenation   |
|                             |                              |                                                |of the transmission key with the ephemeral secret key.               |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

These elements are encoded as in §7.5 [#protocol-actionencodingandconsensus]_.

OrchardZSA Asset Burn Description
----------------------------------

An OrchardZSA Asset Burn description is encoded in a transaction as an instance of an ``AssetBurn`` type:

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| 32                          |``assetBase``                 |``byte[32]``                                    |The encoding :math:`\mathsf{asset\_base}` of the Asset Base of the   |
|                             |                              |                                                |Asset being burnt.                                                   |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``valueBurn``                 |``uint64``                                      |The amount being burnt.                                              |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

The $\mathsf{assetBurn}$ set of ZIP 226 [#zip-0226-burn-mechanism]_ is the Action Group's
``vAssetBurn`` (empty if there is none); the consensus rules for its elements are specified there.

Transparent Sighash Information (``TransparentSighashInfo``)
------------------------------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of ``sighashInfo``.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |(see `Sighash versioning`_).                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Sapling Signature (``SaplingSignature``)
----------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of ``sighashInfo``.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |(see `Sighash versioning`_).                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 64                          |``signature``                 |``byte[64]``                                    |An encoding of a RedJubjub signature, which may be either a          |
|                             |                              |                                                |``spendAuthSig`` or ``bindingSig`` depending on context.             |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Orchard Signature (``OrchardSignature``)
----------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of ``sighashInfo``.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |(see `Sighash versioning`_).                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 64                          |``signature``                 |``byte[64]``                                    |An encoding of a RedPallas signature, which may be either a          |
|                             |                              |                                                |``spendAuthSig`` or ``bindingSig`` depending on context.             |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Issuance Authorization Signature (``IssueAuthSignature``)
---------------------------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of ``sighashInfo``.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |(see `Sighash versioning`_).                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``sizeSignature``             |``compactSize``                                 |The size in bytes of ``signature``.                                  |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``signature``                 |``byte[sizeSignature]``                         |The ``issueAuthSig`` encoding of ZIP 227                             |
|                             |                              |                                                |[#zip-0227-issuance-auth-sig]_: a ``0x00`` byte indicating a BIP 340 |
|                             |                              |                                                |signature, followed by the signature.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+


Issuance Action Description (``IssueAction``)
---------------------------------------------

An issuance action, ``IssueAction``, is the instance of issuing a specific Custom Asset, and contains the following fields:

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| 32                          |``assetDescHash``             |``byte[32]``                                    |A hash of the description of the Custom Asset.                       |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nNotes``                    |``compactSize``                                 |The number of notes in the Issuance Action.                          |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 115 × ``nNotes``            |``vNotes``                    |``IssueNoteDescription[nNotes]``                |A sequence of issue note descriptions within the Issuance Action.    |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 1                           |``flagsIssuance``             |``byte``                                        |An 8-bit value representing a set of flags. Ordered from LSB to MSB: |
|                             |                              |                                                |                                                                     |
|                             |                              |                                                |* :math:`\mathsf{finalize}`                                          |
|                             |                              |                                                |* The remaining bits MUST be :math:`0\!`.                            |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

The encoding of ``IssueNoteDescription`` is described below.
Note that we allow the number of notes (represented by ``nNotes``) to be zero.
This allows for issuers to create Issuance Actions to only finalize an issued Asset, without needing them to simultaneously issue more of that Asset.

Issue Note Description (``IssueNoteDescription``)
-------------------------------------------------

An issuance note description, ``IssueNoteDescription`` contains the following fields:

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| 43                          |``recipient``                 |``byte[43]``                                    |The encoding of a recipient's diversified payment address, as        |
|                             |                              |                                                |:math:`\mathsf{LEBS2OSP}_{88}(\mathsf{d})\|                          |
|                             |                              |                                                |\mathsf{LEBS2OSP}_{256}(\mathsf{repr}_{\mathbb{P}}                   |
|                             |                              |                                                |(\mathsf{pk}_\mathsf{d}))\!`, where :math:`\mathsf{d}` is the        |
|                             |                              |                                                |diversifier and :math:`\mathsf{pk_d}` is the diversified             |
|                             |                              |                                                |transmission key. **Non Normative Note**: This is the same as the    |
|                             |                              |                                                |encoding of an Orchard Raw Payment Address, as defined in            |
|                             |                              |                                                |§5.6.4.2 ‘Orchard Raw Payment Addresses’                             |
|                             |                              |                                                |[#protocol-orchardpaymentaddrencoding]_.                             |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``value``                     |``uint64``                                      |The amount being issued in this note.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``rho``                       |``byte[32]``                                    |This is defined and encoded in the same manner as for Orchard notes  |
|                             |                              |                                                |in §3.2.1 ‘Note Plaintexts and Memo Fields’                          |
|                             |                              |                                                |[#protocol-noteptconcept]_.                                          |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``rseed``                     |``byte[32]``                                    |The ``rseed`` field of the note, encoded as for Orchard notes in     |
|                             |                              |                                                |§3.2.1 ‘Note Plaintexts and Memo Fields’ [#protocol-noteptconcept]_. |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Note Plaintexts
---------------

OrchardZSA notes have an additional $\mathsf{AssetBase}$ field, included in the note plaintext as
its byte encoding $\mathsf{asset\_base} = \mathsf{LEBS2OSP}_{\ell_{\mathbb{P}}}(\mathsf{repr}_{\mathbb{P}}(\mathsf{AssetBase}))$
[#zip-0226-note-structure-and-commitment]_. An Orchard note is equivalent to an OrchardZSA note
with $\mathsf{AssetBase} = \mathcal{V}^{\mathsf{Orchard}}$ (as defined in § 5.4.8.3 ‘Homomorphic
Pedersen commitments (Sapling and Orchard)’ [#protocol-concretehomomorphiccommit]_); the two have
the same note commitment, so note plaintexts need not distinguish them.

An OrchardZSA note plaintext consists of

.. math:: (\mathsf{leadByte} : \mathbb{B}^{\mathbb{Y}}, \mathsf{d} : \mathbb{B}^{[\ell_{\mathsf{d}}]}, \mathsf{v} : \{0 .. 2^{\ell_{\mathsf{value}}} - 1\}, \mathsf{rseed} : \mathbb{B}^{\mathbb{Y}[32]}, \mathsf{asset\_base} : \mathbb{B}^{\mathbb{Y}[\ell_{\mathbb{P}}/8]}, \mathsf{memo} : \mathbb{B}^{\mathbb{Y}[512]})

Its encoding, updating § 5.5 ‘Encodings of Note Plaintexts and Memo Fields’
[#protocol-noteptencoding]_, is

.. math:: [\mathsf{leadByte}] \,||\, \mathsf{LEBS2OSP}_{88}(\mathsf{d}) \,||\, \mathsf{I2LEOSP}_{64}(\mathsf{v}) \,||\, \mathsf{rseed} \,||\, \mathsf{asset\_base} \,||\, \mathsf{memo}

where the encodings of $\mathsf{d}$, $\mathsf{v}$, and $\mathsf{rseed}$ are unchanged from § 5.5.

The $\mathsf{leadByte}$ of every note plaintext of an OrchardZSA Action Group output MUST be
$\mathtt{0x04}$, for all Assets including ZEC. This ZIP therefore changes
$\mathsf{allowedLeadBytes}^{\textit{IronwoodPool}}$ of ZIP 2005 [#zip-2005]_ to return
$\{ \mathtt{0x04} \}$ when $\mathsf{txVersion}$ is vZSA.

Rationale for requiring the lead byte to be 0x04
````````````````````````````````````````````````

ZIP 2005's lead byte $\mathtt{0x03}$ [#zip-2005]_ cannot be reused: these plaintexts add
$\mathsf{asset\_base}$, and for lead byte $\mathtt{0x04}$ the $\mathsf{rcm}$ derivation also commits
to it [#zip-0226]_. Using $\mathtt{0x04}$ for every Asset, including ZEC, keeps one plaintext layout
and one $\mathsf{rcm}$ derivation per lead byte.


TxId Digest
===========

For vZSA transactions, transaction identifiers (txids), authorizing-data commitments ("auth
digests"), and signature digests are computed as in ZIP 229 [#zip-0229]_, with a changed
**Ironwood component digest**, a new **Issuance digest**, and, in the auth digest only, a new
**sighash info digest**.

The overall structure of the txid digest is as follows::

    txid_digest_zsa *
    ├── header_digest
    ├── transparent_digest
    │   ├── prevouts_digest
    │   ├── sequence_digest
    │   └── outputs_digest
    ├── sapling_digest_v6
    │   ├── sapling_spends_digest_v6
    │   │   ├── sapling_spends_compact_digest
    │   │   └── sapling_spends_noncompact_digest_v6
    │   │       ├── cv
    │   │       └── rk
    │   ├── sapling_outputs_digest
    │   │   ├── sapling_outputs_compact_digest
    │   │   ├── sapling_outputs_memos_digest
    │   │   └── sapling_outputs_noncompact_digest
    │   └── valueBalance
    ├── orchard_digest_v6
    │   ├── orchard_actions_compact_digest
    │   ├── orchard_actions_memos_digest
    │   ├── orchard_actions_noncompact_digest
    │   ├── flagsOrchard
    │   └── valueBalanceOrchard
    ├── ironwood_digest_zsa *
    │   ├── ironwood_action_groups_digest_zsa *
    │   │   ├── ironwood_actions_compact_digest_zsa *
    │   │   ├── ironwood_actions_memos_digest_zsa *
    │   │   ├── ironwood_actions_noncompact_digest_zsa *
    │   │   ├── flagsIronwood
    │   │   ├── nAGExpiryHeight *
    │   │   └── ironwood_burn_digest *
    │   └── valueBalanceIronwood
    └── issuance_digest *
        ├── issuerLength *
        ├── issuer *
        └── issue_actions_digest *
            ├── assetDescHash *
            ├── issue_notes_digest *
            └── flagsIssuance *

Each node written as ``snake_case`` in this tree is a BLAKE2b-256 hash of its
children, initialized with a personalization string specific to that branch
of the tree. Nodes that are not themselves digests are written in ``camelCase``.
In the specification below, nodes of the tree are presented in depth-first order.

The nodes marked ``*`` are new or changed relative to ZIP 229 [#zip-0229]_; the digests among
them are defined below. All other nodes are as defined in ZIP 229.

txid_digest_zsa
---------------
A BLAKE2b-256 hash of the following values ::

   T.1: header_digest       (32-byte hash output)
   T.2: transparent_digest  (32-byte hash output)
   T.3: sapling_digest_v6   (32-byte hash output)
   T.4: orchard_digest_v6   (32-byte hash output)
   T.5: ironwood_digest_zsa (32-byte hash output)
   T.6: issuance_digest     (32-byte hash output)

The personalization field of this hash is set to::

  "ZcashTxHash_" || CONSENSUS_BRANCH_ID

``ZcashTxHash_`` has 1 underscore character.

As in ZIP 244 [#zip-0244]_, CONSENSUS_BRANCH_ID is the 4-byte little-endian encoding of
the consensus branch ID for the epoch of the block containing the transaction.

T.5: ironwood_digest_zsa
````````````````````````
If $\mathtt{nActionGroupsIronwood} > 0$, this digest is a BLAKE2b-256 hash of the following values::

    T.5a: ironwood_action_groups_digest_zsa  (32-byte hash output)
    T.5b: valueBalanceIronwood               (64-bit signed little-endian)

The personalization field of this hash is set to::

    "ZTxIdIronwdH_zsa"

If $\mathtt{nActionGroupsIronwood} = 0$, ``ironwood_digest_zsa`` is ::

    BLAKE2b-256("ZTxIdIronwdH_zsa", [])

T.5a: ironwood_action_groups_digest_zsa
'''''''''''''''''''''''''''''''''''''''

A BLAKE2b-256 hash of the subset of OrchardZSA Action Groups information for all OrchardZSA Action Groups belonging to the transaction.
For each Action Group, the following elements are included in the hash::

    T.5a.i   : ironwood_actions_compact_digest_zsa     (32-byte hash output)
    T.5a.ii  : ironwood_actions_memos_digest_zsa       (32-byte hash output)
    T.5a.iii : ironwood_actions_noncompact_digest_zsa  (32-byte hash output)
    T.5a.iv  : flagsIronwood                           (1 byte)
    T.5a.v   : nAGExpiryHeight                         (4 bytes)
    T.5a.vi  : ironwood_burn_digest                    (32-byte hash output)

The personalization field of this hash is set to::

    "ZTxIdIrn_AGH_zsa"

T.5a.i: ironwood_actions_compact_digest_zsa
...........................................

A BLAKE2b-256 hash of the subset of OrchardZSA Action information intended to be included in
an updated version of the ZIP-307 [#zip-0307]_ ``CompactBlock`` format for all OrchardZSA
Actions belonging to the Action Group. For each Action, the following elements are included
in the hash::

   T.5a.i.1 : nullifier            (field encoding bytes)
   T.5a.i.2 : cmx                  (field encoding bytes)
   T.5a.i.3 : ephemeralKey         (field encoding bytes)
   T.5a.i.4 : encCiphertext[..84]  (First 84 bytes of field encoding)

The personalization field of this hash is set to::

  "ZTxIdIrn_ACH_zsa"

The field encodings are specified in `OrchardZSA Action Description (OrchardZSAAction)`_.

T.5a.ii: ironwood_actions_memos_digest_zsa
..........................................

A BLAKE2b-256 hash of the subset of OrchardZSA Action information encrypting memo bytes
for all OrchardZSA Actions belonging to the Action Group. For each Action, the following
elements are included in the hash::

   T.5a.ii.1 : encCiphertext[84..596]  (contents of the encrypted memo field)

The personalization field of this hash is set to::

  "ZTxIdIrn_AMH_zsa"

The field encodings are specified in `OrchardZSA Action Description (OrchardZSAAction)`_.

Non-normative note: ZIP 229's ``ironwood_actions_memos_digest_v6`` [#zip-0229]_ hashes
``encCiphertext[52..564]``, the memo range of a note plaintext without $\mathsf{asset\_base}$.
Here the memo starts 32 bytes later, so reusing that node would leave ``encCiphertext[564..596]``
outside every digest.

T.5a.iii: ironwood_actions_noncompact_digest_zsa
................................................

A BLAKE2b-256 hash of the remaining subset of OrchardZSA Action information **not** intended
for inclusion in an updated version of the ZIP 307 [#zip-0307]_ ``CompactBlock``
format, for all OrchardZSA Actions belonging to the Action Group. For each Action,
the following elements are included in the hash::

   T.5a.iii.1 : cv                    (field encoding bytes)
   T.5a.iii.2 : rk                    (field encoding bytes)
   T.5a.iii.3 : encCiphertext[596..]  (post-memo suffix of field encoding)
   T.5a.iii.4 : outCiphertext         (field encoding bytes)

The personalization field of this hash is set to::

  "ZTxIdIrn_ANH_zsa"

The field encodings are specified in `OrchardZSA Action Description (OrchardZSAAction)`_.


T.5a.vi: ironwood_burn_digest
.............................

A BLAKE2b-256 hash of the data from the burn fields of the Action Group. For each tuple in
the Action Group's $\mathsf{assetBurn}$ set, the following elements are included in the hash::

    T.5a.vi.1 : assetBase    (field encoding bytes)
    T.5a.vi.2 : valueBurn    (64-bit unsigned little-endian)

The personalization field of this hash is set to::

    "ZTxIdIrnBurnHash"

In case the Action Group does not burn any Assets (i.e. its $\mathsf{assetBurn}$ set is
empty), the ``ironwood_burn_digest`` is::

    BLAKE2b-256("ZTxIdIrnBurnHash", [])

The field encodings are specified in `OrchardZSA Asset Burn Description`_.


T.6: issuance_digest
````````````````````
A BLAKE2b-256 hash of the following values ::

   T.6a: issuerLength            (field encoding bytes)
   T.6b: issuer                  (field encoding bytes)
   T.6c: issue_actions_digest    (32-byte hash output)

The personalization field of this hash is set to::

  "ZTxIdSAIssueHash"

If $\mathtt{nIssueActions} = 0$, ``issuance_digest`` is::

    BLAKE2b-256("ZTxIdSAIssueHash", [])

The field encodings are specified in `Transaction Format`_.

T.6c: issue_actions_digest
''''''''''''''''''''''''''
A BLAKE2b-256 hash of Issue Action information for all Issuance Actions belonging to the transaction. For each Action, the following elements are included in the hash::

   T.6c.i  : assetDescHash           (field encoding bytes)
   T.6c.ii : issue_notes_digest      (32-byte hash output)
   T.6c.iii: flagsIssuance           (1 byte)

The personalization field of this hash is set to::

  "ZTxIdIssuActHash"

The field encodings are specified in `Issuance Action Description (IssueAction)`_.

T.6c.ii: issue_notes_digest
...........................
A BLAKE2b-256 hash of Note information for all Issue Note Descriptions belonging to the Issuance Action. For each Note, the following elements are included in the hash::

   T.6c.ii.1: recipient                   (field encoding bytes)
   T.6c.ii.2: value                       (field encoding bytes)
   T.6c.ii.3: rho                         (field encoding bytes)
   T.6c.ii.4: rseed                       (field encoding bytes)

The personalization field of this hash is set to::

  "ZTxIdIAcNoteHash"

In case the Issuance Action has no Issue Notes (for example, an Issuance Action that only
finalizes an Asset), ``issue_notes_digest`` is::

    BLAKE2b-256("ZTxIdIAcNoteHash", [])

The field encodings are specified in `Issue Note Description (IssueNoteDescription)`_.

Signature Digest
================

The vZSA signature digest is ``txid_digest_zsa`` with ``transparent_digest`` (T.2) replaced by
``transparent_sig_digest`` as defined in ZIP 244 [#zip-0244-sigdigest]_. For a signature over a
Sapling Spend, an Orchard-pool or Ironwood-pool Action, or the Issuance Bundle,
``transparent_sig_digest`` is computed as ZIP 244 specifies for a Sapling Spend or Orchard Action.

Authorizing Data Commitment
===========================

The auth digest structure is as follows::

    auth_digest_zsa (*)
    ├── transparent_auth_digest
    ├── sapling_auth_digest_v6
    │   ├── vSpendProofsSapling
    │   ├── vSpendAuthSigsSapling
    │   ├── vOutputProofsSapling
    │   ├── bindingSigSapling
    │   └── anchorSapling
    ├── orchard_auth_digest_v6
    │   ├── proofsOrchard
    │   ├── vSpendAuthSigsOrchard
    │   ├── bindingSigOrchard
    │   └── anchorOrchard
    ├── ironwood_auth_digest_zsa (*)
    │   ├── ironwood_action_groups_auth_digest (+)
    │   │   ├── proofsIronwood
    │   │   ├── ironwood_spend_auth_sigs_auth_digest (+)
    │   │   │   └── vSpendAuthSigsIronwood
    │   │   └── anchorIronwood
    │   └── bindingSigIronwood
    ├── issuance_auth_digest (+)
    │   └── issueAuthSig
    └── sighash_info_digest (+)
        ├── vSighashInfo
        ├── sighashInfo of each Sapling signature
        └── sighashInfo of each Orchard signature

where (*) indicates an updated definition, and (+) a new definition, relative to ZIP 229 [#zip-0229]_.

``transparent_auth_digest``, ``sapling_auth_digest_v6`` and ``orchard_auth_digest_v6`` are as in
ZIP 229 [#zip-0229]_, with their Sapling and Orchard signatures hashed as the 64-byte ``signature``
field only. The ``sighashInfo`` values they omit are committed in
``sighash_info_digest`` instead, so what these nodes hash, and their personalizations, are unchanged.

The updated and new nodes are defined below.

auth_digest_zsa
---------------

A BLAKE2b-256 hash of the following values ::

   A.1: transparent_auth_digest   (32-byte hash output)
   A.2: sapling_auth_digest_v6    (32-byte hash output)
   A.3: orchard_auth_digest_v6    (32-byte hash output)
   A.4: ironwood_auth_digest_zsa  (32-byte hash output)
   A.5: issuance_auth_digest      (32-byte hash output)
   A.6: sighash_info_digest       (32-byte hash output)

The personalization field of this hash is set to::

  "ZTxAuthHash_" || CONSENSUS_BRANCH_ID

A.4: ironwood_auth_digest_zsa
`````````````````````````````

If $\mathtt{nActionGroupsIronwood} > 0$, this is a BLAKE2b-256 hash of the following values::

    A.4a: ironwood_action_groups_auth_digest    (32-byte hash output)
    A.4b: bindingSigIronwood                    (field encoding bytes)

The personalization field of this hash is::

    "ZTxAuthIrnH__zsa" (2 underscore characters)

If $\mathtt{nActionGroupsIronwood} = 0$, ``ironwood_auth_digest_zsa`` is::

    BLAKE2b-256("ZTxAuthIrnH__zsa", [])

The field encodings are specified in `Transaction Format`_.

A.4a: ironwood_action_groups_auth_digest
''''''''''''''''''''''''''''''''''''''''

A BLAKE2b-256 hash of the subset of OrchardZSA Action Group information for all OrchardZSA Action Groups belonging to the transaction.
For each OrchardZSA Action Group, the following elements are included in the hash::

    A.4a.i  : proofsIronwood                               (field encoding bytes)
    A.4a.ii : ironwood_spend_auth_sigs_auth_digest         (32-byte hash output)
    A.4a.iii: anchorIronwood                               (field encoding bytes)

The personalization field of this hash is set to::

    "ZTxAuthIrnAGHash"

The field encodings are specified in `Transaction Format`_.

A.4a.ii: ironwood_spend_auth_sigs_auth_digest
.............................................

This is a BLAKE2b-256 hash of the concatenation of ``OrchardSignature`` encodings for the spend authorization signature of each OrchardZSA Action in the OrchardZSA Action Group (i.e. the contents of the ``vSpendAuthSigsIronwood`` field of the ``ActionGroupDescription``)::

    A.4a.ii.1: vSpendAuthSigsIronwood    (field encoding bytes)

The personalization field of this hash is set to::

    "ZTxAuthIrnSASH__" (2 underscore characters)

The encoding of ``OrchardSignature`` is specified in `Orchard Signature (OrchardSignature)`_.

A.5: issuance_auth_digest
`````````````````````````

If $\mathtt{nIssueActions} > 0$, this is a BLAKE2b-256 hash of the field encoding of the
``issueAuthSig`` field of the transaction::

   A.5a: issueAuthSig            (field encoding bytes)

The personalization field of this hash is set to::

  "ZTxAuthIssH__zsa" (2 underscore characters)

If $\mathtt{nIssueActions} = 0$, ``issuance_auth_digest`` is ::

  BLAKE2b-256("ZTxAuthIssH__zsa", [])

The field encodings are specified in `Issuance Authorization Signature (IssueAuthSignature)`_.

A.6: sighash_info_digest
````````````````````````

A BLAKE2b-256 hash of the ``sizeSighashInfo`` and ``sighashInfo`` fields (field encoding bytes) of
each of the following, in order::

    A.6a: each element of vSighashInfo
    A.6b: each element of vSpendAuthSigsSapling, then bindingSigSapling if present
    A.6c: each element of vSpendAuthSigsOrchard, then bindingSigOrchard if present

The personalization field of this hash is set to::

    "ZTxAuthSInfH_zsa"

If none of these are present, ``sighash_info_digest`` is::

    BLAKE2b-256("ZTxAuthSInfH_zsa", [])

The field encodings are specified in `Transaction Format`_.

Block Commitments
=================

The ``hashAuthDataRoot`` component of ``hashBlockCommitments`` [#zip-0244]_ incorporates
``auth_digest_zsa`` for each vZSA transaction, with no further structural change.


==========
Deployment
==========

TODO


========================
Reference implementation
========================

TODO


==========
References
==========

.. [#BCP14] `Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words" <https://www.rfc-editor.org/info/bcp14>`_
.. [#protocol] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3] or later <protocol/protocol.pdf>`_
.. [#protocol-noteptconcept] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 3.2.1: Note Plaintexts and Memo Fields <protocol/protocol.pdf#noteptconcept>`_
.. [#protocol-orchardpaymentaddrencoding] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.6.4.2: Orchard Raw Payment Addresses <protocol/protocol.pdf#orchardpaymentaddrencoding>`_
.. [#protocol-spendencodingandconsensus] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 7.3: Spend Description Encoding and Consensus <protocol/protocol.pdf#spendencodingandconsensus>`_
.. [#protocol-outputencodingandconsensus] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 7.4: Output Description Encoding and Consensus <protocol/protocol.pdf#outputencodingandconsensus>`_
.. [#protocol-actionencodingandconsensus] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 7.5: Action Description Encoding and Consensus <protocol/protocol.pdf#actionencodingandconsensus>`_
.. [#protocol-concretehomomorphiccommit] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.4.8.3: Homomorphic Pedersen commitments (Sapling and Orchard) <protocol/protocol.pdf#concretehomomorphiccommit>`_
.. [#protocol-noteptencoding] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.5: Encodings of Note Plaintexts and Memo Fields <protocol/protocol.pdf#noteptencoding>`_
.. [#protocol-txnencoding] `Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 7.1: Transaction Encoding and Consensus <protocol/protocol.pdf#txnencoding>`_
.. [#zip-0203] `ZIP 203: Transaction Expiry <zip-0203.rst>`_
.. [#zip-0225-transaction-format] `ZIP 225: Version 5 Transaction Format. Specification: Transaction Format <zip-0225.rst#transaction-format>`_
.. [#zip-0226] `ZIP 226: Transfer and Burn of Zcash Shielded Assets <zip-0226.rst>`_
.. [#zip-0226-burn-mechanism] `ZIP 226: Transfer and Burn of Zcash Shielded Assets - Burn Mechanism <zip-0226.rst#burn-mechanism>`_
.. [#zip-0226-enablezsa] `ZIP 226: Transfer and Burn of Zcash Shielded Assets — The enableZSA Flag <zip-0226.rst#the-mathsf-enablezsa-flag>`_
.. [#zip-0226-note-structure-and-commitment] `ZIP 226: Transfer and Burn of Zcash Shielded Assets — Note Structure and Commitment <zip-0226.rst#note-structure-and-commitment>`_
.. [#zip-0226-privacy-implications] `ZIP 226: Transfer and Burn of Zcash Shielded Assets — Privacy Implications <zip-0226.rst#privacy-implications>`_
.. [#zip-0227] `ZIP 227: Issuance of Zcash Shielded Assets <zip-0227.rst>`_
.. [#zip-0227-issuance-auth-sig] `ZIP 227: Issuance of Zcash Shielded Assets — Issuance Authorization Signature Scheme <zip-0227.rst#issuance-authorization-signature-scheme>`_
.. [#zip-0227-issuer-identifier] `ZIP 227: Issuance of Zcash Shielded Assets — Issuer Identifier <zip-0227.rst#issuer-identifier>`_
.. [#zip-0228] `ZIP 228: Asset Swaps for Zcash Shielded Assets <zip-0228.rst>`_
.. [#zip-0229] `ZIP 229: Version 6 Transaction Format <zip-0229.md>`_
.. [#zip-0244] `ZIP 244: Transaction Identifier Non-Malleability <zip-0244.rst>`_
.. [#zip-0244-sigdigest] `ZIP 244: Transaction Identifier Non-Malleability — Signature Digest <zip-0244.rst#signature-digest>`_
.. [#zip-0248] `ZIP 248: Extensible Transaction Format <zip-0248.rst>`_
.. [#zip-0307] `ZIP 307: Light Client Protocol for Payment Detection <zip-0307.rst>`_
.. [#zip-2005] `ZIP 2005: Ironwood Quantum Recoverability <zip-2005.md>`_
