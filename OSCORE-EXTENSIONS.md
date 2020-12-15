Quick overview over OSCORE extensions that exist or are planned
===============================================================

* Group:
  * New flag for indicating signature
  * adds to `info`

* B.2:
  * Builds its own `id_context`

* ace-oscore-profile:
  * Modifies `master_secret` into a structured one

* Sketched B.2:
  * Builds a structured `id_context`

* cachable-oscore:
  * depends on Group
  * Appends itself to `id_context` (currently; to be changed to do whatever becomes consensus)
