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

The key words "MUST", "SHOULD", and "MAY" in this document are to be interpreted as
described in BCP 14 [#BCP14]_ when, and only when, they appear in all capitals.

The terms "Zcash Shielded Asset", "OrchardZSA", "ZSA" and "Asset" in this document are to be interpreted as described in ZIP 226 [#zip-0226]_.

The term "Issuance" and "Issuance Action" in this document are to be interpreted as described in ZIP 227 [#zip-0227]_.

The character § is used when referring to sections of the Zcash Protocol Specification
[#protocol]_.


========
Abstract
========

This proposal defines a new Zcash peer-to-peer transaction format, which supports the
changes for the inclusion of Zcash Shielded Assets [#zip-0226]_,[#zip-0227]_ in a future network upgrade.
It follows the same design pattern as the v6 transaction format [#zip-0229]_.

This proposal also defines the new concept of "sighash algorithm versioning":
where previously each transaction version had a single associated sighash
algorithm, going forward it will be possible for signers to use any sighash
algorithm within the closed set specified for a given transaction version (and
made available in consensus via network upgrades).

This ZIP defines the sighash algorithms associated with this new transaction format.


==========
Motivation
==========

The OrchardZSA protocol requires serialized data elements that are distinct from
any previous Zcash transaction. Since ZIP 244 was activated in NU5, the
v5 and later serialized transaction formats are not consensus-critical.
Thus, this ZIP defines a format that can easily accommodate future extensions,
where elements or a given pool are kept separate.


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

More general forms of extensibility, such as definining a key/value format that
allows for parsers that are unaware of some components, are not required.


=============
Specification
=============

All fields in this specification are encoded as little-endian.

The Zcash transaction format for transaction that include ZSAs is as follows:

------------------
Sighash versioning
------------------

Rough summary:

- Sighash versions are numbered starting from 0 for each tx version.
- v0 is by convention the "commit to all effecting data" sighash algorithm. Other
  versions can commit to whatever makes sense for desired functionality within
  that tx version.
- Consensus rules choose the digest algorithm for each signer based on
  ``sighashVersion``.

The sighash version is encoded as a single byte alongside any associated data
that the sighash algorithm version requires (for deterministically computing the
digest):

``sighashInfo = [sighashVersion] || associatedData``

where ``associatedData`` is specific to the bundle it appears in.

The following sighash algorithm versions are defined for v6 transactions:

+--------------------+-------------------------------------------------------+
|                    |:math:`\hspace{4.5em}` ``associatedData``              |
+--------------------+-------------+-------------+-------------+-------------+
| ``sighashVersion`` | Transparent |   Sapling   |   Orchard   |  Issuance   |
+--------------------+-------------+-------------+-------------+-------------+
| 0                  |``[]``       |``[]``       |``[]``       |``[]``       |
+--------------------+-------------+-------------+-------------+-------------+


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
|                             |                              |                                                 |* ``version`` (bits 30 .. 0) – transaction version.                  |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nVersionGroupId``           |``uint32``                                       |Version group ID (nonzero).                                          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nConsensusBranchId``        |``uint32``                                       |Consensus branch ID (nonzero).                                       |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``lock_time``                 |``uint32``                                       |Unix-epoch UTC time or block height, encoded as in Bitcoin.          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 4                           |``nExpiryHeight``             |``uint32``                                       |A block height in the range {1 .. 499999999} after which             |
|                             |                              |                                                 |the transaction will expire, or 0 to disable expiry. [#zip-0203]_    |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``fee``                       |``uint64``                                       |The fee to be paid by this transaction, in zatoshis.                 |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``zip233Amount``              |``uint64``                                       |The value to be removed from circulation in this transaction, in     |
|                             |                              |                                                 |zatoshis. [#zip-0233]_                                               |
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
|                             |                              |                                                 |§7.3 ‘Spend Description Encoding and Consensus’ (unchanged from V5). |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``nOutputsSapling``           |``compactSize``                                  |Number of Sapling Output Decriptions in ``vOutputsSapling``.         |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
| 276 × ``nOutputsSapling``   |``vOutputsSapling``           |``OutputDescriptionV6[nOutputsSapling]``         |A sequence of Sapling Output descriptions, encoded per               |
|                             |                              |                                                 |§7.4 ‘Output Description Encoding and Consensus’.                    |
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
|``varies``                   |``nActionsOrchard``           |``compactSize``                                  |The number of Orchard Action descriptions in                         |
|                             |                              |                                                 |``vActionsOrchard``.                                                 |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``820 * nActionsOrchard``    |``vActionsOrchard``           |``OrchardAction[nActionsOrchard]``               |A sequence of Orchard Action descriptions, encoded per               |
|                             |                              |                                                 |§7.5 ‘Action Description Encoding and Consensus’.                    |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``1``                        |``flagsOrchard``              |``byte``                                         |An 8-bit value representing a set of flags. Ordered from LSB to MSB: |
|                             |                              |                                                 | * ``enableSpends``                                                  |
|                             |                              |                                                 | * ``enableOutputs``                                                 |
|                             |                              |                                                 | * ``enableCrossAddress`` (new at NU6.3)                             |
|                             |                              |                                                 | * The remaining bits are set to :math:`0\!`.                        |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``8``                        |``valueBalanceOrchard``       |``int64``                                        |The net value of Orchard spends minus outputs.                       |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``32``                       |``anchorOrchard``             |``byte[32]``                                     |A root of the Orchard note commitment tree at some block             |
|                             |                              |                                                 |height in the past.                                                  |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``sizeProofsOrchard``         |``compactSize``                                  |Length in bytes of ``proofsOrchard``. Value is                       |
|                             |                              |                                                 |:math:`2720 + 2272 \cdot \mathtt{nActionsOrchard}\!`.                |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``sizeProofsOrchard``        |``proofsOrchard``             |``byte[sizeProofsOrchard]``                      |Encoding of aggregated zk-SNARK proofs for Orchard Actions.          |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``64 * nActionsOrchard``     |``vSpendAuthSigsOrchard``     |``byte[64 * nActionsOrchard]``                   |Authorizing signatures for each Orchard Action.                      |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+
|``64``                       |``bindingSigOrchard``         |``OrchardSignature``                             |An Orchard binding signature on the SIGHASH transaction hash.        |
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
| varies                      |``issueAuthSig``              |``IssueAuthSignature``                           |The signature of the transaction SIGHASH, signed by the issuer,      |
|                             |                              |                                                 |validated as in Issuance Authorization Signature Scheme              |
|                             |                              |                                                 |[#zip-0227-issuance-auth-sig]_.                                      |
+-----------------------------+------------------------------+-------------------------------------------------+---------------------------------------------------------------------+


* The fields ``valueBalanceSapling`` and ``bindingSigSapling`` are present if and only if
  $\mathtt{nSpendsSapling} + \mathtt{nOutputsSapling} > 0$. If ``valueBalanceSapling``
  is not present, then $\mathsf{v^{balanceSapling}}$ is defined to be $0$.

* The field ``anchorSapling`` is present if and only if $\mathtt{nSpendsSapling} > 0$.

* The elements of ``vSpendProofsSapling`` and ``vSpendAuthSigsSapling`` have a 1:1
  correspondence to the elements of ``vSpendsSapling`` and MUST be ordered such that the
  proof or signature at a given index corresponds to the ``SpendDescriptionV6`` at the
  same index.

* The elements of ``vOutputProofsSapling`` have a 1:1 correspondence to the elements of
  ``vOutputsSapling`` and MUST be ordered such that the proof at a given index corresponds
  to the ``OutputDescriptionV6`` at the same index.

* The fields ``flagsOrchard``, ``valueBalanceOrchard``, ``anchorOrchard``,
  ``sizeProofsOrchard``, ``proofsOrchard``, and ``bindingSigOrchard`` are present if and
  only if $\mathtt{nActionsOrchard} > 0$. If ``valueBalanceOrchard`` is not present,
  then $\mathsf{v^{balanceOrchard}}$ is defined to be $0$.

* The fields ``valueBalanceIronwood`` and ``bindingSigIronwood`` are present if and
  only if $\mathtt{nActionGroupsIronwood} > 0$. If ``valueBalanceIronwood`` is not present,
  then $\mathsf{v^{balanceIronwood}}$ is defined to be $0$.

* The fields ``issueAuthSigLength`` and ``issueAuthSig`` are present if and only if $\mathtt{nIssueActions} > 0$.
  The fields ``issuerLength``, ``issuer``, ``nIssueActions``, and ``vIssueActions`` are always present.
  If $\mathtt{nIssueActions} = 0$ then ``issuerLength`` MUST be set to $0$ (``issuer`` and ``vIssueActions`` will be empty in this case).

* For coinbase transactions, the ``enableSpends`` and ``enableZSAs`` bits MUST be set to $0$.

The encodings of ``tx_in``, and ``tx_out`` are as in prior transaction versions (i.e.
unchanged since Zcash launch). [#protocol-txnencoding]_ [#zip-0225-transaction-format]_

The encodings of ``SpendDescriptionV6``, ``OutputDescriptionV6``, ``ActionGroupDescription``,
``AssetBurn`` and ``IssueAction`` are described below.

The encoding of Sapling Spends and Outputs has changed relative to prior versions in order
to better separate data that describe the effects of the transaction from the proofs of and
commitments to those effects, and for symmetry with this separation in the Orchard-related parts
of the transaction format.

Sapling Output Description (``OutputDescriptionV6``)
----------------------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| 32                          |``cv``                        |``byte[32]``                                    |A value commitment to the net value of the output note.              |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``cmu``                       |``byte[32]``                                    |The :math:`u\!`-coordinate of the note commitment for the output     |
|                             |                              |                                                |note.                                                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``ephemeralKey``              |``byte[32]``                                    |An encoding of an ephemeral Jubjub public key.                       |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 100                         |``encCiphertext``             |``byte[100]``                                   |The encrypted contents of the note plaintext.                        |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 80                          |``outCiphertext``             |``byte[80]``                                    |The encrypted contents of the byte string created by concatenation   |
|                             |                              |                                                |of the transmission key with the ephemeral secret key.               |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

The encodings of each of these elements are defined in §7.4 ‘Output Description Encoding and Consensus’ [#protocol-outputdesc]_.

OrchardZSA Action Group Description
-----------------------------------

The OrchardZSA Action Group Description is encoded in a transaction as an instance of an ``ActionGroupDescription`` type:

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``nActionsIronwood``          |``compactSize``                                 |The number of Action descriptions in ``vActionsIronwood``.           |
|                             |                              |                                                |This MUST have a value strictly greater than ``0``.                  |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 372 × ``nActionsOrchard``   |``vActionsIronwood``          |``OrchardZSAAction[nActionsIronwood]``          |A sequence of OrchardZSA Action descriptions in the Action Group.    |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
|``1``                        |``flagsIronwood``             |``byte``                                        |An 8-bit value representing a set of flags. Ordered from LSB to MSB: |
|                             |                              |                                                | * ``enableSpends``                                                  |
|                             |                              |                                                | * ``enableOutputs``                                                 |
|                             |                              |                                                | * ``enableCrossAddress``                                            |
|                             |                              |                                                | * ``enableZSAs``                                                    |
|                             |                              |                                                | * The remaining bits are set to :math:`0\!`.                        |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``anchorIronwood``            |``byte[32]``                                    |A root of the **Ironwood** note commitment tree at some block height |
|                             |                              |                                                |in the past.                                                         |
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
| varies                      |``sizeProofsIronwood``        |``compactSize``                                 |As defined in §7.1 ‘Transaction Encoding and Consensus’              |
|                             |                              |                                                |[#protocol-txnencoding]_.                                            |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeProofsIronwood``      |``proofsIronwood``            |``byte[sizeProofsIronwood]``                    |The aggregated zk-SNARK proof for all Actions in the Action Group.   |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``vSpendAuthSigsIronwood``    |``OrchardSignature[nActionsIronwood]``          |Authorizing signatures for each Action of the Action Group in a      |
|                             |                              |                                                |transaction.                                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

The encodings of ``OrchardZSAAction`` and ``AssetBurn`` are described below.

* The proofs aggregated in ``proofsIronwood``, and the elements of
  ``vSpendAuthSigsIronwood``, each have a 1:1 correspondence to the elements of
  ``vActionsIronwood`` and MUST be ordered such that the proof or signature at a given
  index corresponds to the ``OrchardZSAAction`` at the same index.
* If the value of ``nAGExpiryHeight`` is nonzero, it MUST be consistent with
  the ``nExpiryHeight`` of the overall transaction. 
* In NU7, ``nExpiryHeight`` MUST be set to ``0``; this restriction is expected
  to be lifted in a future network upgrade.

Rationale for nAGExpiryHeight
`````````````````````````````

We introduce the ``nAGExpiryHeight`` field in this transaction format in order to be forward compatible with Swaps over ZSAs, as proposed in ZIP 228 [#zip-0228]_.
For the OrchardZSA protocol, which does not make use of an additional expiry height for transactions, we set the value of ``nAGExpiryHeight`` to be ``0`` by consensus.
This serves as a default value to represent the situation where there is no expiry, analogous to the convention adopted for ``nExpiryHeight`` in ZIP 203 [#zip-0203]_.

Rationale for including Burn fields inside OrchardZSA Action Groups
```````````````````````````````````````````````````````````````````

Note that this transaction format includes the burn fields of the transaction inside the OrchardZSA Action Group rather than at the transaction level.
This is a design choice that considers the future scenario where Action Groups may be generated by different parties before being bundled together into a transaction.
In such a scenario, the individual parties can burn Assets of their choice in their corresponding Action Groups. 
Maintaining the burn fields at the transaction level would provide the ability to burn Assets only to the party performing the bundling of the Action Groups.


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
|                             |                              |                                                |``spendAuthSigsOrchard`` corresponding to this Action.               |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``cmx``                       |``byte[32]``                                    |The :math:`x\!`-coordinate of the note commitment for the output     |
|                             |                              |                                                |note.                                                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``ephemeralKey``              |``byte[32]``                                    |An encoding of an ephemeral Pallas public key.                       |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 132                         |``encCiphertext``             |``byte[132]``                                   |The encrypted contents of the note plaintext.                        |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 80                          |``outCiphertext``             |``byte[80]``                                    |The encrypted contents of the byte string created by concatenation   |
|                             |                              |                                                |of the transmission key with the ephemeral secret key.               |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

The encodings of each of these elements are defined in §7.5 ‘Action Description Encoding and Consensus’ [#protocol-actiondesc]_.

OrchardZSA Asset Burn Description
----------------------------------

An OrchardZSA Asset Burn description is encoded in a transaction as an instance of an ``AssetBurn`` type:

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| 32                          |``AssetBase``                 |``byte[32]``                                    |For the OrchardZSA protocol, this is the encoding of the Asset Base  |
|                             |                              |                                                |:math:`\mathsf{AssetBase^{Orchard}}\!`.                              |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``valueBurn``                 |``uint64``                                      |The amount being burnt. The value is checked by consensus to be      |
|                             |                              |                                                |non-zero, and less than :math:`\mathsf{MAX\_BURN\_VALUE}`            |
|                             |                              |                                                |[#zip-0226-burn-mechanism]_.                                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

The encodings of each of these elements are defined in ZIP 226 [#zip-0226-burn-mechanism]_.

Transparent Sighash Information (``TransparentSighashInfo``)
------------------------------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of `sighashInfo`.                                  |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |[#zip-0246-sighash-info]_.                                           |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Sapling Signature (``SaplingSignature``)
----------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of `sighashInfo`.                                  |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |[#zip-0246-sighash-info]_.                                           |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 64                          |``signature``                 |``byte[64]``                                    |An encoding of a RedJubjub signature, which may be either a          |
|                             |                              |                                                |``spendAuthSig`` or ``bindingSig`` depending on context.             |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Orchard Signature (``OrchardSignature``)
----------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of `sighashInfo`.                                  |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |[#zip-0246-sighash-info]_.                                           |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 64                          |``signature``                 |``byte[64]``                                    |An encoding of a RedPallas signature, which may be either a          |
|                             |                              |                                                |``spendAuthSig`` or ``bindingSig`` depending on context.             |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Issuance Authorization Signature (``IssueAuthSignature``)
---------------------------------------------------------

+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                         | Data Type                                      | Description                                                         |
+=============================+==============================+================================================+=====================================================================+
| varies                      |``sizeSighashInfo``           |``compactSize``                                 |The size in bytes of `sighashInfo`.                                  |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| ``sizeSighashInfo``         |``sighashInfo``               |``byte[sizeSighashInfo]``                       |The sighash version and associated information                       |
|                             |                              |                                                |[#zip-0246-sighash-info]_.                                           |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``sizeSignature``             |``compactSize``                                 |The size in bytes of `signature`.                                    |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| varies                      |``signature``                 |``byte[sizeSignature]``                         |The signature of the transaction SIGHASH, signed by the issuer,      |
|                             |                              |                                                |preceded by a ``0x00`` byte indicating a BIP 340 signature. It is    |
|                             |                              |                                                |validated as specified in [#zip-0227-issuance-auth-sig]_.            |
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
|                             |                              |                                                |* The remaining bits are set to :math:`0\!`.                         |
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
|                             |                              |                                                |§5.6.4.2 ‘Orchard Raw Payment Addresses’.                            |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 8                           |``value``                     |``uint64``                                      |The amount being issued in this note.                                |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``rho``                       |``byte[32]``                                    |This is defined and encoded in the same manner as for Orchard notes  |
|                             |                              |                                                |in §3.2.1 ‘Note Plaintexts and Memo Fields’.                         |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+
| 32                          |``rseed``                     |``byte[32]``                                    |The ``rseed`` field of the note, encoded as for Orchard notes in     |
|                             |                              |                                                |§3.2.1 ‘Note Plaintexts and Memo Fields’.                            |
+-----------------------------+------------------------------+------------------------------------------------+---------------------------------------------------------------------+

Note Plaintexts
---------------

New note plaintext formats using lead byte {{LEADBYTE}} are introduced for
Sapling and Orchard, in order to support memo bundles and (for Orchard) ZSAs and
quantum recoverability.

The $\mathsf{leadByte}$ MUST be {{LEADBYTE}} for all note plaintexts in transactions including ZSAs.


Orchard Note Plaintext
``````````````````````

OrchardZSA notes have an additional $\mathsf{AssetBase}$ field, which needs to be included
(as its byte sequence encoding $\mathsf{asset\_base}$) in an OrchardZSA note plaintext.

Let $\mathcal{V}^{\mathsf{Orchard}}$ be as defined in
§ 5.4.8.3 ‘Homomorphic Pedersen commitments (Sapling and Orchard)’
[#protocol-concretehomomorphiccommit]_.

An Orchard note is essentially equivalent to an OrchardZSA note with
$\mathsf{AssetBase} = \mathcal{V}^{\mathsf{Orchard}}$; in particular, they have
the same note commitment (as defined in [#zip-0226-note-structure-and-commitment]_),
and need not be distinguished in note plaintexts.

Each new OrchardZSA note plaintext consists of

.. math:: (\mathsf{leadByte} : \mathbb{B}^{\mathbb{Y}}, \mathsf{d} : \mathbb{B}^{[\ell_{\mathsf{d}}]}, \mathsf{v} : \{0 .. 2^{\ell_{\mathsf{value}}} - 1\}, \mathsf{rseed} : \mathbb{B}^{\mathbb{Y}[32]}, \mathsf{asset\_base} : \mathbb{B}^{\mathbb{Y}[\ell_{\mathbb{P}}/8]}, \mathsf{K^{memo}} : \mathbb{B}^{\mathbb{Y}[32]})

where (consistent with [#zip-0226-note-structure-and-commitment]_) $\mathsf{asset\_base}$
is $\mathsf{LEBS2OSP}_{\ell_{\mathbb{P}}}(\mathsf{repr}_{\mathbb{P}}(\mathsf{AssetBase}))$.

The encoding of a new OrchardZSA note plaintext in
§ 5.5 ‘Encodings of Note Plaintexts and Memo Fields’ [#protocol-noteptencoding]_
is similarly updated and is now defined as:

.. math:: [\mathsf{leadByte}] \,||\, \mathsf{LEBS2OSP}_{88}(\mathsf{d}) \,||\, \mathsf{I2LEOSP}_{64}(\mathsf{v}) \,||\, \mathsf{rseed} \,||\, \mathsf{asset\_base} \,||\, \mathsf{memo}

The encodings of $\mathsf{d}$, $\mathsf{v}$, and $\mathsf{rseed}$ remain unchanged from the previous specification in §5.5 ‘Encodings of Note Plaintexts and Memo Fields’ of the Zcash Protocol Specification [#protocol-noteptencoding]_.

Non-normative note: The *use* of the $\mathsf{rseed}$ in Orchard note decryption
changes, but its type and encoding do not.

Rationale for requiring the lead byte to be {{LEADBYTE}}
````````````````````````````````````````````````````````

It was decided to synchronize the changes to note encryption required for
quantum recoverability, and ZSA support with a change to the
transaction format, for two reasons:

* This was necessary because the note ciphertexts must be a different size to
  those in v6 transactions — to accommodate the ZSA $\mathsf{AssetBase}$ field
  for Orchard.

* The quantum recoverability change did not necessarily have to be done at the
  same time as the ZSA changes. However, it would have
  significantly increased overall protocol complexity for there to be a separate
  note plaintext format for quantum-recoverable notes. Also, doing it at the
  same time minimizes the risk that non-conformant wallets would create outputs
  after this change that could not be successfully decrypted by conformant
  wallets. That could potentially result in inaccessible funds, as was observed
  with the previous note plaintext change at the Canopy upgrade [#zip-0212]_.
  (As noted above, it is still possible for funds to be temporarily stuck if a
  *receiving* wallet does not fully support v6 transactions, but this is less
  problematic because upgrading the wallet will unstick them.)


Transaction Identifiers, Auth Digests, and Signature Digests
============================================================

Transaction identifiers (txids), authorizing-data commitments ("auth digests"),
and signature digests are computed as in ZIP 229 [#zip-0229]_, with a newly updated **Ironwood component digest**, and a newly added **Issuance digest**.

TxId Digest
===========

The overall structure of the TxID digest is as follows; each name referenced
here will either be referenced or described in detail below::

    txid_digest
    ├── header_digest
    ├── transparent_digest
    │   ├── prevouts_digest
    │   ├── sequence_digest
    │   └── outputs_digest
    ├── sapling_digest_v6
    │   ├── sapling_spends_digest
    │   │   ├── sapling_spends_compact_digest
    │   │   └── sapling_spends_noncompact_digest_v6
    │   │      ├── cv
    │   │      └── rk
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
    │   ├── valueBalanceOrchard
    ├── ironwood_digest_zsa *
    │   ├── ironwood_action_groups_digest_zsa *
    │   │   ├── ironwood_actions_compact_digest_zsa *
    │   │   ├── ironwood_actions_memos_digest_v6
    │   │   ├── ironwood_actions_noncompact_digest_zsa *
    │   │   ├── flagsIronwood
    │   │   ├── nAGExpiryHeight *
    │   │   └── ironwood_burn_digest *
    │   └── valueBalanceIronwood
    ├── issuance_digest *
    │   ├── issuerLength *
    │   ├── issuer *
    │   └── issue_actions_digest *
    │       ├── assetDescHash *
    │       ├── issue_notes_digest *
    │       └── flagsIssuance *

Each node written as ``snake_case`` in this tree is a BLAKE2b-256 hash of its
children, initialized with a personalization string specific to that branch
of the tree. Nodes that are not themselves digests are written in ``camelCase``.
In the specification below, nodes of the tree are presented in depth-first order.

The nodes with a ``*`` have new definitions given below. All other nodes have the
same definition as in ZIP 229 [#zip-0229]_.

txid_digest
-----------
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
When OrchardZSA Actions Groups are present in the transaction, this digest is a BLAKE2b-256 hash of the following values::

    T.5a: ironwood_action_groups_digest_zsa  (32-byte hash output)
    T.5b: valueBalanceIronwood               (64-bit signed little-endian)

The personalization field of this hash is set to::

    "ZTxIdIronwdH_zsa"

In the case that the transaction has no OrchardZSA Action Groups, ``orchard_digest`` is ::

    BLAKE2b-256("ZTxIdIronwdH_zsa", [])

T.5a: ironwood_action_groups_digest_zsa
'''''''''''''''''''''''''''''''''''''''

A BLAKE2b-256 hash of the subset of OrchardZSA Action Groups information for all OrchardZSA Action Groups belonging to the transaction.
For each Action Group, the following elements are included in the hash::

    T.5a.i   : ironwood_actions_compact_digest_zsa     (32-byte hash output)
    T.5a.ii  : ironwood_actions_memos_digest_v6        (32-byte hash output)
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

T.5a.iii: ironwood_actions_noncompact_digest
............................................

A BLAKE2b-256 hash of the remaining subset of OrchardZSA Action information **not** intended
for inclusion in an updated version of the the ZIP 307 [#zip-0307]_ ``CompactBlock``
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
'''''''''''''''''''''''''''''

A BLAKE2b-256 hash of the data from the burn fields of the transaction. For each tuple in
the $\mathsf{assetBurn}$ set, the following elements are included in the hash::

    T.5a.vi.1 : assetBase    (field encoding bytes)
    T.5a.vi.2 : valueBurn    (64-bit unsigned little-endian)

The personalization field of this hash is set to::

    "ZTxIdIrnBurnHash"

In case the transaction does not perform the burning of any Assets (i.e. the
$\mathsf{assetBurn}$ set is empty), the ``ironwood_burn_digest`` is::

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

In case the transaction has no issuance components, ``issuance_digest`` is::

    BLAKE2b-256("ZTxIdSAIssueHash", [])

The field encodings are specified in `Transaction Format`_.

T.6a: issue_actions_digest
''''''''''''''''''''''''''
A BLAKE2b-256 hash of Issue Action information for all Issuance Actions belonging to the transaction. For each Action, the following elements are included in the hash::

   T.6a.i  : assetDescHash           (field encoding bytes)
   T.6a.ii : issue_notes_digest      (32-byte hash output)
   T.6a.iii: flagsIssuance           (1 byte)

The personalization field of this hash is set to::

  "ZTxIdIssuActHash"

The field encodings are specified in `Issuance Action Description (IssueAction)`.

T.6a.ii: issue_notes_digest
...........................
A BLAKE2b-256 hash of Note information for all Issue Note Descriptions belonging to the Issuance Action. For each Note, the following elements are included in the hash::

   T.6a.i.1: recipient                    (field encoding bytes)
   T.6a.i.2: value                        (field encoding bytes)
   T.6a.i.3: rho                          (field encoding bytes)
   T.6a.i.4: rseed                        (field encoding bytes)

The personalization field of this hash is set to::

  "ZTxIdIAcNoteHash"

In case the transaction has no Issue Notes, ``issue_notes_digest`` is::

    BLAKE2b-256("ZTxIdIAcNoteHash", [])

The field encodings are specified in `Issue Note Description (IssueNoteDescription)`_.

Authorizing Data Commitment
===========================

The authorizing data commitment (auth digest) is computed as in ZIP 229 [#zip-0229]_, with an updated **Ironwood component digest**, and a newly added **Issuance digest**.
The auth digest structure becomes:

    auth_digest_zsa
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
    └── ironwood_auth_digest_zsa (*)
    │   ├── ironwood_action_groups_auth_digest (+)
    │   ├── bindingSigIronwood
    │   └── anchorIronwood
    └── issuance_auth_digest (+)
      └── issueAuthSig

where (*) indicates an updated definition, and (+) indicates a new definition as compared to ZIP 229 [#zip-0229]_.

Note that while the structure of ``transparent_auth_digest``, ``sapling_auth_digest_v6`` and ``orchard_auth_digest_v6`` remains the same as in ZIP 229, the field encodings of some of its components
(viz. ``TransparentSighashInfo``, ``vSpendAuthSigsSapling``, ``bindingSigSapling``, ``vSpendAuthSigsOrchard`` and ``bindingSigOrchard``) differ due to changes to their field encodings in `Transaction Format`_.

The definitions of the updated/added hashes are given below.

auth_digest_zsa
---------------

A.4: ironwood_auth_digest_zsa
`````````````````````````````

In the case that OrchardZSA Action Groups are present, this is a BLAKE2b-256 hash of the following values::

    A.4a: ironwood_action_groups_auth_digest    (32-byte hash output)
    A.4b: bindingSigIronwood                    (field encoding bytes)
    A.4c: anchorIronwood                        (field encoding bytes)

The personalization field of this hash is::

    "ZTxAuthIrnH__zsa" (2 underscore characters)

In case that the transaction has no OrchardZSA Action Groups, ``ironwood_auth_digest_zsa`` is::

    BLAKE2b-256("ZTxAuthIrnH__zsa", [])

The field encodings are specified in `Transaction Format`_.
Note that this means the encoding of ``bindingSigIronwood`` differs from that used in ZIP 229.

A.4a: ironwood_action_groups_auth_digest
''''''''''''''''''''''''''''''''''''''''

A BLAKE2b-256 hash of the subset of OrchardZSA Action Group information for all OrchardZSA Action Groups belonging to the transaction.
For each OrchardZSA Action Group, this is a BLAKE2b-256 hash of the following values::

    A.4a.i  : proofsIronwood                               (field encoding bytes)
    A.4a.ii : ironwood_spend_auth_sigs_auth_digest         (32-byte hash output)

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
Note that this means the encoding of ``vSpendAuthSigsIronwood`` differs from that used in ZIP 229.

A.5: issuance_auth_digest
-------------------------

In the case that Issuance Actions are present, this is a BLAKE2b-256 hash of the field encoding of the ``issueAuthSig`` field of the transaction::

   A.5a: issueAuthSig            (field encoding bytes)

The personalization field of this hash is set to::

  "ZTxAuthIssH__zsa" (2 underscore characters)

In the case that the transaction has no Orchard Actions, ``issuance_auth_digest`` is ::

  BLAKE2b-256("ZTxAuthIssH__zsa", [])

The field encodings are specified in `Issuance Authorization Signature (IssueAuthSignature)`_.


Deployment
==========

TODO

Reference implementation
========================

TODO


References
==========

.. [#BCP14] `Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words" <https://www.rfc-editor.org/info/bcp14>`_
.. [#protocol] `Zcash Protocol Specification, Version 2025.6.2 or later [NU6.1] <protocol/protocol.pdf>`_
.. [#protocol-spenddesc] `Zcash Protocol Specification, Version 2025.6.2 [NU6.1]. Section 4.4: Spend Descriptions <protocol/protocol.pdf#spenddesc>`_
.. [#protocol-outputdesc] `Zcash Protocol Specification, Version 2025.6.2 [NU6.1]. Section 4.5: Output Descriptions <protocol/protocol.pdf#outputdesc>`_
.. [#protocol-actiondesc] `Zcash Protocol Specification, Version 2025.6.2 [NU6.1]. Section 4.6: Action Descriptions <protocol/protocol.pdf#actiondesc>`_
.. [#protocol-concretehomomorphiccommit] `Zcash Protocol Specification, Version 2025.6.2 [NU6.1]. Section 5.4.8.3: Homomorphic Pedersen commitments (Sapling and Orchard) <protocol/protocol.pdf#concretehomomorphiccommit>`_
.. [#protocol-noteptencoding] `Zcash Protocol Specification, Version 2025.6.2 [NU6.1]. Section 5.5: Encodings of Note Plaintexts and Memo Fields <protocol/protocol.pdf#noteptencoding>`_
.. [#protocol-txnencoding] `Zcash Protocol Specification, Version 2025.6.2 [NU6.1]. Section 7.1: Transaction Encoding and Consensus <protocol/protocol.pdf#txnencoding>`_
.. [#zip-0203] `ZIP 203: Transaction Expiry <zip-0203.rst>`_
.. [#zip-0212] `ZIP 212: Allow Recipient to Derive Ephemeral Secret from Note Plaintext <zip-0212.rst>`_
.. [#zip-0225] `ZIP 225: Version 5 Transaction Format <zip-0225.rst>`_
.. [#zip-0225-transaction-format] `ZIP 225: Version 5 Transaction Format. Specification: Transaction Format <zip-0225.rst#transaction-format>`_
.. [#zip-0226] `ZIP 226: Transfer and Burn of Zcash Shielded Assets <zip-0226.rst>`_
.. [#zip-0226-burn-mechanism] `ZIP 226: Transfer and Burn of Zcash Shielded Assets - Burn Mechanism <zip-0226.rst#burn-mechanism>`_
.. [#zip-0226-orchardzsa-transaction-structure] `ZIP 226: Transfer and Burn of Zcash Shielded Assets — OrchardZSA Transaction Structure <zip-0226.rst#orchardzsa-transaction-structure>`_
.. [#zip-0226-note-structure-and-commitment] `ZIP 226: Transfer and Burn of Zcash Shielded Assets — Note Structure and Commitment <zip-0226#note-structure-and-commitment>`_
.. [#zip-0227] `ZIP 227: Issuance of Zcash Shielded Assets <zip-0227.rst>`_
.. [#zip-0227-issuance-auth-sig] `ZIP 227: Issuance of Zcash Shielded Assets — Issuance Authorization Signature Scheme <zip-0227.rst#issuance-authorization-signature-scheme>`_
.. [#zip-0227-issuer-identifier] `ZIP 227: Issuance of Zcash Shielded Assets — Issuer Identifier <zip-0227.rst#issuer-identifier>`_
.. [#zip-0228] `ZIP 228: Asset Swaps for Zcash Shielded Assets <zip-0228.rst>`_
.. [#zip-0229] `ZIP 229: Version 6 Transaction Format <zip-0229.md>`_
.. [#zip-0231] `ZIP 231: Memo Bundles <zip-0231.md>`_
.. [#zip-0231-motivation] `ZIP 231: Memo Bundles — Motivation <zip-0231.md#motivation>`_
.. [#zip-0231-memo-encryption] `ZIP 231: Memo Bundles — Memo Encryption <zip-0231.md#memo-encryption>`_
.. [#zip-0231-sapling-note-plaintext] `ZIP 231: Memo Bundles — Changes to the Zcash Protocol Specification <zip-0231.md#changes-to-the-zcash-protocol-specification>`_
.. [#zip-0233] `ZIP 233: Network Sustainability Mechanism: Removing Funds From Circulation <zip-0233.md>`_
.. [#zip-0235] `ZIP 235: Remove 60% of Transaction Fees From Circulation <zip-0235.md>`_
.. [#zip-0244] `ZIP 244: Transaction Identifier Non-Malleability <zip-0244.rst>`_
.. [#zip-0246-sighash-info] `ZIP 246: Digests for the Withdrawn Version 6 Transaction Format <zip-0246.rst#zip-0246-sighash-info>`_
.. [#zip-0301] `ZIP 301: Zcash Stratum Protocol <zip-0301.rst>`_
.. [#zip-0307] `ZIP 307: Light Client Protocol for Payment Detection <zip-0307.rst>`_
.. [#zip-2002-motivation] `ZIP 231: Explicit Fees — Motivation <zip-2002.rst#motivation>`_
.. [#zip-2005] `ZIP 2005: Quantum Recoverability <zip-2005.md>`_
.. [#zip-2005-security-analysis] `ZIP 2005: Quantum Recoverability — Security Analysis <zip-2005.md#securityanalysis>`_
.. [#draft-arya-deploy-nu7] `draft-arya-deploy-nu7: Deployment of the NU7 Network Upgrade <draft-arya-deploy-nu7.md>`_
