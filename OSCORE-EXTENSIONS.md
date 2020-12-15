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
  * Builds a structured `id_context`

* cachable-oscore:
  * depends on Group
  * Appends itself to `id_context` (currently; to be changed to do whatever becomes consensus)

Potential unified mechanism
---------------------------

* Everything goes into where ace-oscore-profile puts it (ie. `master_salt` becomes an array)
  * The key source determines how exactly.
* For evolvability with devices that can't do it, a single item array is just put in
  * Do we really need that? Depends on whether we can expect that implementors of what's-currently-in-the-draft to just add a [] inbetween their salt sources and their salt.
* Applied to the above:
  * ace-oscore-profile already does that (by design); it picks that the `master_salt` is always `[original_salt, n1, n2]` and never degrades into bare-salt mode
  * Sketched B.2 (which would be runnable even for contexts that initially start out good):
    it's initially `original_salt`, but becomes `[original_salt, r2, r3]` during an exchange and stays that way.
  * Deterministic requests:
    This needs to go into the group part in order to a) allow for request-to-group and b) to get the cryptographic binding between request and reponse.
    So *when* a request has the id-detail, its master salt becomes `[master_salt, id_detail]`.
    (On top of that, the later pairwise derivation may do its own sketched b2, but that's in the pairwise derivation then).
* No two of these mechanisms collide -- but if any further extension wants to go into a spot.
  * This is all assuming that Sketched B.2 could just be applied like that by any pairwise context without involvement of the KDC; for interoperability devices might prefer to only do that if the KDC said that this is A Thing here -- in which case it may be better to use the array version all the time.
  (It's not like regular OSCORE devices can expect to start this and just work without prior agreement either)
  * Added extensions that may conflict with any of this need to state where they are added in.
