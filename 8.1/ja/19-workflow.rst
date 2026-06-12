ワークフローを使って判定する
==========================================

.. index::
    single: Components;Workflow
    single: Workflow

モデルが状態を持つことはよくあります。今はコメントの状態はスパムチェッカーからしか変わりませんが、他の状態の追加を検討してみましょう。

スパムチェックの後に、Webサイトの管理者が全てのコメントをモデレートしたいとしましょう。このプロセスは次の行のようなものです:

* ユーザーがコメントを追加した際の ``submitted`` 状態から始めましょう;

* スパムチェッカーにコメントを分析させ、 ``potential_spam``, ``ham`` (スパムでないメール), ``rejected`` のいずれかの状態にスイッチさせるようにしましょう;

* リジェクトされなければ、Webサイトの管理者がコメントを ``published`` もしくは ``rejected`` の状態に変更するのを待ちましょう。

ロジックを実装するのはそれほど複雑ではありませんが、さらにルールを追加することで複雑になることもあります。ロジックを自分でコーディングするのではなく、Symfony ワークフローコンポーネントを使用してみます:

.. code-block:: terminal

    $ symfony composer req workflow

ワークフローを記述する
---------------------------------

コメントワークフローは、``config/packages/workflow.yaml``  ファイルに記述されます:

.. code-block:: yaml
    :caption: config/packages/workflow.yaml
    :emphasize-lines: 3,4,9,11

    framework:
        workflows:
            comment:
                type: state_machine
                audit_trail:
                    enabled: "%kernel.debug%"
                marking_store:
                    type: 'method'
                    property: 'state'
                supports:
                    - App\Entity\Comment
                initial_marking: submitted
                places:
                    - submitted
                    - ham
                    - potential_spam
                    - spam
                    - rejected
                    - published
                transitions:
                    accept:
                        from: submitted
                        to:   ham
                    might_be_spam:
                        from: submitted
                        to:   potential_spam
                    reject_spam:
                        from: submitted
                        to:   spam
                    publish:
                        from: potential_spam
                        to:   published
                    reject:
                        from: potential_spam
                        to:   rejected
                    publish_ham:
                        from: ham
                        to:   published
                    reject_ham:
                        from: ham
                        to:   rejected

.. index::
    single: Command;workflow:dump

ワークフローをバリデートするために、Mermaid 形式で視覚的な表現を生成します:

.. code-block:: terminal

    $ symfony console workflow:dump comment --dump-format=mermaid

出力を `Mermaid Live Editor`_ に貼り付けるとレンダリングできます。GitHub や GitLab も、Markdown ファイル内の Mermaid ダイアグラムをネイティブにレンダリングします:

.. image:: images/workflow.png
    :align: center

ワークフローを使用する
---------------------------------

現在のメッセージハンドラーのロジックをワークフローに置き換えます:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -6,7 +6,10 @@ use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
    +use Psr\Log\LoggerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
    +use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Workflow\WorkflowInterface;

     #[AsMessageHandler]
     class CommentMessageHandler
    @@ -15,6 +18,9 @@ class CommentMessageHandler
             private EntityManagerInterface $entityManager,
             private SpamChecker $spamChecker,
             private CommentRepository $commentRepository,
    +        private MessageBusInterface $bus,
    +        private WorkflowInterface $commentStateMachine,
    +        private ?LoggerInterface $logger = null,
         ) {
         }

    @@ -25,12 +31,18 @@ class CommentMessageHandler
                 return;
             }

    -        if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
    -            $comment->setState('spam');
    -        } else {
    -            $comment->setState('published');
    +        if ($this->commentStateMachine->can($comment, 'accept')) {
    +            $score = $this->spamChecker->getSpamScore($comment, $message->getContext());
    +            $transition = match ($score) {
    +                2 => 'reject_spam',
    +                1 => 'might_be_spam',
    +                default => 'accept',
    +            };
    +            $this->commentStateMachine->apply($comment, $transition);
    +            $this->entityManager->flush();
    +            $this->bus->dispatch($message);
    +        } elseif ($this->logger) {
    +            $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }
    -
    -        $this->entityManager->flush();
         }
     }

新しいロジックは以下のようになります:

* メッセージ内のコメントにおいて、``accept`` 遷移が可能であれば、スパムチェックを行います;

* 結果に応じた正しい状態遷移を選んでください;

* ``setState()`` メソッドを介して、コメントを更新するための ``apply()`` を呼んでください;

* ``flush()`` を呼び、データベースに変更をコミットしてください;

* ワークフローの再遷移を許容させるため、メッセージを再ディスパッチしてください。

管理者のバリデーションはまだ実装していないので、メッセージの取得実行をすると "コメントメッセージを削除します" とログが吐かれます。

次の章までに自動バリデーションを実装しましょう:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -41,6 +41,9 @@ class CommentMessageHandler
                 $this->commentStateMachine->apply($comment, $transition);
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
    +        } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    +            $this->commentStateMachine->apply($comment, $this->commentStateMachine->can($comment, 'publish') ? 'publish' : 'publish_ham');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

``symfony server:log`` を実行し、フロントエンドでコメントが追加し、順々に状態が遷移することを確認してください。

DIコンテナからサービスを検索する
-----------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

DIを使う際、インターフェースや場合によっては具体的な実装クラスの名前をタイプヒンティングとしてDIコンテナからサービスを取得しますが、インターフェースが複数の実装クラスを持っている場合、Symfonyはどのクラスが必要か推測することができません。その場合は明示する方法が必要です。

このような例は、前章で ``WorkflowInterface`` を注入した時に出くわしたばかりです。

一般的な ``WorkflowInterface`` インターフェースをコンストラクタで注入すると、Symfonyはどのようにしてどちらのワークフローの実装を使うかを推測するでしょうか? Symfonyは引数名を基にした規約を利用します: ``$commentStateMachine`` は ``comment`` ワークフロー ( ``state_machine`` 型) の設定を参照します。

もし規約が思い出せない場合は、 ``debug:container`` コマンドを使いましょう。 ``workflow`` を含む全てのサービスを検索します:

.. code-block:: terminal
    :emphasize-lines: 12
    :class: ignore

    $ symfony console debug:container workflow

     Select one of the following services to display its information:
      [0] console.command.workflow_dump
      [1] workflow.abstract
      [2] workflow.marking_store.method
      [3] workflow.registry
      [4] workflow.security.expression_language
      [5] workflow.twig_extension
      [6] monolog.logger.workflow
      [7] Symfony\Component\Workflow\Registry
      [8] Symfony\Component\Workflow\WorkflowInterface $commentStateMachine
      [9] Psr\Log\LoggerInterface $workflowLogger
     >

``8`` , ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` は、 ``$commentStateMachine`` を引数名として使うことに特別な意味があることを伝えています。

.. note::

    前章で見たように ``debug:autowiring`` を使うことができます:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: より深く学ぶために

    * `ワークフローとステートマシーン`_ に関しての説明と、どちらを使うかについて;

    * `Symfony ワークフローのドキュメント`_.

.. _`Mermaid Live Editor`: https://mermaid.live/
.. _`ワークフローとステートマシーン`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`Symfony ワークフローのドキュメント`: https://symfony.com/doc/current/workflow.html
