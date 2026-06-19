منع البريد العشوائي باستخدام واجهة برمجة التطبيقات (API)
====================================================================================================

.. index::
    single: Spam

يمكن لأي شخص إرسال ملاحظات. حتى الروبوتات والبريد المزعج وأشياء أخرى. يمكننا إضافة "captcha" إلى النموذج حتى نحمي بطريقة ما من الروبوتات ، أو يمكننا استخدام بعض واجهات برمجة التطبيقات (API) لتباعيات خارجية.

لقد قررت استخدام خدمة `Akismet <https://akismet.com>`_ المجانية _ لتوضيح كيفية الاتصال بواجهة برمجة التطبيقات وكيفية إجراء المكالمة "خارج النطاق".

الاشتراك في Akismet
-----------------------------

.. index::
    single: Akismet

قم بالتسجيل للحصول على حساب مجاني على `akismet.com <https://akismet.com>`_ واحصل على مفتاح واجهة برمجة التطبيقات Akismet.

الاعتماد على مكون Symfony HTTPClient
---------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

بدلاً من استخدام مكتبة تلخص واجهة برمجة تطبيقات Akismet ، سنجري جميع مكالمات واجهة برمجة التطبيقات مباشرة. يعد إجراء مكالمات HTTP بأنفسنا أكثر فعالية (ويسمح لنا بالاستفادة من جميع أدوات تصحيح Symfony مثل التكامل مع Symfony Profiler).

لإجراء مكالمات API ، استخدم Symfony HttpClient Component:

.. code-block:: terminal

    $ symfony composer req http-client

تصميم فئة Spam Checker Class  للبريد المزعج
---------------------------------------------------------------

قم بإنشاء فئة  جديدة تحت `` src / `` باسم `` SpamChecker '' لتضمين منطق استدعاء Akismet API ومعالجة جوابها:

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

ترسل طريقة  `` ()request  `` ل HTTP طلب POST إلى عنوان URL Akismet (`` $ this-> endpoint '') ويمرر مجموعة من المعلمات.

تُظهر طريقة `` getSpamScore()  `` ثلاث  قيم بناءً على استجابة استدعاء API:

* ``2``: إذا كان التعليق "بريدًا عشوائيًا" ؛

* ``1``: إذا كان التعليق غير مرغوب فيه ؛

* ``0``: إذا كان التعليق غير مرغوب فيه.

.. tip::

    استخدم عنوان البريد الإلكتروني الخاص بـ `` akismet-Guarantee-spam@example.com `` لفرض نتيجة المكالمة على أنها بريد مزعج.

استخدام متغيرات البيئة
------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

تعتمد فئة ``SpamChecker`` على خاصية ``akismetKey\``. مثل دليل التحميل ، يمكننا حقنه عبر إعداد حاوية ``bind``:

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

لا نريد بالتأكيد تحديد قيمة مفتاح Akismet في ملف التكوين `` services.yaml '' ، لذا نستخدم متغير بيئة بدلاً من ذلك (`` AKISMET_KEY '').

ومن ثم يعود الأمر لكل مطور لتعيين متغير بيئة "حقيقي" أو لتخزين القيمة في ملف ``env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

للإنتاج Production ، يجب تحديد متغير بيئة "حقيقي".

هذا يعمل بشكل جيد ، ولكن إدارة العديد من متغيرات البيئة قد تصبح مرهقة. في مثل هذه الحالة ، لدى Symfony بديل "أفضل" عندما يتعلق الأمر بتخزين الأسرار.

تخزين الأسرار
-------------------------

.. index::
    single: Secret

بدلاً من استخدام العديد من متغيرات البيئة ، يمكن لـ Symfony إدارة * vault * حيث يمكنك تخزين العديد من الأسرار. إحدى السمات الرئيسية هي القدرة على تنفيذ القبو vault  في المخزن Repository  (ولكن بدون المفتاح لفتحه). ميزة أخرى رائعة هي أنه يمكن إدارة قبو vault واحد لكل بيئة.

.. index:: ! Command;secrets:set

الأسرار هي متغيرات بيئية متخفية.

أضف مفتاح Akismet في الخزنة:

.. code-block:: terminal
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

نظرًا لأن هذه هي المرة الأولى التي نقوم فيها بتشغيل هذا الأمر ، فقد تم إنشاء مفتاحين في دليل ``/config/secret/dev``. ثم قام بتخزين سر ``AKISMET_KEY`` في نفس الدليل.

بالنسبة لأسرار التطوير ، يمكنك أن تقرر تنفيذ الخزنة والمفاتيح التي تم إنشاؤها في دليل ``/config/secret/dev``.

يمكن أيضًا تجاوز الأسرار عن طريق تعيين متغير بيئة يحمل نفس الاسم.

التحقق من التعليقات على البريد المزعج
---------------------------------------------------------------------

تتمثل إحدى الطرق البسيطة للتحقق من البريد العشوائي عند إرسال تعليق جديد في الاتصال بمدقق البريد العشوائي قبل تخزين البيانات في قاعدة البيانات:

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

تحقق من أنه يعمل بشكل جيد.

إدارة الأسرار في الإنتاج Production
--------------------------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

للإنتاج ، يضمن SymfonyCloud إعداد *متغيرات البيئة الحساسة*:

.. code-block:: terminal
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

ولكن كما نُقش أعلاه ، قد يكون استخدام أسرار Symfony أفضل. ليس من حيث الأمن ، ولكن من حيث الإدارة السرية لفريق المشروع. يتم تخزين جميع الأسرار في المخزن Repository  ومتغير البيئة الوحيد الذي تحتاج إلى إدارته للإنتاج هو مفتاح فك التشفير. وهذا يجعل من الممكن لأي شخص في الفريق إضافة أسرار الإنتاج حتى إذا لم يكن لديهم إمكانية الوصول إلى خوادم الإنتاج. التثبيت  أكثر صعوبة  نوعا ما بالرغم من ذلك.

.. index::
    single: Command;secrets:generate-keys

أولاً ، قم بإنشاء زوج من المفاتيح لاستخدام الإنتاج:

.. code-block:: terminal

    $ APP_ENV=prod symfony console secrets:generate-keys

.. note:

    The ``APP_ENV=prod`` part before the command allows setting the ``APP_ENV`` environment variable only for this command. On Windows, use ``--env=prod`` instead: ``symfony console secrets:generate-keys --env=prod``

.. index::
    single: Command;secrets:set

أعد إضافة سر Akismet في قبو الإنتاج ولكن بقيمته للإنتاج:

.. code-block:: terminal
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

الخطوة الأخيرة هي إرسال مفتاح فك التشفير إلى SymfonyCloud عن طريق تعيين متغير حساس:

.. code-block:: terminal

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

يمكنك إضافة جميع الملفات وتنفيذها ؛ تمت إضافة مفتاح فك التشفير في .gitignore تلقائيًا ، لذلك لن يتم الالتزام به أبدًا. لمزيد من الأمان ، يمكنك إزالته من جهازك المحلي حيث تم نشره الآن:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: الذهاب أبعد من ذلك

    * `مستندات Component HttpClient <https://symfony.com/doc/current/components/http_client.html>`_؛

    * `معالجات المتغيرة للبيئة <https://symfony.com/doc/current/configuration/env_var_processors.html>`_؛

    * `ملف الغش الخاص بالـ HttpClient في سيمفوني <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
