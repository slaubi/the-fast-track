Spam mit Hilfe einer API verhindern
===================================

.. index::
    single: Spam

Jede*r kann Feedback geben. Sogar Roboter, Spammer und mehr. Wir könnten dem Formular ein "Captcha" hinzufügen, um irgendwie vor Robots geschützt zu sein, oder wir nutzen die API eines Drittanbieters.

Ich habe mich entschieden, den kostenlosen `Akismet-Dienst <https://akismet.com>`_ zu nutzen, um zu demonstrieren, wie man eine API aufruft und wie man diesen Aufruf "out of band" macht.

Bei Akismet anmelden
--------------------

.. index::
    single: Akismet

Melde dich kostenlos bei `akismet.com <https://akismet.com>`_ an. Anschließend erhältst Du einen Akismet-API-Schlüssel.

Die Symfony HTTPClient-Komponente verwenden
-------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Anstatt eine Bibliothek zu verwenden, die die Akismet-API abstrahiert, werden wir alle API-Aufrufe direkt ausführen. Die HTTP-Aufrufe selbst auszuführen ist effizienter (und ermöglicht es uns, von allen Symfony-Debugging-Tools wie der Integration mit dem Symfony Profiler zu profitieren).

Um API-Aufrufe zu tätigen, verwende die Symfony HttpClient-Komponente:

.. code-block:: bash

    $ symfony composer req http-client

Eine Spam-Checker-Klasse erstellen
----------------------------------

Erstelle eine neue Klasse unter ``src/`` mit dem Namen ``SpamChecker``, um die Logik des Aufrufs der Akismet-API und der Interpretation ihrer Responses zu bündeln:

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

Die HTTP-Client-Methode ``request()`` sendet einen POST-Request an die Akismet-URL (``$this->endpoint``) und übergibt ein Array mit Parametern.

Die ``getSpamScore()``-Methode gibt je nach API-Response 3 Werte zurück:

* ``2`` wenn der Kommentar "offenkundiger Spam" ist;

* ``1`` wenn der Kommentar Spam sein könnte;

* ``0`` wenn der Kommentar kein Spam (Ham) ist.

.. tip::

    Verwende die besondere E-Mail-Adresse ``akismet-guaranteed-spam@example.com``, um das Ergebnis des API-Calls als Spam zu erzwingen.

Environment-Variablen verwenden
-------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Die ``SpamChecker``-Klasse stützt sich auf ein ``$akismetKey``-Argument. Wie beim Upload-Verzeichnis können wir es über eine ``bind``-Containereinstellung injizieren:

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

Wir wollen den Wert des Akismet-Schlüssels sicherlich nicht fest in die ``services.yaml``-Konfigurationsdatei schreiben, daher verwenden wir stattdessen eine Environment-Variable (``AKISMET_KEY``).

Eine "echte" Environment-Variable zu setzen oder den Wert in einer ``.env.local``-Datei zu speichern, ist Aufgabe der Entwickler*innen:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Für den Produktivbetrieb sollte eine "echte" Environment-Variable definiert werden.

Das funktioniert gut, aber die Verwaltung vieler Environment-Variablen kann umständlich werden. In einem solchen Fall hat Symfony eine "bessere" Alternative, wenn es um die Speicherung solcher Secrets geht.

Secrets speichern
-----------------

.. index::
    single: Secret

Anstatt viele Environment-Variablen zu verwenden, kann Symfony einen *Vault* (Tresor) verwalten, in dem Du viele Secrets speichern kannst. Ein wichtiges Merkmal ist die Möglichkeit, den Vault im Repository zu committen (aber ohne den Schlüssel, um ihn zu öffnen). Ein weiteres großartiges Merkmal ist, dass es einen Vault pro Environment verwalten kann.

.. index:: ! Command;secrets:set

Secrets sind verschleierte Environment-Variablen.

Füge dem Vault den Akismet-Schlüssel hinzu:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Da wir diesen Befehl das erste mal ausgeführt haben, hat er zwei Schlüssel im ``config/secret/dev/``-Verzeichnis erzeugt. Anschließend wurde das ``AKISMET_KEY``-Secret im selben Verzeichnis gespeichert.

Für die Secrets in der Dev-Environment kannst Du selber entscheiden, ob Du den Vault und die Schlüssel, die im ``config/secret/dev/``-Verzeichnis erzeugt wurden, committen möchtest.

Secrets können auch überschrieben werden, indem eine gleichnamige Einvironment-Variable gesetzt wird.

Kommentare auf Spam überprüfen
--------------------------------

Eine einfache Möglichkeit, nach Spam zu suchen, sobald ein neuer Kommentar abgegeben wird, besteht darin, den Spam-Checker aufzurufen, bevor die Daten in der Datenbank gespeichert werden:

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

Überprüfe, ob es einwandfrei funktioniert.

Secrets im Produktivbetrieb verwalten
-------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

Für den Produktivbetrieb unterstützt SymfonyCloud das Setzen *sensibler Environment-Variablen*:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

Wie bereits erwähnt, könnte die Verwendung von Symfony-Secrets jedoch besser sein. Nicht in Bezug auf die Sicherheit, sondern in Bezug auf das Secret-Management für das Projektteam. Alle Secrets werden im Repository gespeichert, und die einzige Environment-Variable, die Du für den Produktivbetrieb verwalten musst, ist der Entschlüsselungscode. Das ermöglicht es allen im Team, Secrets zum Produktivsystem hinzuzufügen, auch wenn sie keinen Zugriff auf das Produktivsystem haben. Das Setup ist jedoch etwas aufwändiger.

.. index::
    single: Command;secrets:generate-keys

Erzeuge zunächst ein Schlüsselpaar für den Produktivbetrieb:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. note:

    The ``APP_ENV=prod`` part before the command allows setting the ``APP_ENV`` environment variable only for this command. On Windows, use ``--env=prod`` instead: ``symfony console secrets:generate-keys --env=prod``

.. index::
    single: Command;secrets:set

Füge das Akismet-Secret für den Produktivbetrieb nun dem Produktiv-Vault hinzu:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

Der letzte Schritt besteht darin, den Entschlüsselungscode an SymfonyCloud zu senden, indem Du eine sensible Variable setzt:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Du kannst alle Dateien zu Git hinzufügen und committen; der Entschlüsselungscode wurde automatisch zu ``.gitignore`` hinzugefügt, so dass er nie commitet wird. Für mehr Sicherheit kannst Du ihn von Deinem lokalen Computer entfernen weil er ja nun im Produktivsystem verfügbar ist:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Weiterführendes

    * Die `Dokumentation der HttpClient-Komponente <https://symfony.com/doc/current/components/http_client.html>`_;

    * Die `Dokumentation zur Verarbeitung von Environment-Variablen <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * Das `Symfony HttpClient Cheat Sheet <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
