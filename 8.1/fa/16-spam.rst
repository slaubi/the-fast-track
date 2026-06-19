جلوگیری از ارسال محتوای هرز (Spam) با کمک یک API
==============================================================================

.. index::
    single: Spam

هر کسی می‌تواند یک بازخورد ارسال کند. حتی ربات‌ها، تولیدکنندگان محتواهای هرز و غیره. می‌توانیم یک «کپچا (captcha)» به فرم بیافزاییم تا در برابر ربات‌ها محافظت شویم یا اینکه از APIهای شخص ثالث استفاده کنیم.

من تصمیم گرفتم تا از سرویس رایگان `Akismet <https://akismet.com>`_ استفاده کنم تا نشان دهم که چگونه می‌توان یک API را فراخوانی کرده و همچنین این فراخوانی را به «خارج از باند (out of band)» منتقل کرد.

ثبت‌نام در Akismet
----------------------------

.. index::
    single: Akismet

یک حساب کاربری رایگان در `akismet.com <https://akismet.com>`_ ایجاد کرده و کلید Akismet API را دریافت نمایید.

تکیه بر کامپوننت HTTPClient سیمفونی
--------------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

به جای استفاده از کتابخانه‌ای که API مربوط به Akismet را انتراعی کند، ما تمام فراخوانی‌های API را به صورت مستقیم انجام می‌دهیم. انجام فراخوانی‌های HTTP توسط خودمان بهینه‌تر است (و اجازه می‌دهد تا از تمام ابزارهای اشکال‌زدایی سیمفونی همچون یکپارچگی با نمایه‌ساز سیمفونی، بهره ببریم).

برای انجام فراخوانی‌های API، از کامپوننت HttpClient سیمفونی استفاده کنید:

.. code-block:: bash

    $ symfony composer req http-client

طراحی یک کلاس بررسی‌کننده‌ی محتوای هرز
-------------------------------------------------------------------------

به منظور جای دادن تمام منطق مربوط به فراخوانی API مربوط به Akismet و تفسیر پاسخ‌های آن، یک کلاس جدید در پوشه‌ی ``src/`` و با نام ``SpamChecker`` ایجاد نمایید:

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

متد ``request()`` در HTTP client، یک درخواست POST را به URL مربوط به Akismet ارسال می‌کند (``$this->endpoint``) و آرایه‌ای از پارامترها را پاس می‌دهد.

متد ``getSpamScore()`` با توجه به پاسخ فراخوانی API، می‌تواند ۳ مقدار مختلف برگرداند:

* ``2``: اگر کامنت آشکارا یک محتوای هرز باشد؛

* ``1``: اگر کامنت امکان هرز بودن داشته باشد؛

* ``0``: اگر کامنت هرز نباشد (ham)؛

.. tip::

    از آدرس رایانامه‌ی مخصوص ``akismet-guaranteed-spam@example.com`` استفاده کنید تا نتیجه‌ی فراخوانی را مجبور به هرز بودن کنید.

استفاده از متغیر‌های محیط
------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

کلاس ``SpamChecker``، به آرگمان ``$akismetKey`` وابسته است. همچون آدرس پوشه‌ی بارگذاری، می‌توانیم آن را از طریق تنظیم ``bind`` در کانتینر، تزریق کنیم:

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

ما مطمئناً نمی‌خواهیم که مقدار کلید Akismet را در فایل پیکربندی ``services.yaml`` هاردکد کنیم، بنابراین به جای آن از یک متغیر محیط استفاده می‌کنیم  (``AKISMET_KEY``).

پس از این هر توسعه‌دهنده وظیفه دارد تا یک متغیر محیط «واقعی» را تنظیم یا مقدار آن را در فایل (``AKISMET_KEY``) ذخیره کند:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

برای محیط عمل‌آوری، باید یک متغیر محیط «واقعی» تعریف شود.

این روش به خوبی کار می‌کند، اما مدیریت تعداد زیادی متغیر محیط ممکن است مایه‌ی زحمت شود. در این صورت سیمفونی برای ذخیره‌ی رمز‌ها، یک راه جایگزین «بهتر» دارد.

ذخیره‌ی رمز‌ها (Secrets)
---------------------------------------

.. index::
    single: Secret

به جای استفاده از تعداد زیادی متغیر محیط، سیمفونی می‌تواند یک *گاوصندوق (vault)* که می‌توانید تعداد زیادی رمز را در آن ذخیره کنید، مدیریت کند. یک ویژگی کلیدی این روش قابلیت commitکردن گاوصندوق در مخزن Git است (اما بدون کلید که آن را باز می‌کند). یک ویژگی عالی دیگر این است که سیمفونی می‌تواند به ازای هر محیط یک گاوصندوق را مدیریت کند.

.. index:: ! Command;secrets:set

رمز‌ها همان متغیرهای محیط در لباس مبدل هستند.

کلید Akismet را به گاوصندوق اضافه کنید:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

چون این اولین باری است که این فرمان را اجرا می‌کنیم، فرمان دو عدد کلید را در پوشه‌ی ``config/secret/dev/`` تولید می‌کند. سپس رمز ``AKISMET_KEY`` را نیز در همان پوشه ذخیره می‌کند.

برای رمز‌های محیط توسعه، می‌توانید تصمیم بگیرید که گاوصندوق را به همراه کلیدهایی که برایش تولید شده و در پوشه‌ی ``config/secret/dev/`` قرار دارد، commit کنید.

همچنین رمزها می‌توانند از طریق تنظیم یک متغیر محیط با نام یکسان، بازنویسی (override) شوند.

بررسی کامنت‌ها برای یافتن محتوای هرز
--------------------------------------------------------------------

هنگامی که یک کامنت جدید ارسال می‌شود، یک راه آسان برای بررسی هرز بودن محتوا، فراخوانی بررسی‌کننده‌ی محتوای هرز (spam checker) قبل از ذخیره‌ی کامنت در پایگاه‌داده است:

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

بررسی کنید که این روش به درستی کار می‌کند.

مدیریت رمز‌ها در محیط عمل‌آوری
----------------------------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

در محیط عمل‌آوری، SymfonyCloud از تنظیم *متغیرهای محیط حساس* پشتیبانی می‌کند:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

اما همانطور که در بالا بحث شد، استفاده از رمزهای سیمفونی می‌تواند راه بهتری باشد. البته نه از لحاظ امنیت، بلکه از نظر سهولت مدیریت رمز برای تیم پروژه. به این ترتیب، تمام رمزها در مخزن ذخیره شده و تنها متغیر محیط که نیاز دارید در محیط عمل‌آوری مدیریت کنید، کلید رمزگشایی است. این روش این امکان را به وجود می‌آورد که هر یک از اعضای تیم بدون آنکه به سرورهای عمل‌آوری دسترسی داشته باشند، بتوانند رمزهای عمل‌آوری اضافه کنند.

.. index::
    single: Command;secrets:generate-keys

ابتدا یک جفت کلید برای استفاده در محیط عمل‌آوری تولید کنید:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

مجدداً رمز Akismet را به گاوصندوق عمل‌آوری اضافه کنید اما با مقدار آن در محیط عمل‌آوری:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

آخرین گام، ارسال کلید رمزگشایی به SymfonyCloud از طریق تنظیم یک متغیر محیط حساس می‌باشد:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

می‌توانید تمام فایل‌ها را اضافه کرده و commit کنید؛ کلید رمزگشایی به صورت خودکار به فایل ``.gitignore`` اضافه شده است، و بنابراین هرگز commit نخواهد شد. برای ایمنی بیشتر، می‌توانید کلید را از روی رایانه‌ی محلی خود پاک نمایید زیرا که دیگر مستقر شده است:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: بیشتر بدانید

    * `مستندات کامپوننت HttpClient <https://symfony.com/doc/current/components/http_client.html>`_؛

    * `پردازشگر متغیرهای محیط <https://symfony.com/doc/current/configuration/env_var_processors.html>`_؛

    * `برگه‌تقلب سیمفونی HttpClient <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
