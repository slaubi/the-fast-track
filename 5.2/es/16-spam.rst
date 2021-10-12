Previniendo spam con una API
============================

.. index::
    single: Spam

Cualquiera puede enviar sus comentarios. Incluso robots, spammers y otros. Podríamos añadir algún "captcha" al formulario para protegerlo de algún modo de los robots, o podemos usar algunas APIs de terceros.

He decidido utilizar el servicio gratuito de `Akismet <https://akismet.com>`_ para demostrar cómo llamar a una API y realizar la comunicación sobre la marcha.

Registrándonos en Akismet
--------------------------

.. index::
    single: Akismet

Regístrate para obtener una cuenta gratuita en `akismet.com <https://akismet.com>`_ y obtén la llave de la API de Akismet.

Dependiendo del componente HTTPClient de Symfony
------------------------------------------------

.. index::
    single: Components;HTTP Client
    single: HTTP Client

En lugar de usar una librería que abstraiga la API de Akismet, haremos todas las llamadas directamente a la API. Realizar las llamadas HTTP por nosotros mismos es más eficiente (y nos permite beneficiarnos de todas las herramientas de depuración de Symfony como la integración con el Symfony Profiler).

Para hacer llamadas a la API, utiliza el componente HttpClient de Symfony:

.. code-block:: bash

    $ symfony composer req http-client

Diseñando una clase verificadora de spam
-----------------------------------------

Crea una nueva clase bajo ``src/`` llamada ``SpamChecker`` para envolver la lógica de llamar a la API de Akismet e interpretar sus respuestas:

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

El método del cliente HTTP ``request()`` envía una petición POST a la URL de Akismet (``$this->endpoint``) y le pasa una serie de parámetros.

El método ``getSpamScore()`` devuelve 3 valores dependiendo de la respuesta de llamada de la API:

* ``2``: Si el comentario es un "blatant spam";

* ``1``: Si el comentario puede ser spam;

* ``0``: Si el comentario no es spam (*ham*).

.. tip::

    Utiliza la dirección de correo electrónico especial ``akismet-guaranteed-spam@example.com`` para forzar que el resultado de la llamada sea spam.

Usando las variables de entorno
-------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

La clase ``SpamChecker`` se basa en un argumento ``$akismetKey``. Al igual que para el directorio de carga, podemos inyectarlo a través de una configuración del contenedor ``bind``:

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

Ciertamente no queremos codificar el valor de la clave de Akismet en el archivo de configuración ``services.yaml``, así que estamos usando una variable de entorno en su lugar (``AKISMET_KEY``).

Corresponde entonces a cada desarrollador establecer una variable de entorno "real" o almacenar el valor en un archivo ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    AKISMET_KEY=abcdef

En el entorno de producción, debe definirse una variable de entorno "real".

Esto funciona bien, pero la gestión de muchas variables de entorno puede resultar engorrosa. En tal caso, Symfony tiene una alternativa "mejor" cuando se trata de almacenar datos secretos.

Almacenando datos secretos
--------------------------

.. index::
    single: Secret

En lugar de usar muchas variables de entorno, Symfony puede administrar un *vault* (bóveda) donde puedes almacenar muchos datos secretos. Una característica clave es la capacidad de poder enviar el *vault* al repositorio (pero sin la llave para abrirlo). Otra gran característica es que puedes gestionar un *vault* por cada entorno.

.. index:: ! Command;secrets:set

Los secretos son variables de entorno disfrazadas.

Añade la llave de Akismet al *vault*:

.. code-block:: bash
    :class: answers(AKISMET_KEY_VALUE)

    $ symfony console secrets:set AKISMET_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "AKISMET_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Como es la primera vez que ejecutamos este comando, generó dos claves en el directorio ``config/secret/dev/``. Luego almacenó el dato secreto ``AKISMET_KEY`` en ese mismo directorio.

Para los datos secretos de desarrollo, puedes decidir enviar al repositorio el *vault* y las claves que se han generado en el directorio ``config/secret/dev/``.

Los datos secretos también pueden ser sobreescritos configurando una variable de entorno con el mismo nombre.

Comprobando comentarios en busca de spam
----------------------------------------

Una forma sencilla de comprobar si hay spam cuando se envía un nuevo comentario es llamar al verificador de spam antes de almacenar los datos en la base de datos:

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

Comprueba que funciona bien.

Manejando los datos secretos en producción
-------------------------------------------

.. index::
    single: SymfonyCloud;Secret
    single: SymfonyCloud;Environment Variable
    single: Secret
    single: Symfony CLI;var:set

Para el entorno de producción, SymfonyCloud soporta la configuración de *variables de entorno sensibles*:

.. code-block:: bash
    :class: ignore

    $ symfony var:set --sensitive AKISMET_KEY=abcdef

Pero como se mencionó anteriormente, usar los datos secretos de Symfony podría ser mejor. No en términos de seguridad, sino de gestión de los datos secretos por parte del equipo de proyecto. Todos los datos secretos se almacenan en el repositorio y la única variable de entorno que necesitas gestionar para producción es la clave de descifrado. Esto hace posible que cualquier persona del equipo pueda añadir datos secretos en producción incluso si no tienen acceso a los servidores de producción. Sin embargo, la configuración es un poco más complicada.

.. index::
    single: Command;secrets:generate-keys

Primero, genera un par de claves para uso en producción:

.. code-block:: bash

    $ APP_ENV=prod symfony console secrets:generate-keys

.. note:

    The ``APP_ENV=prod`` part before the command allows setting the ``APP_ENV`` environment variable only for this command. On Windows, use ``--env=prod`` instead: ``symfony console secrets:generate-keys --env=prod``

.. index::
    single: Command;secrets:set

Vuelve a añadir la clave de Akismet en el *vault* de producción, pero con su valor en producción:

.. code-block:: bash
    :class: answers(abcdef)

    $ APP_ENV=prod symfony console secrets:set AKISMET_KEY

El último paso es enviar la clave de descifrado a SymfonyCloud configurando una variable sensible:

.. code-block:: bash

    $ symfony var:set --sensitive SYMFONY_DECRYPTION_SECRET=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Puedes añadir y hacer commit de todos los archivos; la clave de descifrado se ha añadido automáticamente al archivo ``.gitignore``, por lo que nunca se enviará al repositorio. Para mayor seguridad, puedes quitarla de tu equipo local puesto que ya ha sido desplegado ahora:

.. code-block:: bash

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Yendo más allá

    * La `documentación del componente HttpClient <https://symfony.com/doc/current/components/http_client.html>`_;

    * Los `Procesadores de Variables de Entorno <https://symfony.com/doc/current/configuration/env_var_processors.html>`_;

    * La `Chuleta de Symfony HttpClient <https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/httpclient_en_43.pdf>`_.
