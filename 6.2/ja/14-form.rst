フォームでフィードバックを受ける
================================================

.. index::
    single: Components;Form
    single: Form

カンファレンスの参加者からフィードバックをしてもらうようにしましょう。 *HTML フォーム* からコメントを投稿できるようにしましょう。

フォームタイプを生成する
------------------------------------

.. index::
    single: Command;make:form

Maker バンドルを使ってフォームクラスを生成します:

.. code-block:: terminal

    $ symfony console make:form CommentFormType Comment

.. code-block:: text
    :class: ignore
    :emphasize-lines: 1

     created: src/Form/CommentFormType.php


      Success!


     Next: Add fields to your form and start using it.
     Find the documentation at https://symfony.com/doc/current/forms.html

``App\Form\CommentFormType`` クラスは `App\Entity\Comment``  エンティティのフォームを定義します:

.. code-block:: php
    :caption: src/Form/CommentFormType.php
    :class: ignore

    namespace App\Form;

    use App\Entity\Comment;
    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\Component\OptionsResolver\OptionsResolver;

    class CommentFormType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $builder, array $options)
        {
            $builder
                ->add('author')
                ->add('text')
                ->add('email')
                ->add('createdAt')
                ->add('photoFilename')
                ->add('conference')
            ;
        }

        public function configureOptions(OptionsResolver $resolver)
        {
            $resolver->setDefaults([
                'data_class' => Comment::class,
            ]);
        }
    }

`フォームタイプ`_ は、モデルに紐付けられた *フォームフィールド* です。投稿されたデータとモデルクラスのプロパティの変換を行います。デフォルトでは、Symfonyは ``Comment`` エンティティのメタデータ（Doctrine のメタデータ）を使用して各フィールドの設定を推測します。例えば、 ``text`` フィールドはデータベースで大きなカラムとして定義されているので ``textarea`` が使われます。

フォームを表示する
---------------------------

ユーザーにフォームを表示するには、コントローラーでフォームを作成し、それをテンプレートに渡しましょう:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 19,29

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -2,7 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Comment;
     use App\Entity\Conference;
    +use App\Form\CommentFormType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    @@ -23,6 +25,9 @@ class ConferenceController extends AbstractController
         #[Route('/conference/{slug}', name: 'conference')]
         public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
         {
    +        $comment = new Comment();
    +        $form = $this->createForm(CommentFormType::class, $comment);
    +
             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    @@ -31,6 +36,7 @@ class ConferenceController extends AbstractController
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::PAGINATOR_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::PAGINATOR_PER_PAGE),
    +            'comment_form' => $form,
             ]);
         }
     }

フォームタイプを直接生成してはいけません。代わりに、``createForm()``  メソッドを使用してください。このメソッドは ``AbstractController`` で実装されており、フォーム作成を簡単にしています。

.. index::
    single: Twig;form

フォームをテンプレートに渡すには、``createView()``  メソッドを使い、テンプレートのためのフォーマットに変換します。

テンプレートにフォームを表示するには、Twig の関数の ``form`` を使います:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 10

    --- a/templates/conference/show.html.twig
    +++ b/templates/conference/show.html.twig
    @@ -30,4 +30,8 @@
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}
    +
    +    <h2>Add your own feedback</h2>
    +
    +    {{ form(comment_form) }}
     {% endblock %}

ブラウザのカンファレンスページを再読み込みすると、フォームの各フィールドが HTML に表示されているはずです（データタイプはモデルから派生しています）:

.. figure:: screenshots/form.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

``form()`` 関数はフォームタイプで定義された全ての情報を元に HTML フォームを生成します。ファイルアップロードのフィールドがある際には、``<form>`` タグに ``enctype=multipart/form-data`` も追加します。さらに、投稿時にエラーがあった際にはエラーメッセージを表示します。デフォルトのテンプレートを上書きすれば、カスタマイズも可能ですが、このプロジェクトではまずこのままでいきましょう。

フォームタイプをカスタマイズする
------------------------------------------------

フォームフィールドは、モデルから設定されていますが、フォームタイプを直接修正してデフォルトの設定をカスタマイズすることも可能です:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Form/CommentFormType.php
    +++ b/src/Form/CommentFormType.php
    @@ -4,20 +4,31 @@ namespace App\Form;

     use App\Entity\Comment;
     use Symfony\Component\Form\AbstractType;
    +use Symfony\Component\Form\Extension\Core\Type\EmailType;
    +use Symfony\Component\Form\Extension\Core\Type\FileType;
    +use Symfony\Component\Form\Extension\Core\Type\SubmitType;
     use Symfony\Component\Form\FormBuilderInterface;
     use Symfony\Component\OptionsResolver\OptionsResolver;
    +use Symfony\Component\Validator\Constraints\Image;

     class CommentFormType extends AbstractType
     {
         public function buildForm(FormBuilderInterface $builder, array $options): void
         {
             $builder
    -            ->add('author')
    +            ->add('author', null, [
    +                'label' => 'Your name',
    +            ])
                 ->add('text')
    -            ->add('email')
    -            ->add('createdAt')
    -            ->add('photoFilename')
    -            ->add('conference')
    +            ->add('email', EmailType::class)
    +            ->add('photo', FileType::class, [
    +                'required' => false,
    +                'mapped' => false,
    +                'constraints' => [
    +                    new Image(['maxSize' => '1024k'])
    +                ],
    +            ])
    +            ->add('submit', SubmitType::class)
             ;
         }

サブミットボタンを１つ追加しました。（テンプレートで ``{{ form(comment_form) }}`` 式を指定しただけのままで可能です）。

``photoFilename`` など、自動設定ができないフィールドもあります。 ``Comment`` エンティティは、写真のファイル名のみを保存する必要がありますが、フォームはファイルアップロードを処理する必要があります。このケースでは、``mapped`` フィールドを false として ``photo`` を追加します。このことで、 ``Comment`` のどのプロパティにもマップされないようになります。アップロードした写真をディスクに保存するといった処理は手動で実装をします。

カスタマイズの例として、フィールドのデフォルトのラベルも修正しました。

.. figure:: screenshots/form-customized.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

モデルをバリデートする
---------------------------------

フォームタイプは、フォームのWeb上で表示内容を設定します（HTML5バリデーションなど）。以下が生成された HTML フォームです:

.. code-block:: html
    :class: ignore

    <form name="comment_form" method="post" enctype="multipart/form-data">
        <div id="comment_form">
            <div >
                <label for="comment_form_author" class="required">Your name</label>
                <input type="text" id="comment_form_author" name="comment_form[author]" required="required" maxlength="255" />
            </div>
            <div >
                <label for="comment_form_text" class="required">Text</label>
                <textarea id="comment_form_text" name="comment_form[text]" required="required"></textarea>
            </div>
            <div >
                <label for="comment_form_email" class="required">Email</label>
                <input type="email" id="comment_form_email" name="comment_form[email]" required="required" />
            </div>
            <div >
                <label for="comment_form_photo">Photo</label>
                <input type="file" id="comment_form_photo" name="comment_form[photo]" />
            </div>
            <div >
                <button type="submit" id="comment_form_submit" name="comment_form[submit]">Submit</button>
            </div>
            <input type="hidden" id="comment_form__token" name="comment_form[_token]" value="DwqsEanxc48jofxsqbGBVLQBqlVJ_Tg4u9-BL1Hjgac" />
        </div>
    </form>

フォームは、コメントをした人のメールアドレスでは、 ``email`` 入力を使用し、ほとんどのフィールドを ``required`` とします。また、hidden フィールドの ``_token`` フィールドで `CSRF attacks`_ 対策をしています。

cURL などの HTTP クライアントを使用するなどして HTML バリデーション が効かないときは、サーバまで無効なデータが到達してしまいます。

``Comment`` データモデルにもバリデーション制約を追加する必要があります:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Entity/Comment.php
    +++ b/src/Entity/Comment.php
    @@ -5,6 +5,7 @@ namespace App\Entity;
     use App\Repository\CommentRepository;
     use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Validator\Constraints as Assert;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
     #[ORM\HasLifecycleCallbacks]
    @@ -16,12 +17,16 @@ class Comment
         private ?int $id = null;

         #[ORM\Column(length: 255)]
    +    #[Assert\NotBlank]
         private ?string $author = null;

         #[ORM\Column(type: Types::TEXT)]
    +    #[Assert\NotBlank]
         private ?string $text = null;

         #[ORM\Column(length: 255)]
    +    #[Assert\NotBlank]
    +    #[Assert\Email]
         private ?string $email = null;

         #[ORM\Column]

フォームを処理する
---------------------------

これでフォームを表示する準備ができました。

フォームを送信してコントローラーでデータベースに情報を永続化する処理をします:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentFormType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    @@ -14,6 +15,11 @@ use Symfony\Component\Routing\Annotation\Route;

     class ConferenceController extends AbstractController
     {
    +    public function __construct(
    +        private EntityManagerInterface $entityManager,
    +    ) {
    +    }
    +
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
    @@ -27,6 +33,15 @@ class ConferenceController extends AbstractController
         {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
    +        $form->handleRequest($request);
    +        if ($form->isSubmitted() && $form->isValid()) {
    +            $comment->setConference($conference);
    +
    +            $this->entityManager->persist($comment);
    +            $this->entityManager->flush();
    +
    +            return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
    +        }

             $offset = max(0, $request->query->getInt('offset', 0));
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

フォームを投稿すると、投稿された内容で ``Comment`` オブジェクトが更新されます。

フォームからは取り除きましたが、カンファレンスは、URL で既に決まっています。

フォームのバリデーションができなかった際は、ページを表示しますが、フォームには先程投稿した内容やエラーメッセージが入っているので、ユーザーに表示することができます。

フォームを試してみましょう。正しく動きデータベースに格納されるはずです（管理者バックエンドで確認してください）。しかし、まだ写真が正しく扱えていません。コントローラーで処理をしていないので正しく動きません。

ファイルをアップロードする
---------------------------------------

アップロードされた写真は、カンファレンスページで表示できるように Web からアクセスできるローカルのディスクに保存されるべきです。 ``public/uploads/photos`` ディレクトリ以下にしましょう。

.. index::
    single: Attribute;Autowire
    single: Autowire

コード中にディレクトリのパスをハードコードしたくないので、設定の中でグローバルな値として保存する方法が必要です。Symfony Containerは サービスに加え、 サービスの設定値として *paramters* も保存することができます:

.. code-block:: diff
    :caption: patch_file

    --- a/config/services.yaml
    +++ b/config/services.yaml
    @@ -4,6 +4,7 @@
     # Put parameters here that don't need to change on each machine where the app is deployed
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
    +    photo_dir: "%kernel.project_dir%/public/uploads/photos"

     services:
         # default configuration for services in *this* file

コンストラクタの引数にサービスを自動的に注入する方法についてはすでにみてきました。コンテナのパラメータについては、 ``Autowire`` アトリビュートを介して明示的に注入することができます。

これで、アップロードされたファイルをどこへどのように保存するか実装するための必要な情報は揃いました:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/ConferenceController.php
    +++ b/src/Controller/ConferenceController.php
    @@ -9,6 +9,8 @@ use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
    +use Symfony\Component\HttpFoundation\File\Exception\FileException;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Annotation\Route;
    @@ -29,13 +31,26 @@ class ConferenceController extends AbstractController
         }

         #[Route('/conference/{slug}', name: 'conference')]
    -    public function show(Request $request, Conference $conference, CommentRepository $commentRepository): Response
    -    {
    +    public function show(
    +        Request $request,
    +        Conference $conference,
    +        CommentRepository $commentRepository,
    +        #[Autowire('%photo_dir%')] string $photoDir,
    +    ): Response {
             $comment = new Comment();
             $form = $this->createForm(CommentFormType::class, $comment);
             $form->handleRequest($request);
             if ($form->isSubmitted() && $form->isValid()) {
                 $comment->setConference($conference);
    +            if ($photo = $form['photo']->getData()) {
    +                $filename = bin2hex(random_bytes(6)).'.'.$photo->guessExtension();
    +                try {
    +                    $photo->move($photoDir, $filename);
    +                } catch (FileException $e) {
    +                    // unable to upload the photo, give up
    +                }
    +                $comment->setPhotoFilename($filename);
    +            }

                 $this->entityManager->persist($comment);
                 $this->entityManager->flush();

写真のアップロードを行うのに、ファイルにランダムな名前を付けます。そして、アップロードされたファイルを写真格納ディレクトリの最終的な場所に移動します。そして、ファイル名はコメントオブジェクトに格納します。

写真の代わりに PDF ファイルをアップロードしてみてください。エラーメッセージが表示されるはずです。デザインは適用していないのでかっこよくはないですが、Webサイトのデザインをする際に綺麗にします。フォームの全ての要素のスタイルを1行の設定で変更するようにします。

フォームをデバッグする
---------------------------------

フォームが投稿された際に、何か問題があったときは、Symfony プロファイラーの "Form" パネルを使用してください。 "Form" パネルでは、オプションや投稿されたデータや内部的にどのように変換されたかなどフォームに関する情報を見ることができます。フォームにエラーがあれば、エラーを一覧として表示します。

一般的なフォームのワークフローは次のようになります:

* フォームがページに表示されます;

* ユーザーは POST リクエストでフォームを送信します;

* サーバーは、ユーザーを他のページ、もしくは同じページにリダイレクトします。

成功時のフォーム投稿のプロファイラーはどうやってアクセスしたら良いでしょうか？ページは、リダイレクトされるので、POST リクエストでデバッグツールバーを使うことができませんが、ノープロブレムです。リダイレクトされた後のページの緑の "200" 部分の左位置をマウスオーバーしてください。"302" リダイレクトとプロファイラーへのリンクが見えるはずです。

.. figure:: screenshots/form-wdt.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

そこをクリックして POST リクエストのプロファイルへアクセスし、 "Form" パネルを見てみましょう:

.. code-block:: terminal
    :class: hide

    $ rm -rf var/cache

.. figure:: screenshots/form-profiler.png
    :alt: /_profiler/450aa5
    :align: center
    :figclass: with-browser

管理者のバックエンドにアップロードされた写真を表示する
---------------------------------------------------------------------------------

まだ、管理者のバックエンドは、写真のファイル名が表示されていますので、実際の写真に変更しましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/src/Controller/Admin/CommentCrudController.php
    +++ b/src/Controller/Admin/CommentCrudController.php
    @@ -9,6 +9,7 @@ use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;
     use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\ImageField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
     use EasyCorp\Bundle\EasyAdminBundle\Filter\EntityFilter;
    @@ -45,7 +46,9 @@ class CommentCrudController extends AbstractCrudController
             yield TextareaField::new('text')
                 ->hideOnIndex()
             ;
    -        yield TextField::new('photoFilename')
    +        yield ImageField::new('photoFilename')
    +            ->setBasePath('/uploads/photos')
    +            ->setLabel('Photo')
                 ->onlyOnIndex()
             ;

アップロードされた写真を Git から除外する
-----------------------------------------------------------

Git リポジトリにアップロードされた画像を格納したくないので、まだコミットしないでください。 ``.gitignore`` ファイルに ``/public/uploads`` ディレクトリを追加してください:

.. code-block:: diff
    :caption: patch_file

    --- a/.gitignore
    +++ b/.gitignore
    @@ -1,3 +1,4 @@
    +/public/uploads

     ###> symfony/framework-bundle ###
     /.env.local

本番サーバーのアップロードされたファイルをソートする
------------------------------------------------------------------------------

最後のステップとして本番サーバーにアップロードされたファイルを格納するようにしましょう。何か特別なことをしないといけないのはなぜでしょうか。それは、ほとんどのクラウドプラットフォームは、読み込み権限のみのコンテナを使用しているからです。Platform.sh も例外ではありません。

Symfony プロジェクトにおいて全てが読み取り権限のみというわけではありません。コンテナを作成する際に（キャッシュウォームアップフェーズで）、可能な限り多くのキャッシュを生成するようにしますが、Symfony がユーザーキャッシュ、ログ、セッションなどをファイルシステムに保存している場合は書き込みできるようにする必要があります。

``.platform.app.yaml`` を見てください。既に ``var/`` ディレクトリは、書き込み可能で  *マウント* することができるようになっています。この ``var/`` ディレクトリが Symfony が書き込むことができる唯一のディレクトリです。ここにキャッシュやログが入ります。

アップロードされた写真のマウントを作成しましょう:

.. code-block:: diff
    :caption: patch_file

    --- a/.platform.app.yaml
    +++ b/.platform.app.yaml
    @@ -35,6 +35,7 @@ web:

     mounts:
         "/var": { source: local, source_path: var }
    +    "/public/uploads": { source: local, source_path: uploads }
         

     relationships:

これでコードをデプロイすれば、ローカルと同じように写真は ``public/uploads/`` ディレクトリに格納されます。

.. sidebar:: より深く学ぶために

    * `SymfonyCasts フォームチュートリアル`_;

    * `Symfony のHTMLフォームをカスタマイズする方法`_;

    * `Symfony のフォームのバリデーション`_;

    * `Symfony のフォームタイプのリファレンス`_;

    * `FlysystemBundle のドキュメント`_, このバンドルは、 AWS S3, Azure, Google Cloud Storage などのクラウドストレージの統合をサポートします;

    * `Symfony の設定パラメーター`_.

    * `Symfony のバリデーション制約`_;

    * `Symfony のフォームチートシート`_.

.. _`CSRF attacks`: https://owasp.org/www-community/attacks/csrf
.. _`フォームタイプ`: https://symfony.com/doc/current/forms.html#form-types
.. _`SymfonyCasts フォームチュートリアル`: https://symfonycasts.com/screencast/symfony-forms
.. _`Symfony のHTMLフォームをカスタマイズする方法`: https://symfony.com/doc/current/form/form_customization.html
.. _`Symfony のフォームのバリデーション`: https://symfony.com/doc/current/forms.html#validating-forms
.. _`Symfony のフォームタイプのリファレンス`: https://symfony.com/doc/current/reference/forms/types.html
.. _`FlysystemBundle のドキュメント`: https://github.com/thephpleague/flysystem-bundle/blob/master/docs/1-getting-started.md
.. _`Symfony の設定パラメーター`: https://symfony.com/doc/current/configuration.html#configuration-parameters
.. _`Symfony のバリデーション制約`: https://symfony.com/doc/current/validation.html#basic-constraints
.. _`Symfony のフォームチートシート`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony2/how_symfony2_forms_works_en.pdf
