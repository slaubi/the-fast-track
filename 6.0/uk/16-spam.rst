Запобігання спаму за допомогою API
=============================================================

.. index::
    single: Spam

Будь-хто може надіслати відгук. Це можуть бути роботи, спамери тощо. Ми можемо додати "капчу" до форми, щоб хоч якось захиститися від роботів, або скористатися API сторонніх розробників.

Я вирішив скористатися безкоштовним сервісом `Akismet`_, щоб продемонструвати, як можна працювати з API та виконувати зовнішні запити.

Реєстрація в Akismet
-------------------------------

.. index::
    single: Akismet

Зареєструйте безкоштовний обліковий запис на `akismet.com`_ і отримайте ключ Akismet API.

Залежність від компоненту Symfony HTTPClient
-------------------------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Замість використання бібліотеки, яка абстрагує API Akismet, ми будемо виконувати всі виклики API безпосередньо. Виконання HTTP-запитів самостійно більш ефективно (і дозволяє нам скористатися всіма інструментами налагодження Symfony, як-от Symfony Profiler).

Розробка класу перевірки на спам
------------------------------------------------------------

Створіть новий клас в ``src/`` із назвою ``SpamChecker``, щоб описати логіку виклику Akismet API та обробки відповідей:

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

Метод HTTP-клієнта ``request()`` відправляє POST-запит до URL-адреси Akismet (``$this->endpoint``) і передає масив параметрів.

Метод ``getSpamScore()`` повертає 3 значення в залежності від відповіді на виклик API:

* ``2``: якщо коментар є "явним спамом";

* ``1``: Якщо коментар може бути спамом;

* ``0``: якщо коментар не є спамом.

.. tip::

    Використовуйте спеціальну адресу електронної пошти — ``akismet-guaranteed-spam@example.com``, щоб змусити результат виклику бути спамом.

Використання змінних середовища
------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Клас ``SpamChecker`` покладається на аргумент ``$akismetKey``. Як і у випадку з каталогом для завантажених файлів, ми можемо ввести його за допомогою налаштування контейнера ``bind``:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -12,6 +12,7 @@ services:
             autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
             bind:
                 string $photoDir: "%kernel.project_dir%/public/uploads/photos"
    +            string $akismetKey: "%env(AKISMET_KEY)%"

         # makes classes in src/ available to be used as services
         # this creates a service per class whose id is the fully-qualified class name

Ми, звичайно, не хочемо жорстко кодувати значення ключа Akismet у файлі конфігурації ``services.yaml``, тому замість нього ми використовуємо змінну середовища (``AKISMET_KEY``).

Надалі кожен розробник має встановити "реальну" змінну середовища або зберегти її значення у файлі ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Для продакшн слід визначити "реальну" змінну середовища.

Це працює добре, але управління багатьма змінними середовища може стати громіздким. На випадок, коли мова заходить про зберігання конфіденційних даних, Symfony має "кращу" альтернативу.

Зберігання конфіденційних даних
------------------------------------------------------------

.. index::
    single: Secret

Замість того щоб використовувати безліч змінних середовища, в Symfony є *vault*, в якому можна зберігати безліч конфіденційних даних. Однією з ключових можливостей є можливість фіксації vault в репозиторії (але без ключа для його відкриття). Ще однією чудовою особливістю є можливість керувати окремим vault у кожному середовищі.

.. index:: ! Command;secrets:set

Конфіденційні дані — це замасковані змінні середовища.

Додайте ключ Akismet у vault:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Оскільки ми вперше запускаємо цю команду, вона згенерувала два ключі в каталозі ``config/secret/dev/``. Далі, у тому ж каталозі, був збережений секретний рядок ``AKISMET_KEY``.

Для розробки, ви можете зафіксувати vault з конфіденційними даними разом з ключами, згенерованими в каталозі ``config/secret/dev/``.

Конфіденційні дані також можна перевизначити, встановивши змінну середовища з тим же ім'ям.

Перевірка коментарів на спам
-----------------------------------------------------

Одним із найпростіших способів перевірки на спам, при відправці нового коментаря, є виклик засобу перевірки на спам перед збереженням даних в базі даних:

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

Перевірте, чи все працює.

Управління конфіденційними даними в продакшн
------------------------------------------------------------------------------------

.. index::
    single: Platform.sh;Secret
    single: Platform.sh;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Для продакшн Platform.sh підтримує налаштування *чутливих змінних середовища*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:AKISMET_KEY --value=abcdef

Але, як обговорювалося вище, використання механізму Symfony для збереження конфіденційних даних може бути кращим варіантом. Не з точки зору безпеки, а з точки зору управління конфіденційними даними командою проекту. Усі конфіденційні дані зберігаються у сховищі, і єдиною змінною середовища, якою потрібно керувати для продакшн, є ключ дешифрування. Це дозволяє будь-якому члену команди додавати конфіденційні дані у продакшн, навіть якщо у нього немає доступу до продакшн серверів. Проте, налаштування цього процесу потребує дещо більшої обізнаності.

.. index::
    single: Command;secrets:generate-keys

По-перше, згенеруйте пару ключів для використання в продакшн:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Повторно додайте секретний рядок Akismet у vault продакшн, але тепер з його продакшн значенням:

.. code-block:: terminal
    :class: answers(abcdef)

    $ symfony console secrets:set AKISMET_KEY --env=prod

Останній крок полягає в тому, щоб відправити ключ дешифрування у Platform.sh, встановивши чутливу змінну:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Ви можете додати й зафіксувати всі файли; файл з ключем дешифрування було автоматично додано у файл ``.gitignore``, тож його ніколи не буде зафіксовано. Для більшої безпеки можна видалити його з вашого локального комп'ютера, якщо проект вже розгорнуто:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Йдемо далі

    * `Документація по компоненту HttpClient`_;

    * `Процесори змінних середовища`_;

    * `Шпаргалка по Symfony HttpClient`_.

.. _`Akismet`: https://akismet.com
.. _`akismet.com`: https://akismet.com
.. _`Документація по компоненту HttpClient`: https://symfony.com/doc/current/components/http_client.html
.. _`Процесори змінних середовища`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Шпаргалка по Symfony HttpClient`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf
