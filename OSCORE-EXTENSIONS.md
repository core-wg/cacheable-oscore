Quick overview over OSCORE extensions that exist or are planned
===============================================================

* Group:
  * New flag for indicating signature
  * Derives many individual contexts that all have their own replay window.
  * adds to `external_aad`; especially `request_kid_context` to be able to respond across `kid_context` changes

* B.1.2:
  * No addition, just allows to get from "my replay window is broken" to "I have a valid replay window"

* B.2:
  * Builds its own `id_context`

* ace-oscore-profile:
  * Modifies `master_salt` into a structured one

* Sketched B.2:
  * Section 2 of https://hackmd.io/XzBeg0NjSH-mhNBpY471Fw?both
  * Builds a structured `id_context`

* cachable-oscore:
  * depends on Group
  * Appends itself to `id_context` (currently; to be changed to do whatever becomes consensus)

  * Sketched B.2 for rekeying the pairwise mode of Group OSCORE:
  * Section 3 of https://hackmd.io/XzBeg0NjSH-mhNBpY471Fw?both
  * Builds a structured `id_context`
  
Potential unified mechanism
---------------------------

* Everything goes into where ace-oscore-profile puts it (ie. `master_salt` becomes an array)
  * The key source determines how exactly.
* For evolvability with devices that can't do it, a single item array is just put in
  * Do we really need that? Depends on whether we can expect that implementors of what's-currently-in-the-draft to just add a [] inbetween their salt sources and their salt.
* Applied to the above:
  * ace-oscore-profile already does that (by design); it picks that the `master_salt` is always `[original_salt, n1, n2]` and never degrades into bare-salt mode
  * Sketched B.2 (which would be runnable even for contexts that initially start out good):
    it's initially `original_salt`, but becomes `[original_salt, r2, r3]` during an exchange and stays that way. Compared to original B.2, the goal shifts from establishing a new ID Context to establishing a new Master Salt.
  * Deterministic requests:
    This needs to go into the group part in order to a) allow for request-to-group and b) to get the cryptographic binding between request and reponse.
    So *when* a request has the id-detail, its master salt becomes `[master_salt, id_detail]`.
    (On top of that, the later pairwise derivation may do its own sketched b2, but that's in the pairwise derivation then).
  * Sketched B.2 for rekeying the pairwise mode of Group OSCORE:
    Here the goal is to derive new pairwise keys for the two peers, keeping the same Group Security Context as is. In the HKDF to derive Pairwise Sender/Recipient Key, the first argument is currently Sender/Recipient Key. Instead, it can become the CBOR byte encoding of `[X, Y]`, where X is the Sender/Recipient Key, while Y is the additional pairwise information exchanged in the message, and included in the 'kid_context' field of the OSCORE option as concatenated to the ID Context.
* No two of these mechanisms collide -- but if any further extension wants to go into a spot.
  * This is all assuming that Sketched B.2 could just be applied like that by any pairwise context without involvement of the KDC; for interoperability devices might prefer to only do that if the KDC said that this is A Thing here -- in which case it may be better to use the array version all the time.
  (It's not like regular OSCORE devices can expect to start this and just work without prior agreement either)
  * Added extensions that may conflict with any of this need to state where they are added in.
