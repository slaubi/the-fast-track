Prevenirea spamului cu un API
=============================

.. index::
    single: Spam

Oricine poate trimite o recenzie. Chiar și roboți, spammeri etc. Am putea adăuga un verificator "captcha" în formular pentru a ne proteja de roboți sau putem folosi un API terț.

Am decis să folosesc serviciul gratuit `Akismet <https://akismet.com>`_ pentru a demonstra cum să apelezi un API și cum să efectuezi apelul „independent” de fluxul de bază.

Creare cont Akismet
-------------------

.. index::
    single: Akismet

Creează un cont gratuit pe `akismet.com <https://akismet.com>`_ și obține cheia API.

Dependența de pachetul Symfony HTTPClient
------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

În loc să folosim o bibliotecă care face abstracție de API-ul Akismet, vom efectua toate apelurile direct. Efectuarea apelurilor HTTP este mai eficientă (și ne permite să beneficiem de toate instrumentele de depanare Symfony, cum ar fi integrarea cu depanatorul Symfony).

Pentru a efectua apeluri API, folosește componenta Symfony HttpClient:

.. code-block:: bash

    $ symfony composer req http-client

Proiectarea unei clase de verificare spam
-----------------------------------------

Creează o nouă clasă sub ``src/`` numită ``SpamChecker`` pentru a defini logica de apelare a API-ului Akismet și interpretarea răspunsurilor primite:

.. code-block:: php
    :emphasize-lines: 14,24
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\Contracts\HttpClient\HttpClientInterface;

    class SpamChecker
    {
        private $client;
        private $endpoint;

        public function __construct(HttpClientInterface $client, string $akismetKey)
        {
            $this->client = $client;
            $this->endpoint = sprintf('https://%s.rest.akismet.com/1.1/comment-check', $akismetKey);
        }

        /**
         * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
         *
         * @throws \RuntimeException if the call did not work
         */
        public function getSpamScore(Comment $comment, array $context): int
        {
            $response = $this->client->request('POST', $this->endpoint, [
                'body' => array_merge($context, [
                    'blog' => 'https://guestbook.example.com',
                    'comment_type' => 'comment',
                    'comment_author' => $comment->getAuthor(),
                    'comment_author_email' => $comment->getEmail(),
                    'comment_content' => $comment->getText(),
                    'comment_date_gmt' => $comment->getCreatedAt()->format('c'),
                    'blog_lang' => 'en',
                    'blog_charset' => 'UTF-8',
                    'is_test' => true,
                ]),
            ]);

            $headers = $response->getHeaders();
            if ('discard' === ($headers['x-akismet-pro-tip'][0] ?? '')) {
                return 2;
            }

            $content = $response->getContent();
            if (isset($headers['x-akismet-debug-help'][0])) {
                throw new \RuntimeException(sprintf('Unable to check for spam: %s (%s).', $content, $headers['x-akismet-debug-help'][0]));
            }

            return 'true' === $content ? 1 : 0;
        }
    }

Metoda ``request()`` a clientului HTTP trimite o solicitare POST la adresa URL Akismet (``$this->endpoint``) și transmite o serie de parametri.

Metoda ``getSpamScore()`` returnează 3 valori în funcție de răspunsul la apelul API:

* ``2``: dacă comentariul este un „spam flagrant”;

* ``1``: dacă comentariul ar putea fi spam;

* ``0``: dacă comentariul nu este spam (ham).

.. tip::

    Folosește adresa de e-mail specială ``akismet-garantat-spam@exemplu.com`` pentru a forța ca rezultatul apelului să fie spam.

Utilizarea variabilelor de mediu
--------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Clasa ``SpamChecker`` se bazează pe un argument ``$akismetKey``. Ca și în cazul directorului de încărcare, îl putem injecta printr-o setare a containerului ``bind``:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -12,6 +12,7 @@ services:
             autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
             bind:
                 $photoDir: "%kernel.project_dir%/public/uploads/photos"
    +            $akismetKey: "%env(AKISMET_KEY)%"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

Cu siguranță nu dorim să codificăm valoarea cheii Akismet din fișierul de configurare ``services.yaml``, deci folosim în schimb o variabilă de mediu (``AKISMET_KEY``).

Apoi, fiecare dezvoltator trebuie să stabilească o variabilă de mediu „reală” sau să stocheze valoarea într-un fișier ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Pentru producție, ar trebui definită o variabilă de mediu „reală”.

Acest lucru funcționează bine, dar gestionarea multor variabile de mediu ar putea deveni greoaie. Într-un astfel de caz, Symfony are o alternativă „mai bună” când vine vorba de stocarea secretelor.

Stocarea secretelor
-------------------

.. index::
    single: Secret

În loc să folosească multe variabile de mediu, Symfony poate gestiona un *seif* în care poți stoca mai multe secrete. O caracteristică cheie este capacitatea de a-l stoca în Git (dar fără cheia de a-l deschide). O altă caracteristică excelentă este capacitatea de a gestiona câte un seif per mediu.

.. index:: ! Command;secrets:set

Secretele sunt variabile de mediu criptate.

Adaugă cheia Akismet în seif:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Deoarece este prima dată când executăm această comandă, aceasta a generat două chei în directorul ``config/secret/dev/``. Apoi a stocat secretul ``AKISMET_KEY`` în același director.

Pentru secrete de dezvoltare, poți decide să salvezi seiful și cheile care au fost generate în directorul ``config/secret/dev/``.

De asemenea, secretele pot fi suprascrise prin setarea unei variabile de mediu purtând aceeași denumire.

Verificarea comentariilor contra spam
-------------------------------------

O modalitate simplă de a verifica mesajele de spam, este să apelezi la verificatorul de spam înainte de a stoca datele în baza de date:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentFormType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\File\Exception\FileException;
    @@ -35,7 +36,7 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -53,6 +54,17 @@ class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +
    +            $context = [
    +                'user_ip' => $request->getClientIp(),
    +                'user_agent' => $request->headers->get('user-agent'),
    +                'referrer' => $request->headers->get('referer'),
    +                'permalink' => $request->getUri(),
    +            ];
    +            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    +                throw new \RuntimeException('Blatant spam, go away!');
    +            }
    +
                 $this->entityManager->flush();

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);

Verifică dacă funcționează corect.

Gestionarea secretelor în producție
-------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

Pentru producție, SymfonyCloud acceptă setarea *variabilelor de mediu sensibile*:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

Dar după cum am menționat mai sus, utilizarea secretelor Symfony ar putea fi mai potrivită. Nu în ceea ce privește securitatea, ci în ceea ce privește managementul secretelor pentru echipa proiectului. Toate secretele sunt stocate în repozitoriu și singura variabilă de mediu pe care trebuie să o gestionezi pentru producție este cheia de decriptare. Acest lucru face posibil pentru oricine din echipă să adauge secrete de producție, chiar dacă nu au acces la serverele de producție. Totuși, configurarea este un pic mai implicată.

.. index::
    single: Command;secrets:generate-keys

Mai întâi, generează o pereche de chei pentru utilizarea în producție:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. note:

    The ``APP_ENV=prod`` part before the command allows setting the ``APP_ENV`` environment variable only for this command. On Windows, use ``--env=prod`` instead: ``symfony console secrets:generate-keys --env=prod``

.. index::
    single: Command;secrets:set

Adăugă din nou secretul Akismet în seiful de producție, dar cu valoarea de producție:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

Ultimul pas este să expediezi cheia de decriptare către SymfonyCloud prin setarea unei variabile sensibile:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Poți adăuga și salva toate fișierele; cheia de decriptare a fost adăugată în ``.gitignore`` automat, deci nu va fi salvată niciodată. Pentru mai multă siguranță, o poți scoate de pe sistemul local, așa cum a fost implementat acum:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Mergând mai departe

    * Documentația componentei `HttpClient <https://symfony.com/doc/current/components/http_client.html>`_;

    * `Procesoare variabile de mediu <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * `Notițe Symfony HttpClient <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
