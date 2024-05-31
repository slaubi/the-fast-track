Prevenire lo spam con un'API
============================

.. index::
    single: Spam

Chiunque può inviare un feedback. Anche robot, spammer e altro ancora. Potremmo aggiungere un po' di "captcha" al form per essere in qualche modo protetti dai robot, oppure possiamo usare API di terze parti.

Ho deciso di utilizzare il servizio antispam gratuito `Akismet`_ per dimostrare come fare chiamate ad un'API e come fare la chiamata "fuori banda".

Iscrizione ad Akismet
---------------------

.. index::
    single: Akismet

Create un account gratuito su `akismet.com`_ e così ottenete la chiave API fornita dal servizio.

Aggiungere il componente HTTPClient di Symfony
----------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Invece di usare una libreria che astrae le API di Akismet, faremo tutte le chiamate API direttamente. Fare da soli le chiamate HTTP è più efficiente (e ci permette di beneficiare di tutti gli strumenti di debug di Symfony, come l'integrazione con il Profiler).

Design di una classe per il controllo dello spam
------------------------------------------------

Create una nuova classe nella cartella ``src/``, chiamatela ``SpamChecker``: la classe conterrà la logica di chiamata alle API di Akismet e la logica per interpretarne le risposte:

.. code-block:: php
    :emphasize-lines: 14,24
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\Contracts\HttpClient\HttpClientInterface;

    class SpamChecker
    {
        private $endpoint;

        public function __construct(
            private HttpClientInterface $client,
            string $akismetKey,
        ) {
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

Il metodo ``request()`` del client HTTP invia una richiesta POST all'URL di Akismet (``$this->endpoint``) e passa un array di parametri.

Il metodo ``getSpamScore()`` restituisce tre possibili valori, che dipendono dalla risposta alla chiamata API:

* ``2``: se il commento è uno "spam palese";

* ``1``: se il commento potrebbe essere spam;

* ``0``: se il commento è sicuro e non è spam (il cosiddetto "ham").

.. tip::

    Usate l'indirizzo speciale ``akismet-guaranteed-spam@example.com`` per forzare il risultato della chiamata a "spam".

Utilizzare le variabili d'ambiente
----------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

La classe ``SpamChecker`` si basa sul parametro ``$akismetKey``. Come per la cartella di caricamento, possiamo iniettarlo tramite un'annotazione dell' ``Autowire``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/SpamChecker.php
    +++ b/src/SpamChecker.php
    @@ -3,6 +3,7 @@
     namespace App;

     use App\Entity\Comment;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Contracts\HttpClient\HttpClientInterface;

     class SpamChecker
    @@ -11,7 +12,7 @@ class SpamChecker

         public function __construct(
             private HttpClientInterface $client,
    -        string $akismetKey,
    +        #[Autowire('%env(AKISMET_KEY)%')] string $akismetKey,
         ) {
             $this->endpoint = sprintf('https://%s.rest.akismet.com/1.1/comment-check', $akismetKey);
         }

Sicuramente non vogliamo forzare il valore della chiave di Akismet direttamente nel codice, per questo motivo usiamo una variabile d'ambiente (``AKISMET_KEY``).

Spetta poi a ogni sviluppatore impostare una variabile d'ambiente "reale" o memorizzare il valore in un file ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Per l'ambiente di produzione, si dovrebbe definire una variabile d'ambiente "reale".

Funziona bene, ma la gestione di molte variabili d'ambiente potrebbe diventare complicata. In questo caso, Symfony offre un'alternativa "migliore" quando si tratta di conservare stringhe segrete.

Salvare stringhe segrete
------------------------

.. index::
    single: Secret

Invece di usare molte variabili d'ambiente, Symfony può gestire un *portachiavi* dove è possibile memorizzare stringhe segrete. Una caratteristica chiave è la possibilità di effettuare il commit del portachiavi nel repository (ma senza la chiave per aprirlo). Un'altra grande caratteristica è che possiamo gestire un portachiavi per ciascun ambiente.

.. index:: ! Command;secrets:set

Le stringhe segrete sono variabili d'ambiente camuffate.

Aggiungete la chiave API Akismet al portachiavi:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Poiché è la prima volta che eseguiamo questo comando, sono state generate due chiavi nella cartella ``config/secret/dev/``. Il comando ha poi memorizzato la stringa segreta ``AKISMET_KEY``, nella stessa cartella.

Per le stringhe segrete usate nell'ambiente di sviluppo, si può decidere di fare il commit del portachiavi e delle chiavi generate nella cartella ``config/secret/dev/``.

Le stringhe segrete possono anche essere sovrascritte, impostando una variabile d'ambiente con lo stesso nome.

Controllo dello spam nei commenti
---------------------------------

Un modo semplice per controllare se un commento appena ricevuto sia da marcare come spam consiste nel richiamare la classe SpamChecker, prima di memorizzare i dati nel database:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,6 +35,7 @@ class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
         ): Response {
             $comment = new Comment();
    @@ -48,6 +50,17 @@ class ConferenceController extends AbstractController
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

Controlliamo che funzioni bene.

Gestire stringhe segrete in produzione
--------------------------------------

.. index::
    single: Platform.sh;Secret
    single: Platform.sh;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Per la produzione, Platform.sh supporta l'impostazione di *variabili d'ambiente sensibili*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef

Ma, come abbiamo già visto, usare le stringhe segrete di Symfony potrebbe essere la scelta migliore. Non in termini di sicurezza, ma in termini di gestione dei segreti nel team. Tutte le stringhe segrete sono memorizzate nel repository e l'unica variabile d'ambiente da gestire per la produzione è la chiave di decrittazione. Questo rende possibile a ciascun membro del team l'aggiunta di stringhe segrete, anche se non ha accesso ai server di produzione. Tuttavia il setup risulterà un po' più complesso.

.. index::
    single: Command;secrets:generate-keys

In primo luogo, generare una coppia di chiavi per l'uso in produzione:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Aggiungere nuovamente la stringa segreta di Akismet nel portachiavi di produzione, ma con il suo valore di produzione:

.. code-block:: terminal
    :class: answers(abcdef)

    $ symfony console secrets:set AKISMET_KEY --env=prod

L'ultimo passo è quello di inviare la chiave di decrittazione a Platform.sh, impostando una variabile:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Si possono aggiungere tutti i file a git in area di stage, ed eseguire il commit. La chiave di decrittazione è stata aggiunta automaticamente al file ``.gitignore``, in modo che sia esclusa da qualsiasi commit. Per maggiore sicurezza, è possibile rimuoverla dalla macchina locale, essendo stata inclusa nel deploy:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Andare oltre

    * `Documentazione del componente HttpClient`_;

    * `Environment Variable Processor`_;

    * `Cheat sheet di Symfony HttpClient`_.

.. _`Akismet`: https://akismet.com
.. _`akismet.com`: https://akismet.com
.. _`Documentazione del componente HttpClient`: https://symfony.com/doc/current/components/http_client.html
.. _`Environment Variable Processor`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Cheat sheet di Symfony HttpClient`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf
