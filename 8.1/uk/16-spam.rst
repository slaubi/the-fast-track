Запобігання спаму за допомогою ШІ
=================================

.. index::
    single: Spam

Будь-хто може надіслати відгук. Це можуть бути роботи, спамери тощо. Ми можемо додати "капчу" до форми, щоб хоч якось захиститися від роботів, або скористатися API сторонніх розробників.

Я вирішив скористатися великою мовною моделлю (англ. Large Language Model), щоб визначати, чи є коментар спамом, і продемонструвати, як використовувати ШІ в застосунку Symfony та як виконувати такі дорогі виклики поза основним потоком.

Отримання ключа API для ШІ
--------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI підтримує багатьох постачальників моделей: OpenAI, Anthropic, Google Gemini, Mistral і навіть локальні моделі за допомогою Ollama. У цьому розділі використовується OpenAI: зареєструйтеся на `platform.openai.com`_ і створіть ключ API. Якщо ви віддаєте перевагу іншому постачальнику, код залишається тим самим; змінюється лише конфігурація.

Залежність від Symfony AI Bundle
--------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

Замість того щоб самостійно викликати HTTP API моделі, ми використаємо Symfony AI Bundle. Він надає абстракцію *платформи* для постачальників моделей (кожен постачальник постачається як власний пакет-міст) і *агента*, який обгортає модель для виконання викликів; а також користується всіма інструментами налагодження Symfony, як-от інтеграцією з Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI — це молодий набір компонентів, який усе ще є експериментальним: його API можуть розвиватися швидше, ніж решта Symfony.

Рецепт моста OpenAI вже налаштував для нас платформу; він посилається на змінну середовища ``OPENAI_API_KEY`` (і додав для неї порожнє значення за замовчуванням у ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Налаштуйте на її основі *агента* за замовчуванням:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Використання змінних середовища
------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Ми, звичайно, не хочемо жорстко кодувати значення ключа в конфігурації; саме тому воно читається зі змінної середовища ``OPENAI_API_KEY``.

Надалі кожен розробник має встановити "реальну" змінну середовища або зберегти її значення у файлі ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

Для продакшн слід визначити "реальну" змінну середовища.

Це працює добре, але управління багатьма змінними середовища може стати громіздким. На випадок, коли мова заходить про зберігання конфіденційних даних, Symfony має "кращу" альтернативу.

Зберігання конфіденційних даних
------------------------------------------------------------

.. index::
    single: Secret

Замість того щоб використовувати безліч змінних середовища, в Symfony є *vault*, в якому можна зберігати безліч конфіденційних даних. Однією з ключових можливостей є можливість фіксації vault в репозиторії (але без ключа для його відкриття). Ще однією чудовою особливістю є можливість керувати окремим vault у кожному середовищі.

.. index:: ! Command;secrets:set

Конфіденційні дані — це замасковані змінні середовища.

Додайте ключ OpenAI API у vault:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Оскільки ми вперше запускаємо цю команду, вона згенерувала два ключі в каталозі ``config/secret/dev/``. Далі, у тому ж каталозі, був збережений секретний рядок ``OPENAI_API_KEY``.

Для розробки, ви можете зафіксувати vault з конфіденційними даними разом з ключами, згенерованими в каталозі ``config/secret/dev/``.

Конфіденційні дані також можна перевизначити, встановивши змінну середовища з тим же ім'ям.

.. index::
    single: Command;secrets:reveal

Щоб прочитати конфіденційні дані з vault, використовуйте ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Розробка класу перевірки на спам
------------------------------------------------------------

.. index::
    single: AI;Prompt

Створіть новий клас в ``src/`` із назвою ``SpamChecker``, щоб обгорнути логіку запиту до моделі про те, чи є коментар спамом:

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

*Системний промпт* повідомляє моделі її роль і обмежує її відповіді; *повідомлення користувача* містить коментар і контекст його надсилання (IP-адреса, user agent).

Метод ``getSpamScore()`` повертає 3 значення в залежності від відповіді моделі:

* ``2``: якщо коментар є "явним спамом";

* ``1``: якщо коментар може бути спамом, або коли модель недоступна;

* ``0``: якщо коментар не є спамом (ham).

Вивід моделі — це вільний текст, навіть коли промпт його обмежує: розбирайте його вільно (приведіть до нижнього регістру, використовуйте ``str_contains()``). А коли модель узагалі не може відповісти, замість того щоб завершуватися помилкою, повертайтеся до модерації людиною: ШІ має допомагати адміністратору, а не блокувати гостьову книгу.

.. tip::

    Спробуйте надіслати коментар, який виглядає як явний спам, як-от "Buy cheap watches at http://example.com/!!!", щоб побачити модель у дії.

Перевірка коментарів на спам
-----------------------------------------------------

Одним із найпростіших способів перевірки на спам, при відправці нового коментаря, є виклик засобу перевірки на спам перед збереженням даних в базі даних:

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

Перевірте, чи все працює.

Обмеження частоти надсилання коментарів
------------------------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

Виявлення спаму захищає вебсайт від витончених спамерів. Додатковим і значно дешевшим захистом є обмеження того, як швидко той самий клієнт може надсилати коментарі: ніхто легітимно не публікує десятки коментарів на годину в гостьовій книзі.

Додайте компонент Symfony Rate Limiter:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Налаштуйте обмежувач, який приймає щонайбільше 5 коментарів на годину від того самого клієнта:

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

Автоматизовані тести легітимно надсилають багато коментарів за короткий проміжок часу, тож ліміт підвищено для середовища ``test``.

Застосуйте обмежувач до надсилання коментарів за допомогою атрибута ``#[RateLimit]``; за замовчуванням він ідентифікує клієнтів за їхньою IP-адресою:

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

Зверніть увагу на аргумент ``methods``: перегляд сторінки конференції — це запит ``GET``, і його не можна обмежувати; обмежуються лише надсилання коментарів (запити ``POST``).

Коли ліміт досягнуто, Symfony автоматично повертає відповідь ``429 Too Many Requests`` із HTTP-заголовком ``Retry-After``, який повідомляє клієнту, коли він може повторити спробу.

Той самий компонент також захищає форму входу адміністратора від атак перебором; увімкнення *обмеження частоти входу* (англ. login throttling) на firewall займає один рядок:

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

За замовчуванням Symfony блокує IP після 5 невдалих спроб входу з тим самим іменем користувача протягом хвилини (успішний вхід скидає лічильник). Використовуйте параметри ``max_attempts`` та ``interval``, щоб налаштувати політику.

Управління конфіденційними даними в продакшн
------------------------------------------------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Для продакшн Upsun підтримує налаштування *чутливих змінних середовища*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

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

Повторно додайте секретний рядок OpenAI API у vault продакшн, але тепер з його продакшн значенням:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

Останній крок полягає в тому, щоб відправити ключ дешифрування у Upsun, встановивши чутливу змінну:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Ви можете додати й зафіксувати всі файли; файл з ключем дешифрування було автоматично додано у файл ``.gitignore``, тож його ніколи не буде зафіксовано. Для більшої безпеки можна видалити його з вашого локального комп'ютера, якщо проект вже розгорнуто:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Йдемо далі

    * `Документація Symfony AI`_;

    * `Процесори змінних середовища`_;

    * `Як зберігати конфіденційну інформацію`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`Документація Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`Процесори змінних середовища`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Як зберігати конфіденційну інформацію`: https://symfony.com/doc/current/configuration/secrets.html
