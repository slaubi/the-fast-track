Ochrona przed spamem za pomocą AI
=================================

.. index::
    single: Spam

Każdy może przesłać opinię. Nawet roboty, spamerzy itd. Możemy dodać zabezpieczenie CAPTCHA do formularza, aby w jakiś sposób ochronić się przed robotami, możemy też użyć zewnętrznych API.

Postanowiłem skorzystać z dużego modelu językowego (ang. Large Language Model), aby zdecydować, czy komentarz jest spamem, i pokazać, jak używać AI w aplikacji Symfony oraz jak wykonywać takie kosztowne wywołania „poza widoczną warstwą”.

Uzyskanie klucza API do AI
--------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI obsługuje wielu dostawców modeli: OpenAI, Anthropic, Google Gemini, Mistral, a nawet modele lokalne za pomocą Ollama. Ten rozdział używa OpenAI: zarejestruj się na `platform.openai.com`_ i utwórz klucz API. Jeśli wolisz innego dostawcę, kod pozostaje taki sam; zmienia się tylko konfiguracja.

Zależność od Symfony AI Bundle
------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Zamiast samodzielnie wywoływać API HTTP modelu, użyjemy Symfony AI Bundle. Dostarcza on abstrakcję *platformy* dla dostawców modeli (każdy dostawca jest dostępny jako osobny pakiet mostka) oraz *agenta*, który opakowuje model, aby wykonywać wywołania; korzysta też ze wszystkich narzędzi Symfony do debugowania, takich jak integracja z Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI to młody zestaw komponentów, wciąż eksperymentalny: jego API mogą ewoluować szybciej niż reszta Symfony.

Przepis mostka OpenAI skonfigurował już dla nas platformę; odwołuje się do zmiennej środowiskowej ``OPENAI_API_KEY`` (i dodał dla niej pustą wartość domyślną w ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Skonfiguruj na tej podstawie domyślnego *agenta*:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Korzystanie ze zmiennych środowiskowych
----------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Z pewnością nie chcemy zapisywać na stałe wartości klucza w konfiguracji; dlatego jest ona odczytywana ze zmiennej środowiskowej ``OPENAI_API_KEY``.

Następnie każdy programista ustawia faktyczną zmienną środowiskową lub zapisuje jej wartość w pliku ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

W przypadku środowiska produkcyjnego, należy zdefiniować faktyczną zmienną środowiskową.

Działa to nieźle, ale zarządzanie wieloma zmiennymi środowiskowymi może stać się uciążliwe. W takim przypadku Symfony pozwala lepiej rozwiązać przechowywanie poufnych danych (ang. secrets).

Przechowywanie poufnych danych (ang. secrets)
---------------------------------------------

.. index::
    single: Secret

Zamiast używać wielu zmiennych środowiskowych, Symfony może zarządzać *sejfem*, w którym można przechowywać wiele poufnych danych. Jedną z kluczowych funkcji jest możliwość zapisywania sejfu w repozytorium (jednak bez klucza do jego otwarcia). Kolejną świetną cechą tego rozwiązania jest to, że możemy zarządzać jednym sejfem w ramach jednego środowiska.

.. index:: ! Command;secrets:set

Poufne dane są zamaskowanymi zmiennymi środowiskowymi.

Dodaj klucz API OpenAI do sejfu:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Ponieważ uruchamiamy to polecenie po raz pierwszy, w katalogu ``config/secret/dev/`` pojawiły się dwa klucze. Następnie w tym samym katalogu został zapisany ``OPENAI_API_KEY``.

Podczas prac w środowisku deweloperskim, możesz zdecydować się na zapisanie w repozytorium sejfu oraz kluczy, które zostały wygenerowane w katalogu ``config/secret/dev/``.

Wartości poufnych danych mogą być nadpisane ustawieniem zmiennej środowiskowej o tej samej nazwie.

.. index::
    single: Command;secrets:reveal

Aby odczytać poufne dane z sejfu, użyj ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Projektowanie klasy Spam Checker
--------------------------------

.. index::
    single: AI;Prompt

Utwórz nową klasę w katalogu ``src/`` pod nazwą ``SpamChecker``, aby opakować logikę pytania modelu, czy komentarz jest spamem:

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

*System prompt* informuje model o jego roli i ogranicza jego odpowiedzi; *wiadomość użytkownika* zawiera komentarz oraz kontekst jego wysłania (adres IP, user agent).

Metoda ``getSpamScore()`` zwraca trzy wartości w zależności od odpowiedzi modelu:

* ``2`` jeśli komentarz jest „rażącym spamem” (ang. blatant spam);

* ``1`` jeśli komentarz może być spamem lub gdy model jest nieosiągalny;

* ``0`` jeśli komentarz nie jest spamem (ham).

Wyjście modelu to wolny tekst, nawet gdy prompt go ogranicza: parsuj je swobodnie (zamień na małe litery, użyj ``str_contains()``). A gdy model w ogóle nie może odpowiedzieć, zamiast zawodzić, przejdź na moderację przez człowieka: AI powinno pomagać administratorowi, nigdy nie blokować księgi gości.

.. tip::

    Spróbuj przesłać komentarz, który wygląda na rażący spam, np. "Buy cheap watches at http://example.com/!!!", aby zobaczyć model w akcji.

Sprawdzanie komentarzy pod kątem spamu
---------------------------------------

Podczas wysyłania nowego komentarza, prostym sposobem na sprawdzenie, czy nie jest on spamem, jest wykorzystanie obiektu klasy ``SpamChecker`` przed zapisaniem danych do bazy danych:

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

Sprawdź, czy dobrze to działa.

Ograniczanie liczby wysyłanych komentarzy (ang. rate limiting)
--------------------------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Wykrywanie spamu chroni witrynę przed wyrafinowanymi spamerami. Uzupełniającą i znacznie tańszą ochroną jest ograniczenie tego, jak szybko ten sam klient może wysyłać komentarze: nikt zgodnie z prawem nie publikuje dziesiątek komentarzy na godzinę w księdze gości.

Dodaj komponent Symfony Rate Limiter:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Skonfiguruj limiter, który akceptuje najwyżej 5 komentarzy na godzinę od tego samego klienta:

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

Zautomatyzowane testy zgodnie z prawem wysyłają wiele komentarzy w krótkim czasie, więc limit jest podniesiony dla środowiska ``test``.

Wymuś limiter na wysyłanych komentarzach za pomocą atrybutu ``#[RateLimit]``; domyślnie identyfikuje on klientów na podstawie ich adresu IP:

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

Zwróć uwagę na argument ``methods``: przeglądanie strony konferencji to żądanie ``GET`` i nie może być ograniczane; ograniczane są tylko wysyłane komentarze (żądania ``POST``).

Gdy limit zostanie osiągnięty, Symfony automatycznie zwraca odpowiedź ``429 Too Many Requests`` z nagłówkiem HTTP ``Retry-After``, który informuje klienta, kiedy może ponowić próbę.

Ten sam komponent chroni również formularz logowania administratora przed atakami brute-force; włączenie *ograniczania logowania* (ang. login throttling) na firewallu zajmuje jedną linię:

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

Domyślnie Symfony blokuje adres IP po 5 nieudanych próbach logowania na tę samą nazwę użytkownika w ciągu minuty (udane logowanie zeruje licznik). Użyj opcji ``max_attempts`` i ``interval``, aby dostosować politykę.

Zarządzanie poufnymi danymi w środowisku produkcyjnym
-------------------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

W środowisku produkcyjnym, Upsun obsługuje ustawianie *poufnych zmiennych środowiskowych*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

Ale jak wspomniano powyżej, użycie sejfu przechowującego poufne dane może być lepsze. Nie zwiększa bezpieczeństwa, ale ułatwia zarządzanie nimi w zespole projektowym. Wszystkie poufne dane są przechowywane w repozytorium, a jedyną zmienną środowiskową, o którą musisz zadbać w środowisku produkcyjnym, jest klucz odszyfrowujący. Dzięki temu każdy w zespole może dodać poufne dane, nawet jeśli nie ma dostępu do serwerów produkcyjnych. Konfiguracja jest jednak nieco bardziej skomplikowana.

.. index::
    single: Command;secrets:generate-keys

Po pierwsze, wygeneruj parę kluczy do użytku produkcyjnego:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note::

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Ponownie wprowadź klucz do API OpenAI w sejfie produkcyjnym, ale z wartością dla środowiska produkcyjnego:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

Ostatnim krokiem jest wysłanie klucza odszyfrowującego do Upsun poprzez ustawienie poufnej zmiennej:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Możesz dodawać i zatwierdzać (ang. commit) w repozytorium wszystkie pliki; klucz odszyfrowujący został dodany do ``.gitignore`` automatycznie, więc nigdy nie zostanie w nim zatwierdzony. Dla większego bezpieczeństwa można go usunąć z maszyny lokalnej, ponieważ został już wdrożony:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Idąc dalej

    * `Dokumentacja Symfony AI`_;

    * `Procesory zmiennych środowiskowych`_;

    * `Jak chronić poufne informacje`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Dokumentacja Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`Procesory zmiennych środowiskowych`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Jak chronić poufne informacje`: https://symfony.com/doc/current/configuration/secrets.html
