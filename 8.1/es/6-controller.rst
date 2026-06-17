Creando un controlador
======================

.. index::
    single: Controller
    single: Routing;Route

Nuestro proyecto de libro de visitas ya está instalado en los servidores de producción, pero hemos hecho trampas. El proyecto no tiene ninguna página web todavía. La página de inicio consiste en una aburrida página de error 404. Vamos a arreglarlo.

Cuando entra una petición HTTP, como en la página de inicio (``http://localhost:8000/``), Symfony intenta encontrar una *ruta* que coincida con la *ruta de la petición* (``/`` en este caso). Una *ruta* es el enlace entre la ruta de la petición y un fichero *PHP que pueda ser llamado*, una función que crea la *respuesta* HTTP para esa petición.

Estos *scripts* PHP que pueden ser invocados se denominan "controladores". En Symfony, la mayoría de los controladores se implementan como clases PHP. Puedes crear esa clase manualmente, pero como nos gusta ir rápido, veamos cómo nos puede ayudar Symfony.

Siendo perezosos con el Maker Bundle
------------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

Para generar controladores sin esfuerzo, podemos usar el paquete ``symfony/maker-bundle``, que ha sido instalado como parte del paquete ``webapp``.

*Maker bundle* te ayuda a generar muchas clases diferentes. Lo usaremos todo el tiempo en este libro. Cada "generador" se define en un comando y todos los comandos forman parte del espacio de nombres del comando ``make``.

.. index::
    single: Command;list

El comando ``list`` incorporado en la consola de Symfony muestra un listado con todos los comandos disponibles bajo un espacio de nombres dado; utilízalo para descubrir todos los generadores proporcionados por el *bundle* maker:

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

Seleccionando un formato de configuración
------------------------------------------

Antes de crear el primer controlador del proyecto, tenemos que decidir los formatos de configuración que queremos utilizar. Symfony soporta YAML, XML, PHP y atributos de PHP desde el primer momento.

Para la *configuración relacionada con los paquetes*, *YAML* es la mejor opción. Este es el formato utilizado en el directorio ``config/``. A menudo, cuando instales un nuevo paquete, la receta de ese paquete agregará un nuevo archivo que termina en ``.yaml`` en ese directorio.

Para la *configuración relacionada con el código PHP*, los *atributos* son una mejor opción ya que se definen junto al código. Permíteme explicarlo con un ejemplo. Cuando llega una petición, alguna configuración necesita decirle a Symfony que la ruta de la petición debe ser gestionada por un controlador específico (una clase PHP). Cuando se utilizan formatos de configuración YAML, XML o PHP, hay dos archivos implicados (el archivo de configuración y el archivo PHP del controlador). Si se utilizan atributos, la configuración se realiza directamente en la clase de controlador.

Puede que te preguntes cómo puedes adivinar el nombre del paquete que se necesita instalar para una característica. La mayoría de las veces, no necesitas saberlo. En muchos casos, Symfony indica el paquete a instalar en sus mensajes de error. Ejecutar ``symfony console make:message`` sin el paquete ``messenger`` por ejemplo habría terminado con una excepción que contiene una pista sobre la instalación del paquete correcto.

Generando un controlador
------------------------

.. index::
    single: Command;make:controller

Crea tu primer *Controlador* a través del comando ``make:controller``:

.. code-block:: terminal
    :class: answers(no)

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

El comando crea una clase ``ConferenceController`` bajo el directorio ``src/Controller/``. La clase generada contiene código base listo para ser personalizado:

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;

    final class ConferenceController extends AbstractController
    {
        #[Route('/conference', name: 'app_conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

El atributo ``#[Route('/conference', name: 'app_conference')]`` es lo que hace al método ``index()`` un controlador (la configuración está junto al código que configura).

Cuando accedes a ``/conference`` en un navegador, se ejecuta el controlador y se devuelve una respuesta.

Modifica la ruta para que coincida con la página de inicio:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'app_conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

La ruta ``name`` será útil cuando queramos hacer referencia a la página de inicio en el código. En lugar de incluir el camino ``/`` en el código, usaremos el nombre de la ruta.

En lugar de la página mostrada por defecto, devolvamos una página HTML sencilla:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -11,8 +11,13 @@ final class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +            <html>
    +                <body>
    +                    <img src="/images/under-construction.gif" />
    +                </body>
    +            </html>
    +            EOF
    +        );
         }
     }

Actualiza la página en el navegador:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

La responsabilidad principal de un controlador es devolver una ``Response`` (respuesta) HTTP para la petición.

Como el resto del capítulo trata sobre código que no vamos a conservar, hagamos un *commit* con nuestros cambios ahora:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

Agregando un huevo de Pascua
----------------------------

Para demostrar cómo puede aprovechar la información de la solicitud una respuesta, agreguemos un pequeño `huevo de Pascua`_ . Cada vez que la página de inicio contenga una cadena de consulta como ``?hello=Fabien``, añadiremos algún texto para saludar a la persona:

.. code-block:: diff
    :emphasize-lines: 18

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -3,17 +3,24 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        $greet = '';
    +        if ($name = $request->query->get('hello')) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
                 <html>
                     <body>
    +                    $greet
                         <img src="/images/under-construction.gif" />
                     </body>
                 </html>

Symfony expone los datos de la solicitud a través de un objeto ``Request``. Cuando Symfony ve un argumento en el controlador con este tipo determinado (type-hint), automáticamente sabe que debe pasárselo. Podemos usarlo para obtener el elemento ``name`` de la cadena de consulta y añadir un título ``<h1>``.

Intenta acceder a ``/`` y luego a ``/?hello=Fabien`` en un navegador para ver la diferencia.

.. note::

    Nota la llamada a ``htmlspecialchars()`` para evitar problemas de XSS. Esto es algo que se hará automáticamente por nosotros cuando cambiemos a un motor de plantillas en condiciones.

También podríamos haber hecho que el nombre formara parte de la URL:

.. code-block:: diff

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,11 +9,11 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    -    #[Route('/', name: 'homepage')]
    -    public function index(Request $request): Response
    +    #[Route('/hello/{name}', name: 'homepage')]
    +    public function index(string $name = ''): Response
         {
             $greet = '';
    -        if ($name = $request->query->get('hello')) {
    +        if ($name) {
                 $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
             }

La parte ``{name}`` de la ruta es un *parámetro de ruta* dinámico - funciona como un comodín. Ahora puedes acceder a ``/hello`` y luego a ``/hello/Fabien`` en un navegador para obtener los mismos resultados que antes. Puedes obtener el *valor* del parámetro ``{name}`` añadiendo un argumento al controlador con el mismo *nombre*. O sea, ``$name``.

Revierte los cambios que acabamos de hacer:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

Depurando variables
-------------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

Una gran ayuda para la depuración es la función de Symfony ``dump()``. Está siempre disponible y te permite volcar variables complejas en un formato agradable e interactivo.

Cambia ``src/Controller/ConferenceController.php`` temporalmente para mostrar con ``dump()`` el objeto Request:

.. code-block:: diff
    :emphasize-lines: 17

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -3,14 +3,17 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    -    {
    +    public function index(Request $request): Response
    +        {
    +        dump($request);
    +
             return new Response(<<<EOF
                 <html>
                     <body>

Cuando actualices la página, observa el nuevo icono del "punto de mira" que hay en la barra de herramientas; te permite inspeccionar el volcado producido por ``dump()``. Haz clic en él para acceder a una página completa donde la navegación es más simple:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

Revierte los cambios que acabamos de hacer:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: Yendo más allá

    * El sistema de `Routing`_  de Symfony;

    * `Tutorial de SymfonyCasts de rutas, controladores y páginas`_ ;

    * `Atributos de PHP`_ ;

    * El componente `HttpFoundation`_ ;

    * Ataques de seguridad `XSS (Cross-Site Scripting)`_ ;

    * La `Chuleta de Enrutamiento en Symfony`_ .

.. _`huevo de Pascua`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`Routing`: https://symfony.com/doc/current/routing.html
.. _`Tutorial de SymfonyCasts de rutas, controladores y páginas`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`Atributos de PHP`: https://www.php.net/attributes
.. _`HttpFoundation`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS (Cross-Site Scripting)`: https://owasp.org/www-community/attacks/xss/
.. _`Chuleta de Enrutamiento en Symfony`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
