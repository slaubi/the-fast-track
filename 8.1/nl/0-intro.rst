Waar gaat het over?
===================

Symfony is een van de meest succesvolle PHP-projecten. Het is zowel een krachtig full-stack framework als een populaire set van herbruikbare componenten.

Sinds de release van Symfony 2.0 in 2011, heeft het project nu een bepaald niveau van maturiteit bereikt. Ik merk dat alles wat we de afgelopen jaren hebben gedaan een geheel vormt. Nieuwe low-level componenten, high-level integraties met andere software en tools die ontwikkelaars helpen om productiever te zijn. De development-ervaring  is aanzienlijk verbeterd zonder dat dit ten koste gaat van de flexibiliteit. Het is nog nooit zo leuk geweest om Symfony voor een project te gebruiken.

Als Symfony nieuw voor je is, laat dit boek je stap voor stap de kracht van het framework en mogelijkheden voor het verbeteren van je productiviteit in het ontwikkelen van een applicatie zien.

Als je al een Symfony-ontwikkelaar bent, kun je het herontdekken. Het framework is de laatste jaren drastisch geëvolueerd en de ontwikkel-ervaring is aanzienlijk verbeterd. Ik heb het gevoel dat veel Symfony-ontwikkelaars nog steeds "vastzitten" in oude gewoonten en dat ze het moeilijk vinden om de nieuwe methodieken te omarmen. Dit is begrijpelijk. Het tempo van onze evolutie is onthutsend. Wanneer er fulltime aan een project wordt gewerkt hebben ontwikkelaars geen tijd om alles wat er in de community gebeurt te volgen. Ook ikzelf kan niet alles op de voet volgen.

En het gaat niet alleen om nieuwe manieren om dingen te doen. Het gaat ook om nieuwe componenten: HTTP-client, Mailer, Workflow, Messenger. Dat zijn grote veranderingen. Deze componenten moeten de manier waarop je over een Symfony-applicatie denkt, veranderen.

Ik voel ook de behoefte aan een nieuw boek omdat het web zich sterk heeft ontwikkeld. Onderwerpen zoals `API's`_ , `SPA's`_ , `containerisatie`_ , `Continuous Deployment`_ , en vele andere zouden nu moeten worden besproken.

Is een boek nog wel jouw tijd waard nu een LLM de code voor je kan schrijven? Ik stelde mezelf die vraag terwijl ik aan deze editie werkte. Mijn antwoord is een volmondig ja. Ik gebruik elke dag AI-tools, en ze schrijven inmiddels bijna al mijn code; maar ze hebben niets veranderd aan wat er echt toe doet. Je moet nog steeds de architectuur van je applicatie begrijpen, beslissen wat je gaat bouwen, de gegenereerde code beoordelen en ontdekken wat subtiel verkeerd is. Je kunt geen code beoordelen die je niet begrijpt; en je kunt een tool niet sturen richting een architectuur die je je niet kunt voorstellen. De reden om boeken te lezen is oordeelsvermogen, geen nostalgie.

Oordeelsvermogen vereist begrip, en begrip is precies wat dit boek opbouwt: een compleet mentaal model van een Symfony-applicatie, van de eerste commit tot productie. Met dat model in je hoofd weet je, de volgende keer dat een LLM een controller of een Messenger-handler voor je genereert, of het klopt, en waarom. We zullen er zelfs eentje aan het werk zetten binnen de applicatie zelf om reacties te modereren.

Ik zal jouw kostbare tijd niet verdoen met lange paragrafen, noch lange uitleg over kernconcepten. Dit boek gaat meer over het traject. Waar te beginnen, welke code te schrijven, het wanneer en hoe. Dit met als doel jouw interesse te wekken voor belangrijke onderwerpen en jou te laten beslissen of je meer wilt leren en je verder wilt verdiepen.

Ik wil de bestaande documentatie ook niet zomaar overnemen. De kwaliteit daarvan is uitstekend. Ik zal naar de documentatie verwijzen in het gedeelte "Verder Gaan" aan het einde van elke stap/hoofdstuk. Beschouw dit boek als een lijst met verwijzingen naar meer bronnen.

Het boek beschrijft het bouwen van een applicatie, van nul tot productie. Maar we zullen niet alles ontwikkelen om de applicatie geheel gereed te maken voor productie. Het resultaat zal niet perfect zijn. We zullen bochten afsnijden. Hoogstwaarschijnlijk gaan we zelfs voorbij aan het afhandelen van sommige edge cases, validaties of tests. Best practices zullen niet altijd gerespecteerd worden. Toch zullen we bijna elk aspect van een modern Symfony-project behandelen.

Toen ik aan dit boek begon te werken, was het allereerste wat ik deed het programmeren van de uiteindelijke applicatie. Ik was onder de indruk van het resultaat en de snelheid die ik kon volhouden bij het toevoegen van functies, met zeer weinig moeite. Dat is te danken aan de documentatie en het feit dat Symfony je niet in de weg zit. Ik ben er van overtuigd dat Symfony nog op vele manieren verbeterd kan worden (en ik heb ook aantekeningen gemaakt over mogelijke verbeteringen), maar de ervaring van de ontwikkelaar is veel beter dan een paar jaar geleden. Ik wil iedereen erover vertellen.

Het boek is verdeeld in stappen. Elke stap is onderverdeeld in substappen. Als het goed is, zijn ze snel te lezen. Maar nog belangrijker, ik nodig je uit om te programmeren terwijl je leest. Schrijf de code, test het, rol het uit, pas het aan.

En tenslotte: aarzel niet om hulp te vragen als je vast komt te zitten. Misschien kom je een edge case tegen, of zit er een typefout in jouw code die moeilijk te vinden en te corrigeren is. Stel vragen. We hebben een fantastische community op `Slack`_ en `GitHub`_.

Klaar om te programmeren? Veel plezier!

.. _`API's`: https://en.wikipedia.org/wiki/Application_programming_interface
.. _`SPA's`: https://en.wikipedia.org/wiki/Single-page_application
.. _`containerisatie`: https://en.wikipedia.org/wiki/OS-level_virtualization
.. _`Continuous Deployment`: https://en.wikipedia.org/wiki/Continuous_deployment
.. _`Slack`: https://symfony.com/slack
.. _`GitHub`: https://github.com/symfony/symfony/discussions
