Spam mit KI verhindern
======================

.. index::
    single: Spam

Jede*r kann Feedback geben. Sogar Roboter, Spammer und mehr. Wir könnten dem Formular ein "Captcha" hinzufügen, um irgendwie vor Robots geschützt zu sein, oder wir nutzen die API eines Drittanbieters.

Ich habe mich entschieden, ein großes Sprachmodell (LLM) entscheiden zu lassen, ob ein Kommentar Spam ist, um zu demonstrieren, wie man KI in einer Symfony-Anwendung nutzt und wie man solche teuren Aufrufe "out of band" macht.

Einen KI-API-Schlüssel besorgen
-------------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI unterstützt viele Modell-Anbieter: OpenAI, Anthropic, Google Gemini, Mistral und sogar lokale Modelle über Ollama. Dieses Kapitel verwendet OpenAI: Melde Dich bei `platform.openai.com`_ an und erstelle einen API-Schlüssel. Wenn Du einen anderen Anbieter bevorzugst, bleibt der Code derselbe; nur die Konfiguration ändert sich.

Das Symfony AI Bundle verwenden
-------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Anstatt die HTTP-API des Modells selbst aufzurufen, verwenden wir das Symfony AI Bundle. Es bietet eine *Plattform*-Abstraktion für die Modell-Anbieter (jeder Anbieter kommt als eigenes Bridge-Paket) und einen *Agenten*, der ein Modell für die Aufrufe umhüllt; und es profitiert von allen Symfony-Debugging-Tools wie der Integration mit dem Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI ist ein junges Set von Komponenten und noch experimentell: Seine APIs können sich schneller weiterentwickeln als der Rest von Symfony.

Das Rezept der OpenAI-Bridge hat die Plattform bereits für uns konfiguriert; es referenziert eine ``OPENAI_API_KEY``-Environment-Variable (und hat in ``.env`` einen leeren Standardwert dafür hinzugefügt):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Konfiguriere darauf aufbauend einen Standard-*Agenten*:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Environment-Variablen verwenden
-------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Wir wollen den Wert des Schlüssels sicherlich nicht fest in der Konfiguration hinterlegen; deshalb wird er aus der ``OPENAI_API_KEY``-Environment-Variable gelesen.

Eine "echte" Environment-Variable zu setzen oder den Wert in einer ``.env.local``-Datei zu speichern, ist Aufgabe der Entwickler*innen:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

Für den Produktivbetrieb sollte eine "echte" Environment-Variable definiert werden.

Das funktioniert gut, aber die Verwaltung vieler Environment-Variablen kann umständlich werden. In einem solchen Fall hat Symfony eine "bessere" Alternative, wenn es um die Speicherung solcher Secrets geht.

Secrets speichern
-----------------

.. index::
    single: Secret

Anstatt viele Environment-Variablen zu verwenden, kann Symfony einen *Vault* (Tresor) verwalten, in dem Du viele Secrets speichern kannst. Ein wichtiges Merkmal ist die Möglichkeit, den Vault im Repository zu committen (aber ohne den Schlüssel, um ihn zu öffnen). Ein weiteres großartiges Merkmal ist, dass es einen Vault pro Environment verwalten kann.

.. index:: ! Command;secrets:set

Secrets sind verschleierte Environment-Variablen.

Füge dem Vault den OpenAI-API-Schlüssel hinzu:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Da wir diesen Befehl das erste mal ausgeführt haben, hat er zwei Schlüssel im ``config/secret/dev/``-Verzeichnis erzeugt. Anschließend wurde das ``OPENAI_API_KEY``-Secret im selben Verzeichnis gespeichert.

Für die Secrets in der Dev-Environment kannst Du selber entscheiden, ob Du den Vault und die Schlüssel, die im ``config/secret/dev/``-Verzeichnis erzeugt wurden, committen möchtest.

Secrets können auch überschrieben werden, indem eine gleichnamige Einvironment-Variable gesetzt wird.

.. index::
    single: Command;secrets:reveal

Um ein Secret aus dem Vault wieder auszulesen, verwende ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Eine Spam-Checker-Klasse erstellen
----------------------------------

.. index::
    single: AI;Prompt

Erstelle eine neue Klasse unter ``src/`` mit dem Namen ``SpamChecker``, um die Logik zu bündeln, die das Modell fragt, ob ein Kommentar Spam ist:

.. code-block:: php
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\AI\Agent\AgentInterface;
    use Symfony\AI\Platform\Exception\ExceptionInterface;
    use Symfony\AI\Platform\Message\Message;
    use Symfony\AI\Platform\Message\MessageBag;

    class SpamChecker
    {
        public function __construct(
            private AgentInterface $agent,
        ) {
        }

        /**
         * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
         */
        public function getSpamScore(Comment $comment, array $context): int
        {
            $messages = new MessageBag(
                Message::forSystem(<<<PROMPT
                    You moderate comments submitted to a conference guestbook.
                    Classify the comment as "ham", "maybe spam", or "blatant spam".
                    Only answer with the classification.
                    PROMPT),
                Message::ofUser(sprintf(<<<COMMENT
                    IP: %s
                    User agent: %s
                    Author: %s (%s)
                    Comment: %s
                    COMMENT,
                    $context['user_ip'] ?? '',
                    $context['user_agent'] ?? '',
                    $comment->getAuthor(),
                    $comment->getEmail(),
                    $comment->getText(),
                )),
            );

            try {
                $answer = strtolower($this->agent->call($messages)->getContent());
            } catch (ExceptionInterface) {
                // when the model cannot answer, let a human moderate the comment
                return 1;
            }

            return match (true) {
                str_contains($answer, 'blatant spam') => 2,
                str_contains($answer, 'maybe spam') => 1,
                default => 0,
            };
        }
    }

Der *System-Prompt* teilt dem Modell seine Rolle mit und schränkt seine Antworten ein; die *User-Message* enthält den Kommentar und den Kontext seiner Übermittlung (IP-Adresse, User Agent).

Die ``getSpamScore()``-Methode gibt je nach Antwort des Modells 3 Werte zurück:

* ``2``: wenn der Kommentar eindeutig Spam ist ("blatant spam");

* ``1``: wenn der Kommentar Spam sein könnte, oder wenn das Modell nicht erreichbar ist;

* ``0``: wenn der Kommentar kein Spam ist (ham).

Die Ausgabe eines Modells ist Freitext, auch wenn der Prompt sie einschränkt: Parse sie großzügig (wandle sie in Kleinbuchstaben um, verwende ``str_contains()``). Und wenn das Modell gar nicht antworten kann, greife auf menschliche Moderation zurück, statt fehlzuschlagen: KI soll dem Admin helfen, niemals das Gästebuch blockieren.

.. tip::

    Versuche, einen Kommentar abzuschicken, der eindeutig nach Spam aussieht, wie "Buy cheap watches at http://example.com/!!!", um das Modell bei der Arbeit zu sehen.

Kommentare auf Spam überprüfen
------------------------------

Eine einfache Möglichkeit, nach Spam zu suchen, sobald ein neuer Kommentar abgegeben wird, besteht darin, den Spam-Checker aufzurufen, bevor die Daten in der Datenbank gespeichert werden:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,7 +35,8 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -48,6 +50,17 @@ final class ConferenceController extends AbstractController
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

Die Frequenz von Kommentar-Übermittlungen begrenzen
---------------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Spam-Erkennung schützt die Website vor raffinierten Spammern. Ein ergänzender und viel günstigerer Schutz besteht darin, zu begrenzen, wie schnell derselbe Client Kommentare übermitteln kann: Niemand postet legitimerweise Dutzende Kommentare pro Stunde in ein Gästebuch.

Füge die Symfony-Rate-Limiter-Komponente hinzu:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Konfiguriere einen Limiter, der höchstens 5 Kommentare pro Stunde vom selben Client akzeptiert:

.. code-block:: yaml
    :caption: config/packages/rate_limiter.yaml

    framework:
        rate_limiter:
            comment_submission:
                policy: 'fixed_window'
                limit: 5
                interval: '1 hour'

    when@test:
        framework:
            rate_limiter:
                comment_submission:
                    limit: 1000

Automatisierte Tests übermitteln legitimerweise viele Kommentare in kurzer Zeit, daher wird das Limit für die ``test``-Environment angehoben.

Setze den Limiter für Kommentar-Übermittlungen mit dem ``#[RateLimit]``-Attribut durch; standardmäßig identifiziert er Clients anhand ihrer IP-Adresse:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    +use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
    @@ -31,6 +32,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(
             Request $request,

Beachte das ``methods``-Argument: Das Aufrufen einer Konferenzseite ist ein ``GET``-Request und darf nicht begrenzt werden; nur Kommentar-Übermittlungen (``POST``-Requests) werden es.

Wenn das Limit erreicht ist, gibt Symfony automatisch eine ``429 Too Many Requests``-Response mit einem ``Retry-After``-HTTP-Header zurück, der dem Client mitteilt, wann er es erneut versuchen kann.

Dieselbe Komponente schützt auch das Anmeldeformular des Admins vor Brute-Force-Angriffen; das Aktivieren von *Login Throttling* auf der Firewall braucht nur eine Zeile:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -19,6 +19,7 @@ security:
             main:
                 lazy: true
                 provider: app_user_provider
    +            login_throttling: ~
                 form_login:
                     login_path: app_login
                     check_path: app_login

Standardmäßig blockiert Symfony eine IP nach 5 fehlgeschlagenen Anmeldeversuchen mit demselben Benutzernamen innerhalb einer Minute (eine erfolgreiche Anmeldung setzt den Zähler zurück). Verwende die Optionen ``max_attempts`` und ``interval``, um die Richtlinie anzupassen.

Secrets im Produktivbetrieb verwalten
-------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Für den Produktivbetrieb unterstützt Upsun das Setzen *sensibler Environment-Variablen*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

Wie bereits erwähnt, könnte die Verwendung von Symfony-Secrets jedoch besser sein. Nicht in Bezug auf die Sicherheit, sondern in Bezug auf das Secret-Management für das Projektteam. Alle Secrets werden im Repository gespeichert, und die einzige Environment-Variable, die Du für den Produktivbetrieb verwalten musst, ist der Entschlüsselungscode. Das ermöglicht es allen im Team, Secrets zum Produktivsystem hinzuzufügen, auch wenn sie keinen Zugriff auf das Produktivsystem haben. Das Setup ist jedoch etwas aufwändiger.

.. index::
    single: Command;secrets:generate-keys

Erzeuge zunächst ein Schlüsselpaar für den Produktivbetrieb:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note::

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Füge das OpenAI-API-Schlüssel-Secret nun dem Produktiv-Vault hinzu, aber mit seinem Produktiv-Wert:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

Der letzte Schritt besteht darin, den Entschlüsselungscode an Upsun zu senden, indem Du eine sensible Variable setzt:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Du kannst alle Dateien zu Git hinzufügen und committen; der Entschlüsselungscode wurde automatisch zu ``.gitignore`` hinzugefügt, so dass er nie commitet wird. Für mehr Sicherheit kannst Du ihn von Deinem lokalen Computer entfernen weil er ja nun im Produktivsystem verfügbar ist:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Weiterführendes

    * Die `Symfony-AI-Dokumentation`_;

    * Die `Dokumentation zur Verarbeitung von Environment-Variablen`_;

    * `Wie man sensible Informationen geheim hält`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Symfony-AI-Dokumentation`: https://symfony.com/doc/current/ai/index.html
.. _`Dokumentation zur Verarbeitung von Environment-Variablen`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Wie man sensible Informationen geheim hält`: https://symfony.com/doc/current/configuration/secrets.html
