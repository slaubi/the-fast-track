Construirea interfeței pentru utilizator
=========================================

.. index::
    single: Twig
    single: Templates

Totul este gata pentru a crea prima versiune a interfeței de utilizare a site-ului. Nu o vom face drăguță, doar funcțională deocamdată.

Îți amintești filtrul din controlerul folosit la surpriză, pentru a evita problemele de securitate (XSS)? Din acest motiv nu vom folosi direct PHP pentru șabloanele noastre. În schimb, vom folosi Twig. Pe lângă gestionarea filtrării textului, `Twig`_ aduce o mulțime de funcții utile pe care le vom folosi, cum ar fi moștenirea (template inheritance).

Instalarea Twig
---------------

Nu trebuie să adăugăm Twig ca dependență, deoarece a fost deja instalat ca o *dependență tranzitivă* a EasyAdmin. Dar dacă decizi să treci la un alt pachet de administrator mai târziu? Unul care folosește un API și un front-end React, de exemplu. Probabil că nu va mai depinde de Twig și astfel Twig va fi eliminat automat atunci când elimini EasyAdmin.

Ca o măsură de precauție, hai să-i spunem lui Composer că proiectul depinde cu adevărat de Twig, independent de EasyAdmin. Adăugarea ca orice altă dependență este suficientă:

.. code-block:: bash

    $ symfony composer req twig

Twig face parte acum din principalele dependențe ale proiectului în ``composer.json``:

.. code-block:: diff
    :class: ignore

    --- a/composer.json
    +++ b/composer.json
    @@ -14,6 +14,7 @@
             "symfony/framework-bundle": "4.4.*",
             "symfony/maker-bundle": "^1.0@dev",
             "symfony/orm-pack": "dev-master",
    +        "symfony/twig-pack": "^1.0",
             "symfony/yaml": "4.4.*"
         },
         "require-dev": {

Utilizarea Twig-ului pentru șabloane
-------------------------------------

.. index::
    single: Twig;Layout
    single: Twig;block

Toate paginile de pe site-ul web vor avea aceeași *machetă*. Când instalezi Twig, un director ``templates/`` va fi creat automat și va adăuga un model de machetă în ``base.html.twig``.

.. code-block:: twig
    :caption: templates/base.html.twig
    :class: ignore

    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="UTF-8">
            <title>{% block title %}Welcome!{% endblock %}</title>
            {% block stylesheets %}{% endblock %}
        </head>
        <body>
            {% block body %}{% endblock %}
            {% block javascripts %}{% endblock %}
        </body>
    </html>

O machetă este definită prin elementele de tip ``block``, un loc unde șabloanele care îl vor moșteni pe cel curent, *șabloanele copil*, *extind* structura machetei și adaugă propriul conținut.

.. index::
    single: Twig;extends
    single: Twig;for

Hai să facem un șablon pentru pagina principală a proiectului în ``templates/conference/index.html.twig``:

.. code-block:: twig
    :caption: templates/conference/index.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook{% endblock %}

    {% block body %}
        <h2>Give your feedback!</h2>

        {% for conference in conferences %}
            <h4>{{ conference }}</h4>
        {% endfor %}
    {% endblock %}

Șablonul *extinde* ``base.html.twig`` și redefinește blocurile ``title`` și ``body``.

.. index::
    single: Twig;Syntax

Notația ``{% %}`` dintr-un șablon indică *acțiuni* și *structuri*.

Notația ``{{ }}`` este folosita pentru a *afișa* valorile variabilelor. ``{{conference}}`` afișează reprezentarea conferinței (rezultatul apelării metodei ``__toString`` din obiectul ``Conference``).

Utilizarea Twig într-un controler
----------------------------------

Actualizează controlerul pentru a reda șablonul Twig:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,24 +2,21 @@

     namespace App\Controller;

    +use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;
    +use Twig\Environment;

     class ConferenceController extends AbstractController
     {
         /**
          * @Route("/", name="homepage")
          */
    -    public function index(): Response
    +    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response(<<<EOF
    -<html>
    -    <body>
    -        <img src="/images/under-construction.gif" />
    -    </body>
    -</html>
    -EOF
    -        );
    +        return new Response($twig->render('conference/index.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
         }
     }

Aici se întâmplă multe.

Pentru a putea reda un șablon, avem nevoie de obiectul ``Environment`` în Twig (punctul principal de intrare în Twig). Observă că solicităm instanța Twig, declarând tipul instanței în metoda controlerului. Symfony este suficient de inteligent pentru a ști să injecteze obiectul potrivit.

De asemenea, avem nevoie de obiectul Repository pentru conferințe pentru a obține toate conferințele din baza de date.

În codul controlerului, metoda ``render()`` redă șablonul și transmite o serie de variabile șablonului. Transmitem lista de obiecte ``Conference`` în variabila ``conferences``.

Un controler este o clasă PHP standard. Nu e necesar să extindem clasa ``AbstractController`` dacă dorim să fim expliciți în privința dependențelor noastre. Poți să îl elimini (dar nu o fă, deoarece vom folosi scurtăturile pe care le oferă în etapele viitoare).

Crearea paginii pentru o conferință
-------------------------------------

Fiecare conferință ar trebui să aibă o pagină dedicată pentru a afișa comentariile. Adăugarea unei noi pagini implică adăugarea unui controler nou, definirea unei rute noi pentru acesta și crearea șablonului aferent.

Adăugă o metodă ``show()`` în ``src/Controller/ConferenceController.php``:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,6 +2,8 @@

     namespace App\Controller;

    +use App\Entity\Conference;
    +use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    @@ -19,4 +21,15 @@ class ConferenceController extends AbstractController
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    +
    +    /**
    +     * @Route("/conference/{id}", name="conference")
    +     */
    +    public function show(Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
    +    {
    +        return new Response($twig->render('conference/show.html.twig', [
    +            'conference' => $conference,
    +            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +        ]));
    +    }
     }

Această metodă are un comportament special pe care nu l-am mai folosit până acum: solicităm injectarea unei instanțe de tip ``Conference`` în metodă. Doar că în baza de date sunt mai multe conferințe. Symfony este capabil să o găsească pe cea dorită pe baza câmpului ``{id}`` furnizat în URL (``id`` fiind cheia primară a tabelului ``conference`` din baza de date).

Preluarea comentariilor legate de conferință se poate face prin metoda ``findBy()`` care primește un criteriu de căutare din argumentul din controler.

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;for
    single: Twig;if
    single: Twig;else
    single: Twig;asset
    single: Twig;format_datetime
    single: Twig;length

Ultimul pas este crearea fișierului ``template/Conference/show.html.twig``:

.. code-block:: twig
    :caption: templates/conference/show.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

    {% block body %}
        <h2>{{ conference }} Conference</h2>

        {% if comments|length > 0 %}
            {% for comment in comments %}
                {% if comment.photofilename %}
                    <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" />
                {% endif %}

                <h4>{{ comment.author }}</h4>
                <small>
                    {{ comment.createdAt|format_datetime('medium', 'short') }}
                </small>

                <p>{{ comment.text }}</p>
            {% endfor %}
        {% else %}
            <div>No comments have been posted yet for this conference.</div>
        {% endif %}
    {% endblock %}

În acest șablon, folosim notația ``|`` pentru a apela *filtrele* Twig. Un filtru transformă o valoare. `` comments|length`` returnează numărul de comentarii și ``comment.createdAt|format_datetime('medium', 'short')`` formatează data pentru a putea fi citită ușor.

Accesează adresa „primei” conferințe prin ``/conference/1`` și observă următoarea eroare:

.. figure:: screenshots/intl-twig-error.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

Eroarea provine din filtrul ``format_datetime``, deoarece nu face parte din nucleul Twig. Mesajul de eroare îți oferă un indiciu despre pachetul care trebuie instalat pentru a remedia problema:

.. code-block:: bash

    $ symfony composer req "twig/intl-extra:^3"

Acum pagina funcționează corect.

Stabilirea legăturilor între pagini
-------------------------------------

.. index::
    single: Twig;Link
    single: Link

Ultimul pas pentru a finaliza prima noastră versiune a interfeței de utilizator este adăugarea legăturilor către paginile conferințelor în pagina principală:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -7,5 +7,8 @@

         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
    +        <p>
    +            <a href="/conference/{{ conference.id }}">View</a>
    +        </p>
         {% endfor %}
     {% endblock %}

Scrierea explicită a unei căi în cod nu e o idee bună, din mai multe motive. Cel mai important este că dacă acea cale se schimbă (de la ``/conference/{id}`` la ``/conferences/{id}`` de exemplu), toate legăturile trebuie actualizate manual.

.. index::
    single: Twig;path

În schimb, utilizând *funcția* ``path()`` Twig și folosește *numele căii* (route name):

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/index.html.twig
    +++ b/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="/conference/{{ conference.id }}">View</a>
    +            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}

Funcția ``path()`` generează calea către o pagină folosind numele rutei. Valorile parametrilor de rută sunt trecute ca o matrice Twig (de tip map).

Paginarea comentariilor
-----------------------

.. index::
    single: Doctrine;Paginator
    single: Paginator

Având mii de participanți, ne putem aștepta la câteva comentarii. Dacă le afișăm pe toate într-o singură pagină, dimensiunea ei va crește foarte repede.

Creează o metodă ``getCommentPaginator()`` în clasa Repository pentru comentarii care returnează un *Paginator* bazat pe o conferință și o decalare (offset - de unde să înceapă):

.. code-block:: diff
    :caption: patch_file

    --- a/src/Repository/CommentRepository.php
    +++ b/src/Repository/CommentRepository.php
    @@ -3,8 +3,10 @@
     namespace App\Repository;

     use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
      * @method Comment|null find($id, $lockMode = null, $lockVersion = null)
    @@ -14,11 +16,27 @@ use Doctrine\Persistence\ManagerRegistry;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    public const PAGINATOR_PER_PAGE = 2;
    +
         public function __construct(ManagerRegistry $registry)
         {
             parent::__construct($registry, Comment::class);
         }

    +    public function getCommentPaginator(Conference $conference, int $offset): Paginator
    +    {
    +        $query = $this->createQueryBuilder('c')
    +            ->andWhere('c.conference = :conference')
    +            ->setParameter('conference', $conference)
    +            ->orderBy('c.createdAt', 'DESC')
    +            ->setMaxResults(self::PAGINATOR_PER_PAGE)
    +            ->setFirstResult($offset)
    +            ->getQuery()
    +        ;
    +
    +        return new Paginator($query);
    +    }
    +
         // /**
         //  * @return Comment[] Returns an array of Comment objects
         //  */

Am stabilit numărul maxim de comentarii pe pagină la 2 pentru a ușura testarea.

Pentru a gestiona paginarea în șablon, transmite Doctrine Paginator către Twig în loc de Doctrine Collection:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -6,6 +6,7 @@ use App\Entity\Conference;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;
     use Twig\Environment;
    @@ -25,11 +26,16 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{id}", name="conference")
          */
    -    public function show(Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
         {
    +        $offset = max(0, $request->query->getInt('offset', 0));
    +        $paginator = $commentRepository->getCommentPaginator($conference, $offset);
    +
             return new Response($twig->render('conference/show.html.twig', [
                 'conference' => $conference,
    -            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +            'comments' => $paginator,
    +            'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,
    +            'next' => min(count($paginator), $offset + CommentRepository::PAGINATOR_PER_PAGE),
             ]));
         }
     }

Controlerul primește decalarea ``(offset)`` din interogarea (``$request->query``) ca un număr întreg (``getInt()``), utilizând 0 implicit dacă altă valoare nu există.

Pașii ``anterior`` și ``următor`` sunt calculați pe baza tuturor informațiilor pe care le avem de la paginator.

.. index::
    single: Twig;if

În cele din urmă, actualizează șablonul pentru a adăuga linkuri la paginile următoare și precedente:

.. code-block:: diff
    :caption: patch_file

    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -6,6 +6,8 @@
         <h2>{{ conference }} Conference</h2>

         {% if comments|length > 0 %}
    +        <div>There are {{ comments|length }} comments.</div>
    +
             {% for comment in comments %}
                 {% if comment.photofilename %}
                     <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" />
    @@ -18,6 +20,13 @@

                 <p>{{ comment.text }}</p>
             {% endfor %}
    +
    +        {% if previous >= 0 %}
    +            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +        {% endif %}
    +        {% if next < comments|length %}
    +            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +        {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}

Acum ar trebui să poți naviga comentariile prin linkurile „Previous” și „Next”:

.. figure:: screenshots/pagination-next.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

.. figure:: screenshots/pagination-previous.png
    :alt: /conference/1?offset=2
    :align: center
    :figclass: with-browser

Refactorizarea controlerului
----------------------------

Este posibil să fi observat că ambele metode din ``ConferenceController`` primesc Twig ca argument. În loc să îl injectăm în fiecare metodă, vom utiliza injectarea dependenței prin constructor (ceea ce face ca lista argumentelor să fie mai scurtă și mai puțin redundantă):

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -13,12 +13,19 @@ use Twig\Environment;

     class ConferenceController extends AbstractController
     {
    +    private $twig;
    +
    +    public function __construct(Environment $twig)
    +    {
    +        $this->twig = $twig;
    +    }
    +
         /**
          * @Route("/", name="homepage")
          */
    -    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
    +    public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($twig->render('conference/index.html.twig', [
    +        return new Response($this->twig->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    @@ -26,12 +33,12 @@ class ConferenceController extends AbstractController
         /**
          * @Route("/conference/{id}", name="conference")
          */
    -    public function show(Request $request, Environment $twig, Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    -        return new Response($twig->render('conference/show.html.twig', [
    +        return new Response($this->twig->render('conference/show.html.twig', [
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,

.. sidebar:: Mergând mai departe

    * `Documentația Twig <https://twig.symfony.com/doc/2.x/>`_;

    * `Crearea și utilizarea șabloanelor <https://symfony.com/doc/current/templates.html>`_ în aplicațiile Symfony;

    * `Tutorial SymfonyCasts Twig <https://symfonycasts.com/screencast/symfony/twig-recipe>`_;

    * `Funcțiile și filtrele Twig disponibile numai în Symfony <https://symfony.com/doc/current/reference/twig_reference.html>`_;

    * `Controlerul de bază AbstractController <https://symfony.com/doc/current/controller.html#the-base-controller-classes-services>`_.

.. _`Twig`: https://twig.symfony.com/
