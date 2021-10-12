Ochrona przed spamem przy pomocy API
====================================

.. index::
    single: Spam

Każdy może przesłać opinię. Nawet roboty, spamerzy itd. Możemy dodać zabezpieczenie CAPTCHA do formularza, aby w jakiś sposób ochronić się przed robotami, możemy też użyć zewnętrznych API.

Postanowiłem skorzystać z darmowej usługi `Akismet <https://akismet.com>`_, aby zademonstrować, jak wywołać zapytanie do API i jak wykonać połączenie „poza widoczną warstwą”.

Rejestracja w Akismet
---------------------

.. index::
    single: Akismet

Zarejestruj bezpłatne konto na `akismet.com <https://akismet.com>`_ i uzyskaj klucz API Akismet.

Zależność od komponentu Symfony HTTPClient
---------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Zamiast korzystać z biblioteki, która obsługuje API Akismet, wykonamy wszystkie zapytania do API bezpośrednio. Wykonywanie zapytań HTTP samodzielnie jest bardziej efektywne (i pozwala nam korzystać ze wszystkich narzędzi Symfony do debugowania, takich jak integracja z Symfony Profiler).

Aby wykonywać zapytanie do API użyj komponentu Symfony HttpClient:

.. code-block:: bash

    $ symfony composer req http-client

Projektowanie klasy Spam Checker
--------------------------------

Utwórz nową klasę w katalogu ``src/`` pod nazwą ``SpamChecker`` w której zawrzemy schemat działań odpowiadających za wysłanie zapytania do API Akismet i przetworzenie jego odpowiedzi.

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

Metoda ``request()`` klienta HTTP wysyła zapytanie POST pod URL Akismet (``$this->endpoint``) i przekazuje tablicę parametrów.

Metoda ``getSpamScore()`` zwraca trzy wartości w zależności od odpowiedzi z API:

* ``2`` jeśli komentarz jest „rażącym spamem” (ang. blatant spam);

* ``1`` jeśli komentarz może być spamem;

* ``0`` jeśli komentarz nie jest spamem.

.. tip::

    Użyj specjalnego adresu e-mail ``akismet-guaranteed-spam@example.com``, aby wynik wywołania potraktować jako spam.

Korzystanie ze zmiennych środowiskowych
----------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Klasa ``SpamChecker`` jest zależna od argumentu ``$akismetKey``. Podobnie jak w przypadku katalogu do zapisu plików, możemy wstrzyknąć ten argument za pomocą ustawienia ``bind`` kontenera:

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

Z pewnością nie chcemy zapisywać na stałe wartości klucza Akismet w pliku konfiguracyjnym ``services.yaml`` więc zamiast tego użyjemy zmiennej środowiskowej (``AKISMET_KEY``).

Następnie każdy programista ustawia faktyczną zmienną środowiskową lub zapisuje jej wartość w pliku ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

W przypadku środowiska produkcyjnego, należy zdefiniować faktyczną zmienną środowiskową.

Działa to nieźle, ale zarządzanie wieloma zmiennymi środowiskowymi może stać się uciążliwe. W takim przypadku Symfony pozwala lepiej rozwiązać przechowywanie poufnych danych (ang. secrets).

Przechowywanie poufnych danych (ang. secrets)
---------------------------------------------

.. index::
    single: Secret

Zamiast używać wielu zmiennych środowiskowych, Symfony może zarządzać *sejfem*, w którym można przechowywać wiele poufnych danych. Jedną z kluczowych funkcji jest możliwość zapisywania sejfu w repozytorium (jednak bez klucza do jego otwarcia). Kolejną świetną cechą tego rozwiązania jest to, że możemy zarządzać jednym sejfem w ramach jednego środowiska.

.. index:: ! Command;secrets:set

Poufne dane są zamaskowanymi zmiennymi środowiskowymi.

Dodaj klucz API Akismet do sejfu:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Ponieważ uruchamiamy to polecenie po raz pierwszy, w katalogu ``config/secret/dev/`` pojawiły się dwa klucze. Następnie w tym samym katalogu został zapisany ``AKISMET_KEY``.

Podczas prac w środowisku deweloperskim, możesz zdecydować się na zapisanie w repozytorium sejfu oraz kluczy, które zostały wygenerowane w katalogu ``config/secret/dev/``.

Wartości poufnych danych mogą być nadpisane ustawieniem zmiennej środowiskowej o tej samej nazwie.

Sprawdzanie komentarzy pod kątem spamu
---------------------------------------

Podczas wysyłania nowego komentarza, prostym sposobem na sprawdzenie, czy nie jest on spamem, jest wykorzystanie obiektu klasy ``SpamChecker`` przed zapisaniem danych do bazy danych:

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
    @@ -39,7 +40,7 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{slug}", name="conference")
          */
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository, SpamChecker $spamChecker, string $photoDir): Response
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    @@ -57,6 +58,17 @@ class ConferenceController extends AbstractController
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

Zarządzanie poufnymi danymi w środowisku produkcyjnym
-------------------------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

W środowisku produkcyjnym, SymfonyCloud obsługuje ustawianie *poufnych zmiennych środowiskowych*:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

Ale jak wspomniano powyżej, użycie sejfu przechowującego poufne dane może być lepsze. Nie zwiększa bezpieczeństwa, ale ułatwia zarządzanie nimi w zespole projektowym. Wszystkie poufne dane są przechowywane w repozytorium, a jedyną zmienną środowiskową, o którą musisz zadbać w środowisku produkcyjnym, jest klucz odszyfrowujący. Dzięki temu każdy w zespole może dodać poufne dane, nawet jeśli nie ma dostępu do serwerów produkcyjnych. Konfiguracja jest jednak nieco bardziej skomplikowana.

.. index::
    single: Command;secrets:generate-keys

Po pierwsze, wygeneruj parę kluczy do użytku produkcyjnego:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Ponownie wprowadź klucz do API Akismet w sejfie produkcyjnym, ale z wartością dla środowiska produkcyjnego:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

Ostatnim krokiem jest wysłanie klucza odszyfrowującego do SymfonyCloud poprzez ustawienie poufnej zmiennej:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Możesz dodawać i zatwierdzać (ang. commit) w repozytorium wszystkie pliki; klucz odszyfrowujący został dodany do ``.gitignore`` automatycznie, więc nigdy nie zostanie w nim zatwierdzony. Dla większego bezpieczeństwa można go usunąć z maszyny lokalnej, ponieważ został już wdrożony do SymfonyCloud:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Idąc dalej

    * `Dokumentacja komponentu HttpClient <https://symfony.com/doc/current/components/http_client.html>`_;

    * `Procesory zmiennych środowiskowych <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * `Ściągawka Symfony HttpClient <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
