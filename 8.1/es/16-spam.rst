Previniendo spam con IA
=======================

.. index::
    single: Spam

Cualquiera puede enviar sus comentarios. Incluso robots, spammers y otros. Podríamos añadir algún "captcha" al formulario para protegerlo de algún modo de los robots, o podemos usar algunas APIs de terceros.

He decidido utilizar un modelo de lenguaje grande (LLM) para decidir si un comentario es spam, con el fin de demostrar cómo usar la IA en una aplicación Symfony y cómo realizar este tipo de llamadas costosas "fuera de banda".

Obteniendo una clave de API de IA
---------------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI soporta muchos proveedores de modelos: OpenAI, Anthropic, Google Gemini, Mistral e incluso modelos locales a través de Ollama. Este capítulo usa OpenAI: regístrate en `platform.openai.com`_ y crea una clave de API. Si prefieres otro proveedor, el código se mantiene igual; solo cambia la configuración.

Dependiendo del Symfony AI Bundle
---------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

En lugar de llamar nosotros mismos a la API HTTP del modelo, usaremos el Symfony AI Bundle. Proporciona una abstracción de *platform* para los proveedores de modelos (cada proveedor viene como su propio paquete puente) y un *agent* que envuelve un modelo para hacer las llamadas; y se beneficia de todas las herramientas de depuración de Symfony, como la integración con el Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI es un conjunto de componentes joven y todavía experimental: sus APIs pueden evolucionar más rápido que el resto de Symfony.

La receta del puente de OpenAI ya ha configurado la plataforma por nosotros; hace referencia a una variable de entorno ``OPENAI_API_KEY`` (y añadió un valor predeterminado vacío para ella en ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

Configura un *agent* predeterminado sobre ella:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

Usando las variables de entorno
-------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Ciertamente no queremos codificar el valor de la clave en la configuración; por eso se lee de la variable de entorno ``OPENAI_API_KEY``.

Corresponde entonces a cada desarrollador establecer una variable de entorno "real" o almacenar el valor en un archivo ``.env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

En el entorno de producción, debe definirse una variable de entorno "real".

Esto funciona bien, pero la gestión de muchas variables de entorno puede resultar engorrosa. En tal caso, Symfony tiene una alternativa "mejor" cuando se trata de almacenar datos secretos.

Almacenando datos secretos
--------------------------

.. index::
    single: Secret

En lugar de usar muchas variables de entorno, Symfony puede administrar un *vault* (bóveda) donde puedes almacenar muchos datos secretos. Una característica clave es la capacidad de poder enviar el *vault* al repositorio (pero sin la llave para abrirlo). Otra gran característica es que puedes gestionar un *vault* por cada entorno.

.. index:: ! Command;secrets:set

Los secretos son variables de entorno disfrazadas.

Añade la clave de API de OpenAI al *vault*:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

Como es la primera vez que ejecutamos este comando, generó dos claves en el directorio ``config/secret/dev/``. Luego almacenó el dato secreto ``OPENAI_API_KEY`` en ese mismo directorio.

Para los datos secretos de desarrollo, puedes decidir enviar al repositorio el *vault* y las claves que se han generado en el directorio ``config/secret/dev/``.

Los datos secretos también pueden ser sobreescritos configurando una variable de entorno con el mismo nombre.

.. index::
    single: Command;secrets:reveal

Para volver a leer un secreto del *vault*, usa ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

Diseñando una clase verificadora de spam
-----------------------------------------

.. index::
    single: AI;Prompt

Crea una nueva clase bajo ``src/`` llamada ``SpamChecker`` para envolver la lógica de preguntar al modelo si un comentario es spam:

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

El *system prompt* le indica al modelo su rol y restringe sus respuestas; el *user message* contiene el comentario y su contexto de envío (dirección IP, user agent).

El método ``getSpamScore()`` devuelve 3 valores dependiendo de la respuesta del modelo:

* ``2``: Si el comentario es un "blatant spam" (spam descarado);

* ``1``: Si el comentario puede ser spam, o cuando no se puede contactar con el modelo;

* ``0``: Si el comentario no es spam (*ham*).

La salida de un modelo es texto libre, incluso cuando el prompt la restringe: analízala de forma flexible (pásala a minúsculas, usa ``str_contains()``). Y cuando el modelo no puede responder en absoluto, recurre a la moderación humana en lugar de fallar: la IA debe ayudar al administrador, nunca bloquear el libro de visitas.

.. tip::

    Intenta enviar un comentario que parezca claramente spam, como "Buy cheap watches at http://example.com/!!!", para ver el modelo en acción.

Comprobando comentarios en busca de spam
----------------------------------------

Una forma sencilla de comprobar si hay spam cuando se envía un nuevo comentario es llamar al verificador de spam antes de almacenar los datos en la base de datos:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,7 +7,8 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,8 +35,9 @@ final class ConferenceController extends AbstractController
             Request $request,
             #[MapEntity(mapping: ['slug' => 'slug'])]
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

Comprueba que funciona bien.

Limitando la frecuencia de envío de comentarios
-----------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

La detección de spam protege el sitio web contra los spammers sofisticados. Una protección complementaria y mucho más barata es limitar la velocidad a la que un mismo cliente puede enviar comentarios: nadie publica legítimamente decenas de comentarios por hora en un libro de visitas.

Añade el componente Symfony Rate Limiter:

.. code-block:: terminal

    $ symfony composer req rate-limiter

Configura un limitador que acepte como máximo 5 comentarios por hora del mismo cliente:

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

Las pruebas automatizadas envían legítimamente muchos comentarios en un corto periodo de tiempo, por lo que el límite se eleva para el entorno ``test``.

Aplica el limitador a los envíos de comentarios con el atributo ``#[RateLimit]``; por defecto, identifica a los clientes por su dirección IP:

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
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(
             Request $request,

Fíjate en el argumento ``methods``: navegar por la página de una conferencia es una petición ``GET`` y no debe limitarse; solo se limitan los envíos de comentarios (peticiones ``POST``).

Cuando se alcanza el límite, Symfony devuelve automáticamente una respuesta ``429 Too Many Requests`` con una cabecera HTTP ``Retry-After`` que le indica al cliente cuándo puede reintentar.

El mismo componente también protege el formulario de inicio de sesión del administrador contra los ataques de fuerza bruta; activar el *login throttling* en el firewall requiere una línea:

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

Por defecto, Symfony bloquea una IP tras 5 intentos fallidos de inicio de sesión sobre el mismo nombre de usuario en un minuto (un inicio de sesión correcto reinicia el contador). Usa las opciones ``max_attempts`` e ``interval`` para ajustar la política.

Manejando los datos secretos en producción
-------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

Para el entorno de producción, Upsun soporta la configuración de *variables de entorno sensibles*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

Pero como se mencionó anteriormente, usar los datos secretos de Symfony podría ser mejor. No en términos de seguridad, sino de gestión de los datos secretos por parte del equipo de proyecto. Todos los datos secretos se almacenan en el repositorio y la única variable de entorno que necesitas gestionar para producción es la clave de descifrado. Esto hace posible que cualquier persona del equipo pueda añadir datos secretos en producción incluso si no tienen acceso a los servidores de producción. Sin embargo, la configuración es un poco más complicada.

.. index::
    single: Command;secrets:generate-keys

Primero, genera un par de claves para uso en producción:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note:

    En Linux y sistemas operativos similares, usa ``APP_RUNTIME_ENV=prod`` en lugar de ``--env=prod`` ya que esto evita compilar la aplicación para el entorno ``prod``:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

Vuelve a añadir la clave de API de OpenAI en el *vault* de producción, pero con su valor en producción:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

El último paso es enviar la clave de descifrado a Upsun configurando una variable sensible:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

Puedes añadir y hacer commit de todos los archivos; la clave de descifrado se ha añadido automáticamente al archivo ``.gitignore``, por lo que nunca se enviará al repositorio. Para mayor seguridad, puedes quitarla de tu equipo local puesto que ya ha sido desplegado ahora:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: Yendo más allá

    * La `documentación de Symfony AI`_ ;

    * Los `Procesadores de Variables de Entorno`_ ;

    * `Cómo mantener en secreto la información sensible`_ .

.. _`platform.openai.com`: https://platform.openai.com
.. _`documentación de Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`Procesadores de Variables de Entorno`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Cómo mantener en secreto la información sensible`: https://symfony.com/doc/current/configuration/secrets.html
