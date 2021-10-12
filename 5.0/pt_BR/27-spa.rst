Construindo um SPA
==================

.. index::
    single: SPA
    single: Mobile

A maioria dos comentários será enviada ao longo da conferência, onde algumas pessoas não levam um laptop. Mas provavelmente têm um smartphone. Que tal criar um aplicativo móvel para verificar rapidamente os comentários da conferência?

Uma maneira de criar um aplicativo móvel como esse é construir um Single Page Application (SPA) em JavaScript. Um SPA é executado localmente, pode usar armazenamento local, pode chamar uma API HTTP remota e pode aproveitar os service workers para criar uma experiência praticamente nativa.

Criando o Aplicativo
--------------------

Para criar o aplicativo móvel, vamos usar o `Preact`_ e o **Encore do Symfony.** **Preact** é uma base pequena e eficiente, adequada para o SPA do Livro de Vistas.

Para manter o site e o SPA consistentes, vamos reutilizar as folhas de estilo Sass do site para o aplicativo móvel.

Crie o aplicativo SPA no diretório ``spa`` e copie as folhas de estilo do site:

.. code-block:: bash

    $ mkdir -p spa/src spa/public spa/assets/css
    $ cp assets/css/*.scss spa/assets/css/
    $ cd spa

.. note::

    Criamos um diretório ``public``, pois vamos interagir com o SPA principalmente através de um navegador. Poderíamos chamá-lo ``build`` caso quiséssemos apenas construir um aplicativo móvel.

Inicialize o arquivo ``package.json`` (o equivalente do ``composer.json`` para JavaScript):

.. code-block:: bash

    $ yarn init -y

Agora, adicione algumas dependências obrigatórias:

.. code-block:: bash

    $ yarn add @symfony/webpack-encore @babel/core @babel/preset-env babel-preset-preact preact html-webpack-plugin bootstrap

Por medida de segurança, adicione um arquivo ``.gitignore``:

.. code-block:: text
    :caption: .gitignore

    /node_modules
    /public
    /yarn-error.log
    # used later by Cordova
    /app

A última etapa de configuração é criar a configuração do Webpack Encore:

.. code-block:: javascript
    :caption: webpack.config.js
    :emphasize-lines: 8,11

    const Encore = require('@symfony/webpack-encore');
    const HtmlWebpackPlugin = require('html-webpack-plugin');

    Encore
        .setOutputPath('public/')
        .setPublicPath('/')
        .cleanupOutputBeforeBuild()
        .addEntry('app', './src/app.js')
        .enablePreactPreset()
        .enableSingleRuntimeChunk()
        .addPlugin(new HtmlWebpackPlugin({ template: 'src/index.ejs', alwaysWriteToDisk: true }))
    ;

    module.exports = Encore.getWebpackConfig();

Criando o Template Principal do SPA
-----------------------------------

Hora de criar o template inicial no qual o Preact irá renderizar o aplicativo:

.. code-block:: html
    :caption: src/index.ejs
    :emphasize-lines: 12

    <!DOCTYPE html>
    <html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />
        <meta name="msapplication-tap-highlight" content="no" />
        <meta name="viewport" content="user-scalable=no, initial-scale=1, maximum-scale=1, minimum-scale=1, width=device-width" />

        <title>Conference Guestbook application</title>
    </head>
    <body>
        <div id="app"></div>
    </body>
    </html>

A tag ``<div>`` é onde o aplicativo será renderizado pelo JavaScript. Aqui está a primeira versão do código que renderiza a view "Hello World":

.. code-block:: text
    :caption: src/app.js
    :emphasize-lines: 3,11

    import {h, render} from 'preact';

    function App() {
        return (
            <div>
                Hello world!
            </div>
        )
    }

    render(<App />, document.getElementById('app'));

A última linha registra a função ``App()`` no elemento ``#app`` da página HTML.

Agora está tudo pronto!

Executando um SPA no Navegador
------------------------------

.. index::
    single: Symfony CLI;server:start
    single: Symfony CLI;server:stop

Como esse aplicativo é independente do site principal, precisamos executar outro servidor web:

.. code-block:: bash
    :class: hide

    $ symfony server:stop

.. code-block:: bash

    $ symfony server:start -d --passthru=index.html

A flag ``--passthru`` informa ao servidor web para passar todas as requisições HTTP para o arquivo ``public/index.html`` (``public/`` é o diretório raiz padrão do servidor web). Esta página é gerenciada pelo aplicativo Preact e obtém a página que vai renderizar pelo histórico do "navegador".

Para compilar os arquivos CSS **e JavaScript**, execute ``yarn``:

.. code-block:: bash

    $ yarn encore dev

Abra o SPA em um navegador:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

E confira o nosso SPA olá mundo:

.. figure:: screenshots/spa.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Adicionando um Roteador para Lidar com Estados
----------------------------------------------

No momento, o SPA não é capaz de manipular páginas diferentes. Para implementar várias páginas, precisamos de um roteador, assim como o Symfony. Vamos usar o **preact-router**. Ele recebe uma URL como entrada e encontra um componente Preact para exibir.

Instale o preact-router:

.. code-block:: bash

    $ yarn add preact-router

Crie uma página para ser a página inicial (um *componente Preact*):

.. code-block:: text
    :caption: src/pages/home.js

    import {h} from 'preact';

    export default function Home() {
        return (
            <div>Home</div>
        );
    };

E outra para ser a página da conferência:

.. code-block:: text
    :caption: src/pages/conference.js

    import {h} from 'preact';

    export default function Conference() {
        return (
            <div>Conference</div>
        );
    };

Substitua a ``div`` "Hello World" pelo componente ``Router``:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 15,17,20-23

    --- a/src/app.js
    +++ b/src/app.js
    @@ -1,9 +1,22 @@
     import {h, render} from 'preact';
    +import {Router, Link} from 'preact-router';
    +
    +import Home from './pages/home';
    +import Conference from './pages/conference';

     function App() {
         return (
             <div>
    -            Hello world!
    +            <header>
    +                <Link href="/">Home</Link>
    +                <br />
    +                <Link href="/conference/amsterdam2019">Amsterdam 2019</Link>
    +            </header>
    +
    +            <Router>
    +                <Home path="/" />
    +                <Conference path="/conference/:slug" />
    +            </Router>
             </div>
         )
     }

Construa o aplicativo novamente:

.. code-block:: bash

    $ yarn encore dev

Se você atualizar o aplicativo no navegador, agora você pode clicar nos links "Home" e no da conferência. Note que a URL e os botões voltar/avançar do seu navegador funcionam conforme esperado.

Estilizando o SPA
-----------------

Voltando ao site, vamos adicionar o loader do Sass:

.. code-block:: bash

    $ yarn add node-sass sass-loader

Ative o loader do Sass no Webpack e adicione uma referência à folha de estilo:

.. code-block:: diff
    :caption: patch_file

    --- a/src/app.js
    +++ b/src/app.js
    @@ -1,3 +1,5 @@
    +import '../assets/css/app.scss';
    +
     import {h, render} from 'preact';
     import {Router, Link} from 'preact-router';

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -7,6 +7,7 @@ Encore
         .cleanupOutputBeforeBuild()
         .addEntry('app', './src/app.js')
         .enablePreactPreset()
    +    .enableSassLoader()
         .enableSingleRuntimeChunk()
         .addPlugin(new HtmlWebpackPlugin({ template: 'src/index.ejs', alwaysWriteToDisk: true }))
     ;

Agora podemos atualizar o aplicativo para usar as folhas de estilo:

.. code-block:: diff
    :caption: patch_file

    --- a/src/app.js
    +++ b/src/app.js
    @@ -9,10 +9,20 @@ import Conference from './pages/conference';
     function App() {
         return (
             <div>
    -            <header>
    -                <Link href="/">Home</Link>
    -                <br />
    -                <Link href="/conference/amsterdam2019">Amsterdam 2019</Link>
    +            <header className="header">
    +                <nav className="navbar navbar-light bg-light">
    +                    <div className="container">
    +                        <Link className="navbar-brand mr-4 pr-2" href="/">
    +                            &#128217; Guestbook
    +                        </Link>
    +                    </div>
    +                </nav>
    +
    +                <nav className="bg-light border-bottom text-center">
    +                    <Link className="nav-conference" href="/conference/amsterdam2019">
    +                        Amsterdam 2019
    +                    </Link>
    +                </nav>
                 </header>

                 <Router>

Construa o aplicativo mais uma vez:

.. code-block:: bash

    $ yarn encore dev

Agora você pode desfrutar de um SPA totalmente estilizado:

.. figure:: screenshots/spa-home.png
    :alt: /
    :align: center
    :figclass: with-browser spa

Obtendo Dados da API
--------------------

A estrutura do aplicativo Preact está concluída: O Preact Router manipula os estados da página - incluindo o placeholder para o slug da conferência - e a folha de estilo principal do aplicativo é usada para estilizar o SPA.

Para tornar o SPA dinâmico, precisamos buscar os dados da API através de chamadas HTTP.

Configure o Webpack para expor a variável de ambiente do endpoint da API:

.. code-block:: diff
    :caption: patch_file

    --- a/webpack.config.js
    +++ b/webpack.config.js
    @@ -1,3 +1,4 @@
    +const webpack = require('webpack');
     const Encore = require('@symfony/webpack-encore');
     const HtmlWebpackPlugin = require('html-webpack-plugin');

    @@ -10,6 +11,9 @@ Encore
         .enableSassLoader()
         .enableSingleRuntimeChunk()
         .addPlugin(new HtmlWebpackPlugin({ template: 'src/index.ejs', alwaysWriteToDisk: true }))
    +    .addPlugin(new webpack.DefinePlugin({
    +        'ENV_API_ENDPOINT': JSON.stringify(process.env.API_ENDPOINT),
    +    }))
     ;

     module.exports = Encore.getWebpackConfig();

A variável de ambiente ``API_ENDPOINT`` deve apontar para o servidor web do site onde temos o endpoint da API em ``/api``. Vamos configurá-la corretamente quando executarmos o ``yarn encore`` em breve.

Crie um arquivo ``api.js`` para abstrair a recuperação dos dados da API:

.. code-block:: text
    :caption: src/api/api.js

    function fetchCollection(path) {
        return fetch(ENV_API_ENDPOINT + path).then(resp => resp.json()).then(json => json['hydra:member']);
    }

    export function findConferences() {
        return fetchCollection('api/conferences');
    }

    export function findComments(conference) {
        return fetchCollection('api/comments?conference='+conference.id);
    }

Agora você pode ajustar os componentes header e home:

.. code-block:: diff
    :caption: patch_file

    --- a/src/app.js
    +++ b/src/app.js
    @@ -2,11 +2,23 @@ import '../assets/css/app.scss';

     import {h, render} from 'preact';
     import {Router, Link} from 'preact-router';
    +import {useState, useEffect} from 'preact/hooks';

    +import {findConferences} from './api/api';
     import Home from './pages/home';
     import Conference from './pages/conference';

     function App() {
    +    const [conferences, setConferences] = useState(null);
    +
    +    useEffect(() => {
    +        findConferences().then((conferences) => setConferences(conferences));
    +    }, []);
    +
    +    if (conferences === null) {
    +        return <div className="text-center pt-5">Loading...</div>;
    +    }
    +
         return (
             <div>
                 <header className="header">
    @@ -19,15 +31,17 @@ function App() {
                     </nav>

                     <nav className="bg-light border-bottom text-center">
    -                    <Link className="nav-conference" href="/conference/amsterdam2019">
    -                        Amsterdam 2019
    -                    </Link>
    +                    {conferences.map((conference) => (
    +                        <Link className="nav-conference" href={'/conference/'+conference.slug}>
    +                            {conference.city} {conference.year}
    +                        </Link>
    +                    ))}
                     </nav>
                 </header>

                 <Router>
    -                <Home path="/" />
    -                <Conference path="/conference/:slug" />
    +                <Home path="/" conferences={conferences} />
    +                <Conference path="/conference/:slug" conferences={conferences} />
                 </Router>
             </div>
         )
    --- a/src/pages/home.js
    +++ b/src/pages/home.js
    @@ -1,7 +1,28 @@
     import {h} from 'preact';
    +import {Link} from 'preact-router';
    +
    +export default function Home({conferences}) {
    +    if (!conferences) {
    +        return <div className="p-3 text-center">No conferences yet</div>;
    +    }

    -export default function Home() {
         return (
    -        <div>Home</div>
    +        <div className="p-3">
    +            {conferences.map((conference)=> (
    +                <div className="card border shadow-sm lift mb-3">
    +                    <div className="card-body">
    +                        <div className="card-title">
    +                            <h4 className="font-weight-light">
    +                                {conference.city} {conference.year}
    +                            </h4>
    +                        </div>
    +
    +                        <Link className="btn btn-sm btn-blue stretched-link" href={'/conference/'+conference.slug}>
    +                            View
    +                        </Link>
    +                    </div>
    +                </div>
    +            ))}
    +        </div>
         );
    -};
    +}

Finalmente, o Preact Router está passando o placeholder "slug" para o componente Conference como uma propriedade. Use-o para exibir a conferência apropriada e seus comentários, novamente usando a API; e adapte a renderização para usar os dados da API:

.. code-block:: diff
    :caption: patch_file

    --- a/src/pages/conference.js
    +++ b/src/pages/conference.js
    @@ -1,7 +1,48 @@
     import {h} from 'preact';
    +import {findComments} from '../api/api';
    +import {useState, useEffect} from 'preact/hooks';
    +
    +function Comment({comments}) {
    +    if (comments !== null && comments.length === 0) {
    +        return <div className="text-center pt-4">No comments yet</div>;
    +    }
    +
    +    if (!comments) {
    +        return <div className="text-center pt-4">Loading...</div>;
    +    }
    +
    +    return (
    +        <div className="pt-4">
    +            {comments.map(comment => (
    +                <div className="shadow border rounded-lg p-3 mb-4">
    +                    <div className="comment-img mr-3">
    +                        {!comment.photoFilename ? '' : (
    +                            <a href={ENV_API_ENDPOINT+'uploads/photos/'+comment.photoFilename} target="_blank">
    +                                <img src={ENV_API_ENDPOINT+'uploads/photos/'+comment.photoFilename} />
    +                            </a>
    +                        )}
    +                    </div>
    +
    +                    <h5 className="font-weight-light mt-3 mb-0">{comment.author}</h5>
    +                    <div className="comment-text">{comment.text}</div>
    +                </div>
    +            ))}
    +        </div>
    +    );
    +}
    +
    +export default function Conference({conferences, slug}) {
    +    const conference = conferences.find(conference => conference.slug === slug);
    +    const [comments, setComments] = useState(null);
    +
    +    useEffect(() => {
    +        findComments(conference).then(comments => setComments(comments));
    +    }, [slug]);

    -export default function Conference() {
         return (
    -        <div>Conference</div>
    +        <div className="p-3">
    +            <h4>{conference.city} {conference.year}</h4>
    +            <Comment comments={comments} />
    +        </div>
         );
    -};
    +}

Agora o SPA precisa conhecer a URL da nossa API, através da variável de ambiente ``API_ENDPOINT``. Configure-a como a URL do servidor web da API (executando no diretório ``..``):

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore dev

Agora você pode executar em segundo plano também:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` symfony run -d --watch=webpack.config.js yarn encore dev --watch

E o aplicativo no navegador agora deve funcionar corretamente:

.. figure:: screenshots/spa-home-final.png
    :alt: /
    :align: center
    :figclass: with-browser spa

.. figure:: screenshots/spa-conference-final.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser spa

Uau! Agora temos um SPA totalmente funcional, com roteador e dados reais. Podemos organizar ainda mais o aplicativo Preact se quisermos, mas ele já está funcionando muito bem.

Implantando o SPA em Produção
-------------------------------

.. index::
    single: SymfonyCloud;Multi-Applications

A SymfonyCloud permite implantar várias aplicações por projeto. A adição de outra aplicação pode ser feita criando um arquivo ``.symfony.cloud.yaml`` em qualquer subdiretório. Crie um arquivo desse no diretório ``spa/`` com o nome ``spa``:

.. code-block:: yaml
    :caption: .symfony.cloud.yaml
    :emphasize-lines: 1

    name: spa

    type: php:7.3
    size: S

    build:
        flavor: none

    web:
        commands:
            start: sleep
        locations:
            "/":
                root: "public"
                index:
                    - "index.html"
                scripts: false
                expires: 10m

    hooks:
        build: |
            set -x -e

            curl -s https://get.symfony.com/cloud/configurator | (>&2 bash)
            (>&2
                unset NPM_CONFIG_PREFIX
                export NVM_DIR=${SYMFONY_APP_DIR}/.nvm

                yarn-install

                set +x && . "${SYMFONY_APP_DIR}/.nvm/nvm.sh" && set -x

                yarn encore prod
            )

.. index::
    single: SymfonyCloud;Routes

Edite o arquivo ``.symfony/routes.yaml``, roteando o subdomínio ``spa.`` para a aplicação ``spa``, armazenada no diretório raiz do projeto:

.. code-block:: bash

    $ cd ../

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 4,5

    --- a/.symfony/routes.yaml
    +++ b/.symfony/routes.yaml
    @@ -1,2 +1,5 @@
    +"https://spa.{all}/": { type: upstream, upstream: "spa:http" }
    +"http://spa.{all}/": { type: redirect, to: "https://spa.{all}/" }
    +
     "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
     "http://{all}/": { type: redirect, to: "https://{all}/" }

Configurando CORS para o SPA
----------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

Se você implantar o código agora, ele não funcionará, pois o navegador bloqueará a requisição para a API. Precisamos permitir explicitamente que o SPA acesse a API. Obtenha o nome de domínio atual atribuído à sua aplicação:

.. code-block:: bash

    $ symfony env:urls --first

Defina a variável de ambiente ``CORS_ALLOW_ORIGIN`` adequadamente:

.. code-block:: bash

    $ symfony var:set "CORS_ALLOW_ORIGIN=^`symfony env:urls --first | sed 's#/$##' | sed 's#https://#https://spa.#'`$"

Supondo que seu domínio seja ``https://master-5szvwec-hzhac461b3a6o.eu.s5y.io/``, então as chamadas a ``sed`` o converterão para ``https://spa.master-5szvwec-hzhac461b3a6o.eu.s5y.io``.

Também precisamos definir a variável de ambiente ``API_ENDPOINT``:

.. code-block:: bash

    $ symfony var:set API_ENDPOINT=`symfony env:urls --first`

Faça o commit e implante:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -a -m'Add the SPA application'
    $ symfony deploy

Acesse o SPA em um navegador especificando a aplicação como uma flag:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote --app=spa

Usando Cordova para Construir um Aplicativo para Smartphone
-----------------------------------------------------------

.. index::
    single: SPA;Cordova
    single: Apache Cordova
    single: Cordova

O **Apache Cordova** é uma ferramenta que cria aplicativos multiplataforma para smartphones. Boa notícia: ele pode usar o SPA que acabamos de criar.

Vamos instalá-lo:

.. code-block:: bash

    $ cd spa
    $ yarn global add cordova

.. note::

    Você também precisa instalar o Android SDK. Esta seção menciona apenas Android, mas o Cordova funciona com todas as plataformas móveis, incluindo iOS.

Crie a estrutura de diretórios do aplicativo:

.. code-block:: bash
    :class: answers(n)

    $ cordova create app

E execute a geração do aplicativo Android:

.. code-block:: bash
    :class: ignore

    $ cd app
    $ cordova platform add android
    $ cd ..

Isso é tudo o que você precisa. Agora você pode construir os arquivos de produção e movê-los para o Cordova:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore production
    $ rm -rf app/www
    $ mkdir -p app/www
    $ cp -R public/ app/www

Execute o aplicativo em um smartphone ou emulador:

.. code-block:: bash
    :class: ignore

    $ cordova run android

.. sidebar:: Indo Além

    * `O site oficial do Preact <https://preactjs.com/>`_;

    * `O site oficial do Cordova <https://cordova.apache.org/>`_.

.. _`preact`: https://preactjs.com/
