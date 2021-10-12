ساخت یک SPA
=================

.. index::
    single: SPA
    single: Mobile

اکثر کامنت‌ها در طول کنفرانس ارسال می‌شوند که کاربران با خود لپتاپ به همراه ندارند. اما آن‌ها احتمالاً گوشی هوشمند دارند. چطور است که یک اپلیکیشن موبایل برای بررسی سریع کامنت‌های کنفرانس، ایجاد کنیم؟

یک راه برای ایجاد چنین اپلیکیشن موبایلی‌ای، ساخت یک اپلیکیشن تک‌صفحه‌ای (SPA) به کمک Javascript است. یک SPA به صورت محلی اجرا می‌شود، می‌تواند از انبار محلی (local storage) استفاده کند، می‌تواند یک HTTP API ریموت را فراخوانی کند و از سرویس‌های کارگر (service workers) بهره بگیرد تا یک تجربه‌ی تقریباً بومی (native) را ایجاد کند.

ایجاد اپلیکیشن
---------------------------

ما می‌خواهیم برای ایجاد اپلیکیشن موبایلی، از `Preact`_ و **Symfony Encore** استفاده کنیم. **Preact** یک پایه‌ی کوچک، کارا و درخور برای SPA مربوط به Guestbook است.

برای اینکه هم وب‌سایت و هم SPA را سازگار نماییم، می‌خواهیم stylesheet‌های Sass مربوط به وب‌سایت را در اپلیکیشن موبایل بازاستفاده کنیم.

اپلیکیشن SPA را در پوشه‌ی ``spa`` ایجاد کنید و stylesheet‌های وب‌سایت را کپی کنید:

.. code-block:: bash

    $ mkdir -p spa/src spa/public spa/assets/css
    $ cp assets/css/*.scss spa/assets/css/
    $ cd spa

.. note::

    از آنجایی که عمدتاً از طریق مرورگر با SPA در تعامل خواهیم بود، یک پوشه‌ی ``public`` نیز ایجاد کرده‌ایم. اگر تنها می‌خواستیم که یک اپلیکیشن موبایلی ایجاد کنیم، می‌توانستیم نام آن را ``build`` بگذاریم.

فایل ``package.json`` (هم‌ارز فایل ``composer.json`` در JavaScript) را ایجاد کنید:

.. code-block:: bash

    $ yarn init -y

حالا تعدادی وابستگی الزامی را اضافه کنید:

.. code-block:: bash

    $ yarn add @symfony/webpack-encore @babel/core @babel/preset-env babel-preset-preact preact html-webpack-plugin bootstrap

علاوه بر این، فایل ``.gitignore`` را هم اضافه کنید:

.. code-block:: text
    :caption: .gitignore

    /node_modules
    /public
    /yarn-error.log
    # used later by Cordova
    /app

آخرین گام پیکربندی، ایجاد پیکربندی Webpack Encore است:

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

ایجاد قالب اصلی SPA
--------------------------------

زمان آن است که قالب اولیه را که در آن Preact اپلیکیشن را render خواهد کرد،  ایجاد کنیم:

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

تگ ``<div>`` جایی است که اپلیکیشن توسط JavaScript در آن render خواهد شد. این اولین نسخه از کد است که نمایشی از «Hello World» را render می‌کند:

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

آخرین خط، تابع ``App()`` را بر روی المان ``#app`` از صفحه‌ی HTML، ثبت می‌کند.

حالا همه چیز آماده است!

اجرای یک SPA در مرورگر
-------------------------------------

.. index::
    single: Symfony CLI;server:start
    single: Symfony CLI;server:stop

ار آنجایی که وب‌سایت اصلی از اپلیکیشن مستقل است، ما نیاز داریم که یک وب سرور دیگر اجرا کنیم:

.. code-block:: bash
    :class: hide

    $ symfony server:stop

.. code-block:: bash

    $ symfony server:start -d --passthru=index.html

پرچم ``--passthru`` به وب سرور می‌گوید که تمام درخواست‌های HTML را به فایل ``public/index.html``بدهد (پوشه‌ی ریشه‌ی پیشفرض در وب سرور، ``public/`` است). این صفحه توسط اپلیکیشن Preact مدیریت شده و صفحه‌ی مربوطه را می‌گیرد تا از طریق تاریخچه‌ی «مرورگر» آن را render کند.

برای کامپایل‌کردن فایل‌های CSS و **JavaScript**، فرمان ``yarn`` را اجرا کنید:

.. code-block:: bash

    $ yarn encore dev

SPA را در مرورگر باز کنید:

.. code-block:: bash
    :class: ignore

    $ symfony open:local

و به SPA‌ی ساده‌ی ما نگاه کنید:

.. figure:: screenshots/spa.png
    :alt: /
    :align: center
    :figclass: with-browser spa

افزودن یک راه‌یاب (Router) برای رسیدگی به وضعیت‌ها
---------------------------------------------------------------------------------------

SPA در حال حاضر قادر به رسیدگی به صفحه‌های مختلف نیست. برای پیاده‌سازی صفحه‌های متعدد، مشابه سیمفونی نیاز به یک راه‌یاب داریم. ما قصد داریم از **preact-router** استفاده کنیم. این راه‌یاب یک URL می‌گیرد و یک کامپوننت Preact منطبق با آن را نمایش می‌دهد.

preact-router را نصب کنید:

.. code-block:: bash

    $ yarn add preact-router

یک صفحه برای صفحه‌ی اصلی ایجاد کنید (یک *کامپوننتِ Preact*):

.. code-block:: text
    :caption: src/pages/home.js

    import {h} from 'preact';

    export default function Home() {
        return (
            <div>Home</div>
        );
    };

و یک صفحه‌ی دیگر برای صفحه‌ی کنفرانس ایحاد کنید:

.. code-block:: text
    :caption: src/pages/conference.js

    import {h} from 'preact';

    export default function Conference() {
        return (
            <div>Conference</div>
        );
    };

``div`` مربوط به «Hello World» را با کامپوننت ``Router`` جایگزین کنید:

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

اپلیکیشن را مجدداً بسازید:

.. code-block:: bash

    $ yarn encore dev

حالا اگر اپلیکیشن را در مرورگر تازه‌سازی کنید، می‌توانید بر روی پیوندهای «Home» و کنفرانس کلیک کنید. توجه کنید که URL مرورگر و کلیدهای قبلی/بعدی مرورگر به همان نحوی که انتظار دارید کار می‌کنند.

زیباسازیِ SPA
----------------------

مشابه کاری که برای وب‌سایت انجام دادیم، بیایید بارگیرنده‌ی Sass را اضافه کنیم:

.. code-block:: bash

    $ yarn add node-sass sass-loader

بارگیرنده‌ی Sass را در Webpack فعال کنید و یک ارجاع به stylesheet بیافزایید:

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

حالا می‌توانیم اپلیکیشن را به‌روزرسانی کنیم تا از stylesheetها استفاده کند:

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

یک بار دیگر اپلیکیشن را بسازید:

.. code-block:: bash

    $ yarn encore dev

حالا می‌توانید از SPA کاملاً زیبا‌سازی‌شده، لذت ببرید:

.. figure:: screenshots/spa-home.png
    :alt: /
    :align: center
    :figclass: with-browser spa

واکشی داده از API
----------------------------

حالا ساختار اپلیکیشن Preact تکمیل شده است: راه‌یاب Preact به وضعیت صفحات (شامل محل قرارگیری slug مربوط به کنفرانس) رسیدگی می‌کند و stylesheet اپلیکیشن اصلی برای زیباسازی SPA استفاده می‌گردد.

ما برای پویاکردن SPA، نیاز داریم که از طریق فراخوانی‌های HTTP، داده‌ها را از API واکشی کنیم.

Webpack را پیکربندی کنید تا متغیر محیط مربوط به پایانه‌ی API را ارائه کند:

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

متغیر محیط ``API_ENDPOINT`` باید در وب سرور وب‌سایت به جایی که پایانه‌ی API قرار دارد، یعنی ``/api``، اشاره کند. ما آن را به زودی و هنگامی که ``yarn encore`` را اجرا کنیم، پیکربندی خواهیم کرد.

فایل ``api.js`` که دریافت داده از API را انتزاعی می‌کند، ایجاد کنید:

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

حالا شما می‌توانید کامپوننت‌های سربرگ و خانه (home) را وفق بدهید:

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

در نهایت، راه‌یاب Preact محل قرارگیری «slug» را به عنوان یک ویژگی به کامپوننت Conference می‌دهد. از آن استفاده کنید تا کنفرانس صحیح و کامنت‌هایش را مجدداً با بهره‌گیری از API، نمایش دهید و renderشدن را وفق دهید تا از داده‌های API استفاده کند:

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

حالا SPA نیاز دارد تا URL مربوط به API ما را از طریق متغیر محیط ``API_ENDPOINT`` بداند. آن را بر روی URL وب سرورِ API تنظیم کنید (در حال اجرا در پوشه‌ی ``..``):

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore dev

همچنین شما می‌توانید آن را در پس‌زمینه اجرا کنید:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` symfony run -d --watch=webpack.config.js yarn encore dev --watch

و حالا اپلیکیشن باید در مرورگر به‌درستی کار کند:

.. figure:: screenshots/spa-home-final.png
    :alt: /
    :align: center
    :figclass: with-browser spa

.. figure:: screenshots/spa-conference-final.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser spa

واو! حالا ما یک SPA کاملاً کارا همراه با راه‌یاب و داده‌های واقعی داریم. اگر بخواهیم، می‌توانیم اپلیکیشن Preact را بیش از این سازماندهی کنیم، اما در حال حاضر عالی کار می‌کند.

استقرار SPA در محیط عمل‌آوری
--------------------------------------------------

.. index::
    single: SymfonyCloud;Multi-Applications

SymfonyCloud اجازه می‌دهد تا برای هر پروژه، چندین اپلیکیشن مستقر کنید. افزودن یک اپلیکیشن دیگر، می‌تواند از طریق ایجاد یک فایل ``.symfony.cloud.yaml`` در هر زیرپوشه‌ای انجام شود. یکی از این فایل‌ها را در پوشه‌ی ``spa/`` و با نام ``spa`` ایجاد کنید:

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

فایل ``.symfony/routes.yaml`` را ویرایش نمایید تا زیردامنه‌ی ``spa.`` را به اپلیکیشن ``spa`` که در پوشه‌ی ریشه‌ی پروژه ذخیره شده است، هدایت کند:

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

پیکربندی CORS برای SPA
----------------------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

اگر حالا کد را مستقر کنید، کار نخواهد کرد، چون مرورگر درخواست‌های API را مسدود می‌کند. ما نیاز داریم تا به صورت صریح اجازه دهیم که SPA به API دسترسی پیدا کند. نام دامنه‌ی فعلی که به اپلیکیشن ضمیمه شده است را بگیرید:

.. code-block:: bash

    $ symfony env:urls --first

متغیر محیط ``CORS_ALLOW_ORIGIN`` را بر اساس آن تعریف کنید:

.. code-block:: bash

    $ symfony var:set "CORS_ALLOW_ORIGIN=^`symfony env:urls --first | sed 's#/$##' | sed 's#https://#https://spa.#'`$"

اگر دامنه‌ی شما ``https://master-5szvwec-hzhac461b3a6o.eu.s5y.io/`` باشد، فراخوانی‌های ``sed`` آن را به ``https://spa.master-5szvwec-hzhac461b3a6o.eu.s5y.io`` تبدیل می‌کنند.

همچنین ما نیاز داریم که متغیر محیط ``API_ENDPOINT`` را تنظیم کنیم:

.. code-block:: bash

    $ symfony var:set API_ENDPOINT=`symfony env:urls --first`

commit کرده و مستقر کنید:

.. code-block:: bash
    :class: ignore

    $ git add .
    $ git commit -a -m'Add the SPA application'
    $ symfony deploy

در مرورگر از طریق مشخص‌کردن اپلیکیشن با پرچم، به SPA دسترسی پیدا کنید:

.. code-block:: bash
    :class: ignore

    $ symfony open:remote --app=spa

استفاده از Cordova برای ساخت یک اپلیکیشن گوشی هوشمند
-----------------------------------------------------------------------------------------

.. index::
    single: SPA;Cordova
    single: Apache Cordova
    single: Cordova

**Apache Cordova** ابزاری برای ساخت اپلیکیشن‌های گوشی هوشمند به صورت چندسکویی (cross-platform) است. و یک خبر خوب، این ابزار می‌توان از SPAای که همین الان ساختیم، استفاده کند.

بیایید آن را نصب کنیم:

.. code-block:: bash

    $ cd spa
    $ yarn global add cordova

.. note::

    علاوه بر این شما نیاز دارید که Android SDK را نصب کنید. این بخش تنها Android را ذکر می‌کند اما Cordova با تمام پلتفرم‌های موبایل از جمله iOS کار می‌کند.

ساختار پوشه‌ی اپلیکیشن را ایجاد کنید:

.. code-block:: bash
    :class: answers(n)

    $ cordova create app

و اپلیکیشن Android را تولید کنید:

.. code-block:: bash
    :class: ignore

    $ cd app
    $ cordova platform add android
    $ cd ..

این تمام چیزی است که نیاز دارید. حالا می‌توانید فایل‌های نهایی را بسازید و آن‌ها را به Cordova منتقل کنید:

.. code-block:: bash

    $ API_ENDPOINT=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL --dir=..` yarn encore production
    $ rm -rf app/www
    $ mkdir -p app/www
    $ cp -R public/ app/www

اپلیکیشن را بر روی یک گوشی هوشمند یا یک شبیه‌ساز اجرا کنید:

.. code-block:: bash
    :class: ignore

    $ cordova run android

.. sidebar:: بیشتر بدانید

    * `وب‌سایت رسمی Preact <https://preactjs.com/>`_؛

    * `وب‌سایت رسمی Cordova <https://cordova.apache.org/>`_.

.. _`preact`: https://preactjs.com/
