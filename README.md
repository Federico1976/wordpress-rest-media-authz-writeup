# WordPress REST API featured_media Authorization Bypass

Ricerca tecnica su una vulnerabilità di authorization boundary nella REST API di WordPress.

Un utente con ruolo Author o Contributor può referenziare come featured_media un attachment privato appartenente a un amministratore, anche quando non può leggere direttamente il media o il post privato parent.

Il risultato è una catena di impatto composta da:

- bypass di autorizzazione sulla relazione featured_media
- ID oracle per identificare attachment immagine non leggibili
- esposizione pubblica del file media tramite frontend
- leak di metadati come alt text associati al media privato
- variante Contributor con staging pending e pubblicazione successiva da parte di un admin

Stato: ricerca locale in ambiente controllato.

Target testato: WordPress core REST API in laboratorio locale.
