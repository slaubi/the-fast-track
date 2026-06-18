Защита от спама с помощью ИИ
============================

.. index::
    single: Spam

Отзыв может оставить кто угодно, включая роботов и спамеров. Поэтому, чтобы снизить поток нежелательных сообщений, можно добавить в форму "капчу" или воспользоваться сторонними API.

Я решил доверить решение о том, является ли комментарий спамом, большой языковой модели (LLM), чтобы показать, как использовать ИИ в приложении Symfony и как выполнять такие дорогостоящие вызовы "вне основного потока".

Получение API-ключа для ИИ
--------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI поддерживает множество поставщиков моделей: OpenAI, Anthropic, Google Gemini, Mistral и даже локальные модели через Ollama. В этой главе используется OpenAI: зарегистрируйтесь на `platform.openai.com`_ и создайте API-ключ. Если вы предпочитаете другого поставщика, код остаётся тем же; меняется только конфигурация.

Добавление бандла Symfony AI
----------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Вместо того чтобы самим вызывать HTTP API модели, мы воспользуемся бандлом Symfony AI. Он предоставляет абстракцию *платформы* для поставщиков моделей (каждый поставщик поставляется отдельным пакетом-мостом) и *агента*, который оборачивает модель для выполнения вызовов; а ещё он пользуется всеми инструментами отладки Symfony, включая интеграцию с профилировщиком Symfony:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI — это молодой набор компонентов, всё ещё экспериментальный: его API могут меняться быстрее, чем остальная часть Symfony.

Рецепт моста OpenAI уже настроил платформу за нас; он ссылается на переменную окружения ``OPENAI_API_KEY`` (и добавил для неё пустое значение по умолчанию в ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Настройте поверх неё *агента* по умолчанию:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Использование переменных окружения
----------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Разумеется, мы не будем хранить значение ключа в конфигурации, поэтому оно читается из переменной окружения ``OPENAI_API_KEY``.

Затем каждый разработчик может сам определить переменную окружения в терминале или хранить ключ в файле ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

Однако в продакшене должна быть определена только "реальная" переменная окружения в терминале.

Это неплохой рабочий вариант, хотя управление множеством переменных окружения может стать обременительным. В таком случае у Symfony есть "лучшая" альтернатива, когда речь заходит о хранении конфиденциальных данных.

Хранение конфиденциальных данных
--------------------------------

.. index::
    single: Secret

Вместо использования множества переменных окружения, в Symfony есть *хранилище*, в котором можно поместить много конфиденциальных данных. Среди одной из ключевых особенностей — можно сохранить хранилище в репозитории (но без ключа, чтобы его открыть). Другое замечательное преимущество состоит в том, что для каждого окружения может быть создано собственное хранилище.

.. index:: ! Command;secrets:set

На деле такие конфиденциальные данные являются неявными переменными окружения.

Добавьте API-ключ OpenAI в хранилище:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Поскольку мы запускаем эту команду впервые, она сгенерировала два ключа в директорию ``config/secret/dev/``. Затем эта команда сохранила секретную строку ``OPENAI_API_KEY`` в этой же директории.

Для хранения конфиденциальных данных в процессе разработки вы можете сохранить в репозитории хранилище вместе с ключами в директории ``config/secret/dev/``.

Все конфиденциальные данные также можно переопределить путём определения одноимённой переменной окружения.

.. index::
    single: Command;secrets:reveal

Чтобы прочитать секрет из хранилища, используйте ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Создание класса для проверки на спам
------------------------------------

.. index::
    single: AI;Prompt

В директории ``src/`` создадим новый класс ``SpamChecker``, который будет содержать логику запроса к модели о том, является ли комментарий спамом:

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

*Системный промпт* сообщает модели её роль и ограничивает её ответы; *пользовательское сообщение* содержит комментарий и контекст его отправки (IP-адрес, user agent).

Метод ``getSpamScore()`` возвращает 3 значения в зависимости от ответа модели:

* ``2``: если комментарий — явный спам ("blatant spam");

* ``1``: если комментарий может быть спамом, или когда модель недоступна;

* ``0``: если комментарий не является спамом (ham).

Ответ модели — это произвольный текст, даже когда промпт его ограничивает: разбирайте его с запасом прочности (приводите к нижнему регистру, используйте ``str_contains()``). А когда модель совсем не может ответить, переходите к модерации человеком вместо того, чтобы завершаться с ошибкой: ИИ должен помогать администратору, а не блокировать гостевую книгу.

.. tip::

    Попробуйте отправить комментарий, который выглядит как явный спам, например "Buy cheap watches at http://example.com/!!!", чтобы увидеть модель в действии.

Проверка комментариев на спам
-----------------------------

При отправке нового комментария одним из простых способов проверить его на спам — вызвать класс-проверщик перед сохранением данных в базу данных:

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

Убедитесь, что всё работает правильно.

Ограничение частоты отправки комментариев
-----------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Обнаружение спама защищает сайт от изощрённых спамеров. Дополнительная и гораздо более дешёвая защита — ограничить скорость, с которой один и тот же клиент может отправлять комментарии: никто не оставляет в гостевой книге десятки комментариев в час с честными намерениями.

Добавьте компонент Symfony Rate Limiter:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Настройте ограничитель, принимающий не более 5 комментариев в час от одного клиента:

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

Автоматические тесты по понятным причинам отправляют много комментариев за короткое время, поэтому для окружения ``test`` лимит увеличен.

Примените ограничитель к отправке комментариев с помощью атрибута ``#[RateLimit]``; по умолчанию он идентифицирует клиентов по их IP-адресу:

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

Обратите внимание на аргумент ``methods``: просмотр страницы конференции — это ``GET``-запрос, и его нельзя ограничивать; ограничиваются только отправки комментариев (``POST``-запросы).

При достижении лимита Symfony автоматически возвращает ответ ``429 Too Many Requests`` с HTTP-заголовком ``Retry-After``, сообщающим клиенту, когда можно повторить попытку.

Этот же компонент защищает форму входа администратора от атак перебором; включение *троттлинга входа* на файрволе занимает одну строку:

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

По умолчанию Symfony блокирует IP после 5 неудачных попыток входа с одним и тем же именем пользователя в течение минуты (успешный вход сбрасывает счётчик). Используйте опции ``max_attempts`` и ``interval``, чтобы настроить политику.

Управление конфиденциальными данными в продакшене
-------------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Для продакшена Upsun поддерживает установку *конфиденциальных переменных окружения*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

Однако, как отмечалось выше, использование механизма Symfony для управления конфиденциальными данными может быть более предпочтительным вариантом. Но не с точки зрения безопасности, а в плане управления секретными данными в команде проекта. Поскольку все конфиденциальные данные хранятся в репозитории, то чтобы использовать их в продакшене нужна только специальная переменная окружения с ключом дешифрования. Благодаря такому подходу каждый участник команды может добавить новые защищённые переменные окружения для использования в продакшене, даже если у него нет к нему доступа. Хотя для настройки этого процесса нужно кое-что сделать.

.. index::
    single: Command;secrets:generate-keys

Прежде всего, сгенерируйте пару ключей для использования в продакшене:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    On Linux and similiar OSes, use ``APP_RUNTIME_ENV=prod`` instead of ``--env=prod`` as this avoids compiling the application for the ``prod`` environment:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Повторно добавьте API-ключ OpenAI в хранилище продакшена, но теперь уже с его действительным значением:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

Последний шаг — отправьте ключ дешифрования в Upsun, установив специальную для этого переменную:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Можно добавить все файлы в репозиторий, так как файл с ключом дешифрования уже игнорируется в ``.gitignore``, поэтому он никогда не будет зафиксирован. Для большей безопасности лучше удалите его с вашего компьютера, потому что он уже есть в продакшене и больше не понадобится:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Двигаемся дальше

    * `Документация Symfony AI`_;

    * `Процессоры переменных окружения`_;

    * `Как сохранить конфиденциальную информацию в секрете`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Документация Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`Процессоры переменных окружения`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Как сохранить конфиденциальную информацию в секрете`: https://symfony.com/doc/current/configuration/secrets.html
