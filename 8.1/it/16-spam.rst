Prevenire lo spam con l'IA
==========================

.. index::
    single: Spam

Chiunque può inviare un feedback. Anche robot, spammer e altro ancora. Potremmo aggiungere un po' di "captcha" al form per essere in qualche modo protetti dai robot, oppure possiamo usare API di terze parti.

Ho deciso di utilizzare un grande modello linguistico (LLM) per decidere se un commento sia spam, per dimostrare come usare l'IA in un'applicazione Symfony e come fare queste chiamate costose "fuori banda".

Ottenere una chiave API di IA
-----------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI supporta molti fornitori di modelli: OpenAI, Anthropic, Google Gemini, Mistral e persino modelli locali tramite Ollama. Questo capitolo usa OpenAI: create un account su `platform.openai.com`_ e generate una chiave API. Se preferite un altro fornitore, il codice resta lo stesso; cambia solo la configurazione.

Aggiungere il bundle Symfony AI
-------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Invece di chiamare noi stessi le API HTTP del modello, useremo il bundle Symfony AI. Fornisce un'astrazione di *piattaforma* per i fornitori di modelli (ogni fornitore arriva come pacchetto bridge dedicato) e un *agente* che avvolge un modello per effettuare le chiamate; e beneficia di tutti gli strumenti di debug di Symfony, come l'integrazione con il Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI è un insieme di componenti giovane e ancora sperimentale: le sue API possono evolversi più velocemente del resto di Symfony.

La ricetta del bridge OpenAI ha già configurato la piattaforma per noi; fa riferimento a una variabile d'ambiente ``OPENAI_API_KEY`` (e ha aggiunto in ``.env`` un valore predefinito vuoto):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Configurare un *agente* predefinito basato su di essa:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Utilizzare le variabili d'ambiente
----------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Sicuramente non vogliamo forzare il valore della chiave direttamente nella configurazione; per questo motivo viene letta dalla variabile d'ambiente ``OPENAI_API_KEY``.

Spetta poi a ogni sviluppatore impostare una variabile d'ambiente "reale" o memorizzare il valore in un file ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

Per l'ambiente di produzione, si dovrebbe definire una variabile d'ambiente "reale".

Funziona bene, ma la gestione di molte variabili d'ambiente potrebbe diventare complicata. In questo caso, Symfony offre un'alternativa "migliore" quando si tratta di conservare stringhe segrete.

Salvare stringhe segrete
------------------------

.. index::
    single: Secret

Invece di usare molte variabili d'ambiente, Symfony può gestire un *portachiavi* dove è possibile memorizzare stringhe segrete. Una caratteristica chiave è la possibilità di effettuare il commit del portachiavi nel repository (ma senza la chiave per aprirlo). Un'altra grande caratteristica è che possiamo gestire un portachiavi per ciascun ambiente.

.. index:: ! Command;secrets:set

Le stringhe segrete sono variabili d'ambiente camuffate.

Aggiungete la chiave API di OpenAI al portachiavi:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Poiché è la prima volta che eseguiamo questo comando, sono state generate due chiavi nella cartella ``config/secret/dev/``. Il comando ha poi memorizzato la stringa segreta ``OPENAI_API_KEY``, nella stessa cartella.

Per le stringhe segrete usate nell'ambiente di sviluppo, si può decidere di fare il commit del portachiavi e delle chiavi generate nella cartella ``config/secret/dev/``.

Le stringhe segrete possono anche essere sovrascritte, impostando una variabile d'ambiente con lo stesso nome.

.. index::
    single: Command;secrets:reveal

Per rileggere una stringa segreta dal portachiavi, usare ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Design di una classe per il controllo dello spam
------------------------------------------------

.. index::
    single: AI;Prompt

Create una nuova classe nella cartella ``src/``, chiamatela ``SpamChecker``: la classe conterrà la logica che chiede al modello se un commento sia spam:

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

Il *prompt di sistema* comunica al modello il suo ruolo e ne vincola le risposte; il *messaggio utente* contiene il commento e il contesto del suo invio (indirizzo IP, user agent).

Il metodo ``getSpamScore()`` restituisce tre possibili valori, che dipendono dalla risposta del modello:

* ``2``: se il commento è palesemente spam ("blatant spam");

* ``1``: se il commento potrebbe essere spam, o quando il modello non è raggiungibile;

* ``0``: se il commento non è spam (ham).

L'output di un modello è testo libero, anche quando il prompt lo vincola: analizzarlo con tolleranza (convertirlo in minuscolo, usare ``str_contains()``). E quando il modello non riesce proprio a rispondere, ripiegare sulla moderazione umana invece di fallire: l'IA deve aiutare l'amministratore, mai bloccare il libro degli ospiti.

.. tip::

    Provate a inviare un commento che sembri palesemente spam, come "Buy cheap watches at http://example.com/!!!", per vedere il modello al lavoro.

Controllo dello spam nei commenti
---------------------------------

Un modo semplice per controllare se un commento appena ricevuto sia da marcare come spam consiste nel richiamare la classe SpamChecker, prima di memorizzare i dati nel database:

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
             #[MapQueryParameter] int $offset = 0,
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

Controlliamo che funzioni bene.

Limitare la frequenza di invio dei commenti
-------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Rilevare lo spam protegge il sito dagli spammer sofisticati. Una protezione complementare e molto più economica consiste nel limitare la velocità con cui lo stesso client può inviare commenti: nessuno pubblica legittimamente decine di commenti all'ora su un libro degli ospiti.

Aggiungere il componente Rate Limiter di Symfony:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Configurare un limitatore che accetti al massimo 5 commenti all'ora dallo stesso client:

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

I test automatizzati inviano legittimamente molti commenti in poco tempo, quindi il limite viene alzato per l'ambiente ``test``.

Applicare il limitatore agli invii dei commenti con l'attributo ``#[RateLimit]``; in modo predefinito, identifica i client tramite il loro indirizzo IP:

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

Si noti l'argomento ``methods``: navigare nella pagina di una conferenza è una richiesta ``GET`` e non deve essere limitata; lo sono solo gli invii dei commenti (richieste ``POST``).

Quando il limite viene raggiunto, Symfony restituisce automaticamente una risposta ``429 Too Many Requests`` con un header HTTP ``Retry-After`` che indica al client quando potrà riprovare.

Lo stesso componente protegge anche il form di login dell'amministratore dagli attacchi a forza bruta; abilitare il *login throttling* sul firewall richiede una sola riga:

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

In modo predefinito, Symfony blocca un IP dopo 5 tentativi di login falliti sullo stesso nome utente entro un minuto (un login riuscito azzera il contatore). Usare le opzioni ``max_attempts`` e ``interval`` per regolare la politica.

Gestire stringhe segrete in produzione
--------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Per la produzione, Upsun supporta l'impostazione di *variabili d'ambiente sensibili*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

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

Aggiungere nuovamente la stringa segreta della chiave API di OpenAI nel portachiavi di produzione, ma con il suo valore di produzione:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

L'ultimo passo è quello di inviare la chiave di decrittazione a Upsun, impostando una variabile:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Si possono aggiungere tutti i file a git in area di stage, ed eseguire il commit. La chiave di decrittazione è stata aggiunta automaticamente al file ``.gitignore``, in modo che sia esclusa da qualsiasi commit. Per maggiore sicurezza, è possibile rimuoverla dalla macchina locale, essendo stata inclusa nel deploy:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Andare oltre

    * La `documentazione di Symfony AI`_;

    * `Environment Variable Processor`_;

    * `Come mantenere segrete le informazioni sensibili`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`documentazione di Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`Environment Variable Processor`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Come mantenere segrete le informazioni sensibili`: https://symfony.com/doc/current/configuration/secrets.html
