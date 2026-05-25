# WordPress REST API `featured_media` Authorization Bypass

## Sintesi

Questa ricerca documenta una vulnerabilità di authorization boundary nella REST API di WordPress relativa al campo `featured_media`.

Un utente autenticato con ruolo **Author** può creare o modificare un proprio post e assegnare come immagine in evidenza un attachment privato appartenente a un amministratore, anche quando non ha il permesso di leggere direttamente né il media né il post privato a cui quel media è collegato.

La relazione vulnerabile è questa:

```json
{
  "featured_media": "<PRIVATE_ADMIN_ATTACHMENT_ID>"
}
```

Il problema non è un semplice leak REST del media. Il media resta correttamente non leggibile tramite endpoint /wp/v2/media/<id> per l’utente non autorizzato. Tuttavia, la REST API accetta comunque quell’ID come relazione featured_media su un post controllato dall’attaccante.

Quando il post dell’attaccante viene pubblicato, il frontend WordPress renderizza pubblicamente il file media privato e i suoi metadati visibili, ad esempio l’alt_text.

Impatto

L’impatto dimostrato in laboratorio è composto da più primitive concatenabili:

Bypass di autorizzazione sulla relazione featured_media.
Uso di attachment privati admin come featured image su post controllati da utenti meno privilegiati.
Esposizione pubblica del file media attraverso il frontend.
Leak di metadati associati al media privato, incluso alt_text.
ID oracle: un Author può distinguere attachment immagine validi/non validi provando ID nel campo featured_media.
Variante Contributor: un Contributor senza publish_posts e senza upload_files può preparare un post pending con featured image privata admin; se un admin pubblica il contenuto senza notare la relazione, il media viene esposto pubblicamente.
Root cause

La root cause è nella gestione del campo featured_media dentro il controller REST dei post.

Il metodo interessato è:

WP_REST_Posts_Controller::handle_featured_media()

Il flusso osservato è:

protected function handle_featured_media( $featured_media, $post_id ) {
        $featured_media = (int) $featured_media;
        if ( $featured_media ) {
                $result = set_post_thumbnail( $post_id, $featured_media );
                if ( $result ) {
                        return true;
                } else {
                        return new WP_Error(
                                'rest_invalid_featured_media',
                                __( 'Invalid featured media ID.' ),
                                array( 'status' => 400 )
                        );
                }
        } else {
                return delete_post_thumbnail( $post_id );
        }
}

La funzione verifica se set_post_thumbnail() riesce, ma non verifica che l’utente corrente abbia diritto a leggere, usare o referenziare l’attachment target.

In pratica:

l’autorizzazione viene fatta sul post che l’utente sta creando/modificando;
il target featured_media viene accettato come ID;
manca una verifica di accesso sul media referenziato;
il media privato admin viene collegato a un post pubblico dell’attaccante.
Security boundary atteso

Se un utente non può leggere direttamente un media privato admin tramite REST, non dovrebbe poterlo referenziare in un proprio contenuto pubblico in modo da causarne il rendering pubblico.

Nel test locale, invece, WordPress blocca correttamente la lettura diretta:

GET /wp-json/wp/v2/media/<PRIVATE_ADMIN_ATTACHMENT_ID>?context=edit

Risultato atteso e osservato:

{
  "code": "rest_forbidden_context",
  "message": "Non hai i permessi per modificare questo articolo.",
  "data": {
    "status": 403
  }
}

Ma accetta comunque lo stesso ID dentro:

POST /wp-json/wp/v2/posts

oppure:

PATCH /wp-json/wp/v2/posts/<AUTHOR_POST_ID>

con body:

{
  "featured_media": "<PRIVATE_ADMIN_ATTACHMENT_ID>"
}
Scenario Author
Condizioni

Account attaccante:

ruolo: Author
può creare post pubblici
può modificare i propri post
non può leggere media privati admin
non può leggere post privati admin

Account vittima:

ruolo: Administrator
possiede un post privato
possiede un attachment immagine collegato a quel post privato
Risultato

L’Author crea direttamente un post pubblico assegnando il media privato admin come featured image.

Risposta REST osservata:

HTTP/1.1 201 Created

La risposta contiene:

{
  "author": 6,
  "status": "publish",
  "featured_media": 193
}

Proof DB:

post privato admin:
ID=192 post_type=post post_status=private post_author=1

media admin privato:
ID=193 post_type=attachment post_author=1 post_parent=192

post pubblico author:
ID=194 post_type=post post_status=publish post_author=6 _thumbnail_id=193

Il frontend anonimo del post pubblico dell’Author renderizza il media privato admin:

<img
  src="http://wordpress.local/wp-content/uploads/2026/04/admin-private-direct-featured-1777562160.png"
  class="attachment-post-thumbnail size-post-thumbnail wp-post-image"
  alt="ADMIN_PRIVATE_DIRECT_MEDIA_ALT_SECRET_1777562160"
/>

Questo dimostra che un media non leggibile direttamente dall’Author viene esposto pubblicamente attraverso la relazione featured_media.

Scenario ID oracle

La seconda primitiva è un oracle sugli ID degli attachment.

Un Author può creare un proprio draft e provare valori diversi nel campo featured_media.

Quando l’ID corrisponde a un attachment immagine valido, WordPress accetta la relazione e il campo featured_media viene impostato.

Quando l’ID non corrisponde a un media valido per featured image, la relazione non viene applicata.

Output osservato:

[-] miss: media_id=192 featured_media=0
[+] ORACLE HIT: media_id=193 accepted as featured_media
[-] miss: media_id=194 featured_media=0
[-] miss: media_id=195 featured_media=0
[+] ORACLE HIT: media_id=196 accepted as featured_media
[-] miss: media_id=197 featured_media=0

Questo permette a un Author di distinguere attachment immagine esistenti/non esistenti, inclusi attachment che non può leggere direttamente tramite REST.

La chain finale è:

l’Author non può leggere /wp/v2/media/196;
usa featured_media=196 come oracle;
WordPress accetta l’ID;
l’Author pubblica un post con quel media;
il frontend anonimo espone il file e l’alt_text.
Scenario Contributor

La variante Contributor è interessante perché abbassa ulteriormente il requisito di privilegio.

Il Contributor testato aveva:

read=YES
edit_posts=YES
publish_posts=NO
upload_files=NO
edit_others_posts=NO

Nonostante non possa pubblicare post e non possa caricare file, il Contributor può creare un post pending con featured_media impostato su un media privato admin.

Risposta REST osservata:

HTTP/1.1 201 Created

La risposta contiene:

{
  "status": "pending",
  "author": 4,
  "featured_media": 201
}

Proof DB:

post privato admin:
ID=200 post_type=post post_status=private post_author=1

media admin privato:
ID=201 post_type=attachment post_author=1 post_parent=200

post pending contributor:
ID=202 post_type=post post_status=pending post_author=4 _thumbnail_id=201

Se un admin approva/pubblica normalmente il post pending, il frontend pubblico renderizza il media privato admin:

<img
  src="http://wordpress.local/wp-content/uploads/2026/04/admin-private-contributor-featured-1777571359.png"
  class="attachment-post-thumbnail size-post-thumbnail wp-post-image"
  alt="ADMIN_PRIVATE_CONTRIBUTOR_MEDIA_ALT_SECRET_1777571359"
/>

Questa variante è rilevante perché il Contributor non ha bisogno di pubblicare direttamente: può preparare la relazione malevola in un contenuto pending.

Controlli negativi

Durante i test sono stati verificati i seguenti controlli negativi:

un utente Author non può leggere direttamente il post privato admin;
un utente Author non può leggere direttamente il media privato admin via REST;
un utente Subscriber non può modificare il post dell’Author;
il media privato resta formalmente non leggibile via REST per l’utente non autorizzato;
l’esposizione avviene tramite relazione accettata e rendering frontend, non tramite permesso diretto sul media endpoint.
Severità proposta

La severità dipende dal modello di contenuto del sito.

Valutazione tecnica proposta:

Medium/High per siti dove attachment privati possono contenere dati sensibili;
High quando editor/admin usano media privati per documenti, immagini riservate, screenshot, asset interni o allegati non destinati alla pubblicazione;
più grave in ambienti multi-autore o editoriali, dove Author/Contributor sono ruoli semi-trusted.

La presenza dell’ID oracle aumenta l’impatto perché l’attaccante non deve conoscere preventivamente con certezza l’ID del media: può scoprire attachment immagine validi tramite differenze nel comportamento di featured_media.

Fix consigliato

Prima di accettare un valore featured_media, WordPress dovrebbe verificare che l’utente corrente possa leggere/usare l’attachment target.

Una possibile mitigazione è aggiungere un controllo esplicito dentro handle_featured_media() o prima della chiamata a set_post_thumbnail().

Esempio concettuale:

if ( $featured_media && ! current_user_can( 'read_post', $featured_media ) ) {
        return new WP_Error(
                'rest_cannot_use_featured_media',
                __( 'Sorry, you are not allowed to use this media item as featured media.' ),
                array( 'status' => rest_authorization_required_code() )
        );
}

In alternativa, il controllo dovrebbe usare la stessa logica REST/media già applicata quando l’utente prova a leggere direttamente l’attachment via /wp/v2/media/<id>.

Stato disclosure

La vulnerabilità è stata riprodotta in laboratorio locale su WordPress core REST API.

La ricerca è pubblicata come writeup tecnico, con PoC minimizzate e senza dati di terzi.
