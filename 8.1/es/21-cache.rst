Almacenando en caché para mejorar el rendimiento
=================================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Pueden aparecer problemas de rendimiento aparejados a la popularidad. Algunos ejemplos típicos: índices de base de datos que faltan o cientos de peticiones SQL por página. No tendrás ningún problema con una base de datos vacía, pero con más tráfico y datos que no hagan más que crecer, podrían surgir en algún momento.

Agregando cabeceras de caché HTTP
----------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

El uso de estrategias de almacenamiento en caché HTTP es una excelente manera de maximizar el rendimiento para los usuarios finales con poco esfuerzo. Añade una caché de proxy inversa en producción para habilitar el almacenamiento en caché y utiliza una `CDN`_ para acercar el contenido al usuario final y obtener un rendimiento aún mejor.

Vamos a almacenar en caché la página de inicio durante una hora:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -33,9 +33,12 @@ class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/index.html.twig', [
    +        $response = new Response($this->twig->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         #[Route('/conference/{slug}', name: 'conference')]

El método ``setSharedMaxAge()`` configura la expiración de la caché para proxies inversos. Utiliza ``setMaxAge()`` para controlar la caché del navegador. El tiempo se expresa en segundos (1 hora = 60 minutos = 3600 segundos).

El almacenamiento en caché de la página de la conferencia es más difícil, ya que es más dinámico. Cualquiera puede añadir un comentario en cualquier momento, y nadie quiere esperar una hora para verlo en línea. En tales casos, utiliza la estrategia de *validación HTTP* .

Activando el HTTP Cache Kernel de Symfony
-----------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

Para probar la estrategia de caché HTTP, habilita el proxy inverso HTTP de Symfony:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -15,3 +15,5 @@ framework:
         #fragments: true
         php_errors:
             log: true
    +
    +    http_cache: true

Además de ser un proxy inverso HTTP completo, el proxy inverso HTTP de Symfony (a través de la clase ``HttpCache``) añade información de depuración muy útil como cabeceras HTTP. Esto es de gran ayuda cuando se quieren validar las cabeceras de caché que hemos establecido.

Compruébalo en la página de inicio:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

Para la primera petición, el servidor de caché te dice que fue un ``miss`` (fallo) y que realizó un ``store`` (almacenamiento) para almacenar en caché la respuesta. Comprueba la cabecera ``cache-control`` para ver la estrategia de caché configurada.

Para las solicitudes posteriores, la respuesta se almacena en caché (también se ha actualizado ``age`` (edad)):

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

Evitando solicitudes SQL con ESI
--------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

El oyente ``TwigEventSubscriber`` inyecta una variable global en Twig con todos los objetos conferencia. Lo hace para cada una de las páginas del sitio web. Es, con seguridad, un buen candidato a tener en cuenta de cara a la optimización del rendimiento.

No añadirás nuevas conferencias todos los días, así que el código está consultando los mismos datos exactos de la base de datos una y otra vez.

Puede que queramos almacenar en caché los nombres de las conferencias y los slugs con Symfony Cache, pero siempre que sea posible me gusta confiar en la infraestructura de almacenamiento en caché HTTP.

Cuando desees almacenar en caché un fragmento de una página, desplázalo fuera de la petición HTTP actual creando una *subpetición*. *ESI* es una combinación perfecta para este caso de uso. Un ESI es una forma de incrustar el resultado de una petición HTTP en otra.

Crea un controlador que sólo devuelva el fragmento HTML que muestra las conferencias:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -41,6 +41,14 @@ class ConferenceController extends AbstractController
             return $response;
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return new Response($this->twig->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
    +    }
    +
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository, string $photoDir): Response
         {

Crea la plantilla correspondiente:

.. code-block:: twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Accede a ``/conference_header`` para comprobar que todo funciona correctamente.

.. index::
    single: Twig;render
    single: Twig;path

¡Es hora de revelar el truco! Actualiza el layout de Twig para llamar al controlador que acabamos de crear:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -16,11 +16,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Y *voilà* . Actualiza la página y el sitio web sigue mostrando lo mismo.

.. tip::

    Utiliza el panel "Request / Response" del profiler de Symfony para obtener más información sobre la petición principal y sus subpeticiones.

Ahora, cada vez que visites una página en el navegador, se ejecutan dos peticiones HTTP, una para el encabezado y otra para la página principal. Has empeorado el rendimiento. ¡Felicidades!

La llamada HTTP del encabezado de la conferencia es actualmente realizada internamente por Symfony, por lo que no se trata de una llamada de ida y vuelta HTTP. Esto también significa que no hay manera de beneficiarse de las cabeceras de caché HTTP.

Convierte la llamada a una llamada HTTP "real" utilizando un ESI.

Primero, habilita el soporte de ESI:

.. code-block:: diff
    :caption: patch_file

    --- a/config/packages/framework.yaml
    +++ b/config/packages/framework.yaml
    @@ -11,7 +11,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

Luego, usa ``render_esi`` en lugar de ``render``:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/base.html.twig
    +++ b/templates/base.html.twig
    @@ -16,7 +16,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

Si Symfony detecta un proxy inverso que sabe cómo tratar con ESIs, habilitará el soporte automáticamente (si no, volverá al comportamiento de realizar la subsolicitud de forma síncrona).

Como el proxy inverso de Symfony soporta ESIs, vamos a comprobar sus registros (primero elimina la caché - ver "Purgar" más abajo):

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

Recarga la página un par de veces: la respuesta ``/`` se almacena en caché y la``/conference_header`` no. Hemos logrado algo maravilloso: tener toda la página en la caché pero con una parte dinámica.

Sin embargo, esto no es lo que queremos. Vamos a incluir en la caché la página de cabecera durante una hora, independientemente de todo lo demás:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -44,9 +44,12 @@ class ConferenceController extends AbstractController
         #[Route('/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($this->twig->render('conference/header.html.twig', [
    +        $response = new Response($this->twig->render('conference/header.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
    +        $response->setSharedMaxAge(3600);
    +
    +        return $response;
         }

         #[Route('/conference/{slug}', name: 'conference')]

La caché está ahora habilitada para ambas peticiones:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

La cabecera ``x-symfony-cache`` contiene dos elementos: la solicitud principal ``/`` y una subpetición (la ESI ``conference_header``). Ambos están en la caché (``fresh``).

La estrategia de caché puede ser diferente de la página principal y sus ESIs. Si tenemos una página "Acerca de", es posible que queramos almacenarla durante una semana en la caché, y que la cabecera se actualice cada hora.

Quita al oyente ya que no lo necesitamos más:

.. code-block:: terminal

    $ rm src/EventSubscriber/TwigEventSubscriber.php

Purgando la caché HTTP para pruebas
------------------------------------

Probar el sitio web en un navegador o mediante pruebas automatizadas se hace un poco más difícil con una capa de caché.

You can manually remove all the HTTP cache by removing the
``var/cache/dev/http_cache/`` directory:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Annotations;Route

Esta estrategia no funciona bien si sólo deseas invalidar algunas URL o si deseas integrar la invalidación de caché en tus pruebas funcionales. Añadamos un pequeño endpoint HTTP, sólo para administradores, para invalidar algunas URL:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -6,8 +6,10 @@ use App\Entity\Comment;
     use App\Message\CommentMessage;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Bundle\FrameworkBundle\HttpCache\HttpCache;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
    @@ -52,4 +54,17 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]);
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store = (new class($kernel) extends HttpCache {})->getStore();
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

El nuevo controlador se ha restringido al método HTTP ``PURGE``. Este método no se encuentra en el estándar HTTP, pero se utiliza ampliamente para invalidar las cachés.

Por defecto, los parámetros de ruta no pueden contener ``/`` , ya que separan los segmentos de URL. Puedes anular esta restricción para el último parámetro de ruta, por ejemplo ``uri``, configurando tu propio patrón de requisitos (``.*``).

La forma en que obtenemos la instancia ``HttpCache`` también puede parecer un poco extraña; estamos usando una clase anónima ya que no es posible acceder a la "real". La instancia ``HttpCache`` envuelve el kernel real, que no conoce la capa de caché como debería ser.

Invalida la página de inicio y el encabezado de la conferencia a través de las siguientes llamadas cURL:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`/admin/http-cache/conference_header

El subcomando ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` devuelve la URL actual del servidor web local.

.. note::

    El controlador no tiene un nombre de ruta ya que nunca será referenciado en el código.

Agrupando rutas similares con un prefijo
----------------------------------------

.. index::
    single: Annotations;Route

Las dos rutas en el controlador de administración tienen el mismo prefijo ``/admin``. En lugar de repetirlo en todas las rutas, refactoriza las rutas para configurar el prefijo en la propia clase:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/AdminController.php
    +++ b/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Annotation\Route;
     use Symfony\Component\Workflow\Registry;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         private $twig;
    @@ -28,7 +29,7 @@ class AdminController extends AbstractController
             $this->bus = $bus;
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, Registry $registry): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -55,7 +56,7 @@ class AdminController extends AbstractController
             ]);
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

Almacenando en la caché operaciones intensivas de CPU/memoria
--------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

No tenemos en nuestro sitio web código que sea intensivo en el uso de CPU o de memoria. Con el fin de hablar de *cachés locales*, vamos a crear un comando que muestre el paso actual en el que estamos trabajando (para ser más precisos, el nombre de la etiqueta Git adjuntada al commit actual de Git).

El componente Process de Symfony te permite ejecutar un comando y recuperar el resultado (salida estándar y de error); instálalo:

.. code-block:: terminal

    $ symfony composer req process

Implementa el comando:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    class StepInfoCommand extends Command
    {
        protected static $defaultName = 'app:step:info';

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return 0;
        }
    }

.. index::
    single: Command;make:command

.. note::

    Podrías haber usado ``make:command`` para crear el comando:

    .. code-block:: terminal
        :class: ignore

        $ symfony console make:command app:step:info

.. index::
    single: Cache
    single: Components;Cache

¿Qué pasa si queremos almacenar en caché la salida durante unos minutos? Utiliza la Cache de Symfony:

.. code-block:: terminal

    $ symfony composer req cache

Y envuelve el código con la lógica de la caché:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Command/StepInfoCommand.php
    +++ b/src/Command/StepInfoCommand.php
    @@ -6,16 +6,31 @@ use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Input\InputInterface;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     class StepInfoCommand extends Command
     {
         protected static $defaultName = 'app:step:info';

    +    private $cache;
    +
    +    public function __construct(CacheInterface $cache)
    +    {
    +        $this->cache = $cache;
    +
    +        parent::__construct();
    +    }
    +
         protected function execute(InputInterface $input, OutputInterface $output): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $this->cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return 0;
         }

Ahora el proceso solo se llama si el elemento ``app.current_step`` no está en la caché.

Analizando y comparando el rendimiento
--------------------------------------

Nunca añadas caché a ciegas. Ten en cuenta que añadir algo de caché añade una capa de complejidad. Y como todos somos muy malos adivinando lo que será rápido y lo que es lento, puedes terminar en una situación en la que la caché hace que tu aplicación sea más lenta.

Siempre mide el impacto de añadir una caché con una herramienta de análisis como `Blackfire <https://blackfire.io/>`_.

Consulta el paso "Rendimiento" para obtener más información sobre cómo puedes utilizar Blackfire para probar tu código antes de la implementación.

Configurando una caché de proxy inverso en producción
-------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: SymfonyCloud;Varnish
    single: Varnish

No utilices el proxy inverso de Symfony en producción. Opta siempre por un proxy inverso como Varnish en tu infraestructura o una CDN comercial.

Agrega Varnish a los servicios SymfonyCloud:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/services.yaml
    +++ b/.symfony/services.yaml
    @@ -2,3 +2,12 @@ db:
         type: postgresql:13
         disk: 1024
         size: S
    +
    +varnish:
    +    type: varnish:6.0
    +    relationships:
    +        application: 'app:http'
    +    configuration:
    +        vcl: !include
    +            type: string
    +            path: config.vcl

.. index::
    single: SymfonyCloud;Routes

Utiliza Varnish como el principal punto de entrada en las rutas:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,2 @@
    -"https://{all}/": { type: upstream, upstream: "app:http" }
    +"https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Finalmente, crea un archivo ``config.vcl`` para configurar Varnish:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Habilitando el soporte de ESI en Varnish
----------------------------------------

El soporte de ESI en Varnish debe estar habilitado explícitamente para cada solicitud. Para hacerlo universal, Symfony utiliza el estándar ``Surrogate-Capability`` y los encabezados ``Surrogate-Control`` para negociar el soporte de ESI:

.. code-block:: vcl
    :caption: .symfony/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

Purgando la caché de Varnish
-----------------------------

Invalidar la caché en producción probablemente nunca deba ser necesario, excepto con fines de emergencia y tal vez en las ramas no ``master``. Si necesitas purgar la caché con frecuencia, probablemente significa que la estrategia de almacenamiento en caché debe ser modificada (bajando el TTL o usando una estrategia de validación en lugar de una de caducidad).

De todos modos, veamos cómo configurar Varnish para la invalidación de la caché:

.. code-block:: diff
    :caption: patch_file

    --- a/.symfony/config.vcl
    +++ b/.symfony/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

En la vida real, probablemente restrinjas por IPs como se describe en la `documentación de Varnish <https://varnish-cache.org/docs/trunk/users-guide/purging.html>`_.

Purga algunas URLs ahora:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`
    $ curl -X PURGE -H 'x-purge-token PURGE_NOW' `symfony env:urls --first`conference_header

Las URLs se ven un poco extrañas porque las URLs devueltas por ``env:urls`` ya terminan con ``/`` .

.. sidebar:: Yendo más allá

    * `Cloudflare <https://www.cloudflare.com>`_ , la plataforma global de nube;

    * `Documentación de Varnish HTTP Cache <https://varnish-cache.org/docs/index.html>`_;

    * `Especificaciones de ESI <https://www.w3.org/TR/esi-lang>`_ y `recursos para desarrolladores de ESI <https://www.akamai.com/us/en/support/esi.jsp>`_ ;

    * `Modelo de validación de caché HTTP <https://symfony.com/doc/current/http_cache/validation.html>`_ ;

    * `Caché HTTP en SymfonyCloud <https://symfony.com/doc/current/cloud/cookbooks/cache.html>`_ .

.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
