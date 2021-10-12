Expunerea unui API cu API Platform
==================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

Am terminat implementarea site-ului de carte de oaspeți. Pentru a permite o utilizare mai largă a datelor, ce zici de expunerea unui API? O aplicație mobilă poate fi utilizată pentru a afișa toate conferințele, comentariile lor și poate permite participanților să trimită comentarii.

În acest pas, vom implementa API-ul numai în citire.

Instalarea API Platform
-----------------------

Expunerea unui API prin elaborarea codului necesar este posibilă, dar dacă dorim să utilizăm standarde, ar fi bine să folosim o soluție care să se ocupe deja de partea dificilă. O soluție precum API Platform:

.. code-block:: bash

    $ symfony composer req api

Expunerea unui API pentru conferințe
-------------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;@Groups

Câteva adnotări la clasa de Conference este tot ce avem nevoie pentru a configura API-ul:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Conference.php
    +++ b/src/Entity/Conference.php
    @@ -2,16 +2,25 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiResource;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     /**
      * @ORM\Entity(repositoryClass=ConferenceRepository::class)
      * @UniqueEntity("slug")
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="conference:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="conference:item"}}},
    + *     order={"year"="DESC", "city"="ASC"},
    + *     paginationEnabled=false
    + * )
      */
     class Conference
     {
    @@ -19,21 +28,29 @@ class Conference
          * @ORM\Id
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
    +     *
    +     * @Groups({"conference:list", "conference:item"})
          */
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
    +     *
    +     * @Groups({"conference:list", "conference:item"})
          */
         private $city;

         /**
          * @ORM\Column(type="string", length=4)
    +     *
    +     * @Groups({"conference:list", "conference:item"})
          */
         private $year;

         /**
          * @ORM\Column(type="boolean")
    +     *
    +     * @Groups({"conference:list", "conference:item"})
          */
         private $isInternational;

    @@ -44,6 +61,8 @@ class Conference

         /**
          * @ORM\Column(type="string", length=255, unique=true)
    +     *
    +     * @Groups({"conference:list", "conference:item"})
          */
         private $slug;

Principala adnotare ``@ApiResource`` configurează API-ul pentru conferințe. Limitează operațiunile posibile la ``get`` și configurează diverse lucruri: cum ar fi câmpurile de afișat și modul de a ordona conferințele.

În mod implicit, principalul punct de intrare pentru API este ``/api`` datorită configurației din ``config/routes/api_platform.yaml`` care a fost adăugată de rețeta pachetului.

O interfață web îți permite să interacționezi cu API-ul:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

Folosește-o pentru a testa diversele posibilități:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

Imaginează-ți timpul necesar pentru a implementa toate acestea de la zero!

Expunerea unui API pentru comentarii
------------------------------------

.. index::
    single: Annotations;@ApiResource
    single: Annotations;@ApiFilter
    single: Annotations;@Groups

Fă același lucru pentru comentarii:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -2,13 +2,26 @@

     namespace App\Entity;

    +use ApiPlatform\Core\Annotation\ApiFilter;
    +use ApiPlatform\Core\Annotation\ApiResource;
    +use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
     use App\Repository\CommentRepository;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Annotation\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     /**
      * @ORM\Entity(repositoryClass=CommentRepository::class)
      * @ORM\HasLifecycleCallbacks()
    + *
    + * @ApiResource(
    + *     collectionOperations={"get"={"normalization_context"={"groups"="comment:list"}}},
    + *     itemOperations={"get"={"normalization_context"={"groups"="comment:item"}}},
    + *     order={"createdAt"="DESC"},
    + *     paginationEnabled=false
    + * )
    + *
    + * @ApiFilter(SearchFilter::class, properties={"conference": "exact"})
      */
     class Comment
     {
    @@ -16,18 +29,24 @@ class Comment
          * @ORM\Id
          * @ORM\GeneratedValue
          * @ORM\Column(type="integer")
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $id;

         /**
          * @ORM\Column(type="string", length=255)
          * @Assert\NotBlank
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $author;

         /**
          * @ORM\Column(type="text")
          * @Assert\NotBlank
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $text;

    @@ -35,22 +54,30 @@ class Comment
          * @ORM\Column(type="string", length=255)
          * @Assert\NotBlank
          * @Assert\Email
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $email;

         /**
          * @ORM\Column(type="datetime")
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $createdAt;

         /**
          * @ORM\ManyToOne(targetEntity=Conference::class, inversedBy="comments")
          * @ORM\JoinColumn(nullable=false)
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $conference;

         /**
          * @ORM\Column(type="string", length=255, nullable=true)
    +     *
    +     * @Groups({"comment:list", "comment:item"})
          */
         private $photoFilename;

Același tip de adnotări sunt utilizate pentru a configura clasa.

Restrângerea comentariilor expuse de API
-----------------------------------------

În mod implicit, platforma API expune toate intrările din baza de date. Dar pentru comentarii, numai cele publicate ar trebui să facă parte din API.

Atunci cănd este nevoie să restricționezi elementele returnate de API, creează un serviciu care implementează ``QueryCollectionExtensionInterface`` pentru a controla interogarea Doctrine folosită pentru colecții și/sau ``QueryItemExtensionInterface`` pentru a controla articolele:

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 13-15,20-22

    namespace App\Api;

    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $qb, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = [])
        {
            if (Comment::class === $resourceClass) {
                $qb->andWhere(sprintf("%s.state = 'published'", $qb->getRootAliases()[0]));
            }
        }
    }

Clasa de extensie de interogare își aplică logica numai pentru resursa ``Comment`` și modifică constructorul de interogări Doctrine pentru a lua în considerare doar comentariile în starea ``published``.

Configurarea CORS
-----------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

În mod implicit, politica de securitate de aceeași origine a clienților HTTP moderni interzice apelarea API-ului din alte domenii. Bundle-ul CORS, instalat ca parte a ``composer req api``, expediază anteturi de partajare a resurselor cu origine încrucișată pe baza variabilei de mediu ``CORS_ALLOW_ORIGIN``.

În mod implicit, valoarea sa, definită în ``.env``, permite solicitări HTTP de la ``localhost`` și ``127.0.0.1`` pe orice port. Exact acest lucru este necesar pentru următorul pas, deoarece vom crea un SPA care va avea propriul său server web care va apela API-ul.

.. sidebar:: Mergând mai departe

    * `Tutorialul API Platform SymfonyCasts <https://symfonycasts.com/screencast/api-platform>`_;

    * Pentru a activa suportul GraphQL, execută ``compozitorul require webonyx/graphql-php``, apoi accesează ``/api/graphql``.
