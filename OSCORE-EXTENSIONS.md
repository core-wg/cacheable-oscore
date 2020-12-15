Quick overview over OSCORE extensions that exist or are planned
===============================================================

* Group:
  * New flag for indicating signature
  * adds to `external_aad`; especially `request_kid_context` to be able to respond across `kid_context` changes

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

* Have a dict where everything gets its (registered? configured-by-whoever-creates-context?) index
* Everything goes into where ace-oscore-profile puts it (ie. `master_salt` becomes an array)
* For evolvability with devices that can't do it, a single item array is just put in
  * Do we really need that? Depends on whether we can expect that implementors of what's-currently-in-the-draft to just add a [] inbetween their salt sources and their salt.
* Whatever is the source of the key material picks a scheme like [kid_context(=gid), id_detail?, b2data?] where extensions currently unused take a null CBOR value
