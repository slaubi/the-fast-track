Защита от спама с помощью API
==================================================

.. index::
    single: Spam

Отзыв может оставить кто угодно, включая роботов и спамеров. Поэтому, чтобы снизить поток спама, мы можем либо добавить в форму капчу, либо использовать сторонние API.

Я решил использовать бесплатный сервис `Akismet <https://akismet.com>`_, чтобы показать, как можно работать с API и как выполнять внешние запросы.

Регистрация в Akismet
---------------------------------

.. index::
    single: Akismet

Зарегистрируйте бесплатный аккаунт на `akismet.com <https://akismet.com>`_ и получите ключ Akismet API.

Добавление компонента Symfony HTTPClient
------------------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

Мы будем обращаться к API Akismet напрямую вместо использования соответствующей библиотеки для этого. Выполнение HTTP-запросов самостоятельно более эффективно (кроме этого даёт использовать инструменты отладки Symfony, включая профилировщик Symfony).

Чтобы выполнить запрос к API воспользуемся Symfony-компонентом HttpClient:

.. code-block:: bash

    $ symfony composer req http-client

Создание класса для проверки на спам
-------------------------------------------------------------------

В директории ``src/`` создадим новый класс ``SpamChecker``, который будет содержать логику отправки запроса к API Akismet и обработку его ответа:

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

Метод HTTP-клиента ``request()`` отправляет POST-запрос на URL-адрес Akismet (``$this->endpoint``) и передаёт массив параметров.

Метод ``getSpamScore()`` возвращает 3 значения в зависимости от ответа на API-вызов:

* ``2``: если комментарий является явным спамом;

* ``1``: если комментарий может быть спамом;

* ``0``: если комментарий не спам (так называемый ham).

.. tip::

    Используйте специальный адрес электронной почты ``akismet-guaranteed-spam@example.com``, чтобы антиспам-сервис пометил такое сообщение как спам и вы таким образом смогли проверить его работу.

Использование переменных окружения
------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Класс ``SpamChecker`` зависит от параметра ``$akismetKey``. Как и в случае с директорией для загруженных файлов, мы можем переместить ключ в параметр контейнера, используя свойство ``bind``:

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

Разумеется, мы не будем хранить значение ключа Akismet в  конфигурационном файле ``services.yaml``, поэтому используем переменную окружения (``AKISMET_KEY``).

Затем каждый разработчик может сам определить переменную окружения в терминале или хранить ключ в файле ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

Однако в продакшене должна быть определена только "реальная" переменная окружения в терминале.

Это неплохой рабочий вариант, хотя управление множеством переменных окружения может стать обременительным. В таком случае у Symfony есть "лучшая" альтернатива, когда речь заходит о хранении конфиденциальных данных.

Хранение конфиденциальных данных
--------------------------------------------------------------

.. index::
    single: Secret

Вместо использования множества переменных окружения, в Symfony есть *хранилище*, в котором можно поместить много конфиденциальных данных. Среди одной из ключевых особенностей — можно сохранить хранилище в репозитории (но без ключа, чтобы его открыть). Другое замечательное преимущество состоит в том, что для каждого окружения может быть создано собственное хранилище.

.. index:: ! Command;secrets:set

На деле такие конфиденциальные данные являются неявными переменными окружения.

Добавьте ключ Akismet в хранилище:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Поскольку мы запускаем эту команду впервые, она сгенерировала два ключа в директорию ``config/secret/dev/``. Затем эта команда сохранила секретную строку ``AKISMET_KEY`` в этой же директории.

Для хранения конфиденциальных данных в процессе разработки вы можете сохранить в репозитории хранилище вместе с ключами в директории ``config/secret/dev/``.

Все конфиденциальные данные также можно переопределить путём определения одноимённой переменной окружения.

Проверка комментариев на спам
-------------------------------------------------------

При отправке нового комментария одним из простых способов проверить его на спам — вызвать антиспам-сервис перед сохранением данных в базе данных:

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

Убедитесь, что всё работает правильно.

Управление конфиденциальными данными в продакшене
----------------------------------------------------------------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

Для продакшена SymfonyCloud поддерживает установку *конфиденциальных переменных окружения*:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

Однако, как отмечалось выше, использование механизма Symfony для управления конфиденциальными данными может быть более предпочтительным вариантом. Но не с точки зрения безопасности, а в плане управления секретными данными в команде проекта. Поскольку все конфиденциальные данные хранятся в репозитории, то чтобы использовать их в продакшене нужна только специальная переменная окружения с ключом дешифрования. Благодаря такому подходу каждый участник команды может добавить новые защищённые переменные окружения для использования в продакшене, даже если у него нет к нему доступа. Хотя для настройки этого процесса нужно кое-что сделать.

.. index::
    single: Command;secrets:generate-keys

Прежде всего, сгенерируйте пару ключей для использования в продакшене:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Повторно добавьте ключ Akismet в хранилище продакшена, но теперь уже с его действительным значением:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

Последний шаг — отправьте ключ дешифрования в SymfonyCloud, установив специальную для этого переменную:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Можно добавить все файлы в репозиторий, так как файл с ключом дешифрования уже игнорируется в ``.gitignore``, поэтому он никогда не будет зафиксирован. Для большей безопасности лучше удалите его с вашего компьютера, потому что он уже есть в продакшене и больше не понадобится:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Двигаемся дальше

    * `Документация компонента HttpClient <https://symfony.com/doc/current/components/http_client.html>`_;

    * `Процессоры переменных окружения <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * `Шпаргалка по компоненту Symfony HttpClient <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
