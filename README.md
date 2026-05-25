# WordPress REST API `featured_media` Authorization Bypass

> Ricerca tecnica locale su una boundary issue nella REST API di WordPress.
> Old school rule: niente rumore, solo evidenza riproducibile.

## TL;DR

Un utente autenticato con ruolo **Author** può creare o modificare un proprio post e assegnare come `featured_media` un attachment privato appartenente a un amministratore.

Questo avviene anche quando lo stesso utente riceve correttamente `403 Forbidden` provando a leggere direttamente:

- il post privato admin parent;
- il media admin privato via `/wp-json/wp/v2/media/<id>`.

La relazione accettata dalla REST API è:

```json
{
  "featured_media": "<PRIVATE_ADMIN_ATTACHMENT_ID>"
}
```

Quando il post controllato dall’attaccante viene pubblicato, il frontend WordPress renderizza pubblicamente l’immagine privata e alcuni metadati esposti nel markup, ad esempio `alt_text`.

## Primitive dimostrate

| Primitive | Ruolo | Stato |
|---|---:|---|
| Assegnazione di media admin privato come `featured_media` su post proprio | Author | Confermata |
| Creazione diretta di post pubblico con `featured_media` privato admin | Author | Confermata |
| Oracle sugli ID di attachment immagine non leggibili | Author | Confermata |
| Esposizione pubblica frontend del file media privato | Author | Confermata |
| Leak di metadati renderizzati, incluso `alt_text` | Author | Confermata |
| Staging di post pending con media privato admin | Contributor | Confermata |
| Pubblicazione successiva da parte admin che rende visibile il media | Contributor + workflow editoriale | Confermata |

## Boundary violata

Il controllo REST protegge correttamente la lettura diretta del media privato, ma non blocca l’uso dello stesso media come relazione `featured_media` su un post controllato da un utente meno privilegiato.

In pratica:

```text
GET /wp-json/wp/v2/media/<private_admin_media_id>?context=edit
Author -> 403 Forbidden

POST /wp-json/wp/v2/posts
{ "status": "publish", "featured_media": <private_admin_media_id> }
Author -> 201 Created
```

La seconda richiesta non dovrebbe accettare un attachment che l’utente non può leggere/usare.

## Root cause

La root cause è nella gestione del campo `featured_media` nel controller REST dei post.

Il metodo interessato delega direttamente a `set_post_thumbnail()`:

```php
protected function handle_featured_media( $featured_media, $post_id ) {
    $featured_media = (int) $featured_media;

    if ( $featured_media ) {
        $result = set_post_thumbnail( $post_id, $featured_media );

        if ( $result ) {
            return true;
        }

        return new WP_Error(
            'rest_invalid_featured_media',
            __( 'Invalid featured media ID.' ),
            array( 'status' => 400 )
        );
    }

    return delete_post_thumbnail( $post_id );
}
```

Manca un controllo esplicito del tipo:

```php
current_user_can( 'read_post', $featured_media )
```

oppure un controllo equivalente che verifichi che l’attachment sia leggibile e utilizzabile dall’utente corrente nel contesto del post target.

## Catena Author

1. Admin possiede un post privato con attachment immagine privato collegato.
2. Author non può leggere né il post privato né il media via REST.
3. Author crea un proprio post pubblico via REST.
4. Author imposta `featured_media` uguale all’ID del media admin privato.
5. WordPress accetta la relazione.
6. Il frontend pubblico renderizza il media privato come immagine in evidenza.
7. Il markup espone anche metadati renderizzati come `alt`.

## Catena Contributor

Il Contributor non ha `publish_posts` e non ha `upload_files`, ma può creare un post `pending`.

La variante dimostrata:

1. Contributor crea un post pending.
2. Nel payload inserisce `featured_media=<PRIVATE_ADMIN_ATTACHMENT_ID>`.
3. WordPress salva la relazione `_thumbnail_id`.
4. Un admin approva/pubblica il post nel normale workflow editoriale.
5. Il frontend espone il media privato admin.

Questa variante è interessante perché l’azione finale può essere mascherata dentro un normale flusso di review editoriale.

## ID oracle

Un Author può usare un proprio draft come sonda e provare ID progressivi nel campo `featured_media`.

Risultato osservato:

```text
miss: media_id=192 -> featured_media=0
hit:  media_id=193 -> accepted as featured_media
miss: media_id=194 -> featured_media=0
hit:  media_id=196 -> accepted as featured_media
```

Questo permette di distinguere attachment immagine validi da ID non utilizzabili, anche quando la lettura diretta del media restituisce `403`.

## Evidenze principali

Le evidenze locali raccolte includono:

- baseline negativa con `403 Forbidden` su media privato admin;
- `201 Created` su post Author con `featured_media` privato admin;
- `_thumbnail_id` nel database puntato verso attachment admin privato;
- render frontend anonimo con `src` verso il file admin privato;
- render frontend anonimo con `alt` contenente marker privato;
- variante Contributor pending poi pubblicata.

## Struttura repository

```text
.
├── README.md
├── SECURITY.md
├── docs/
│   └── REPORT_IT.md
└── poc/
    └── README.md
```

## Disclosure

Questa ricerca è stata condotta in laboratorio locale con utenti, post e media di test.

Il materiale è pubblicato come documentazione tecnica e record di riproducibilità. Non deve essere usato contro installazioni WordPress di terzi senza autorizzazione esplicita.

## Stato

- Ricerca locale: completata
- Report tecnico italiano: incluso in `docs/REPORT_IT.md`
- PoC e output ridotti: in preparazione
- Coordinated disclosure: avviata prima della pubblicazione pubblica del writeup
