O czym jest ta książka?
=========================

Symfony jest jednym z najpopularniejszych projektów PHP. Jest to zarówno stabilny, pełnoprawny framework, jak i popularny zestaw komponentów wielokrotnego użytku.

Od czasu wydania Symfony 2.0 w 2011 roku, projekt osiągnął dojrzałość. Czuję, że wszystko, co osiągnęliśmy w ciągu ostatnich kilku lat, łączy się w piękną całość. Nowe, niskopoziomowe komponenty, wysokopoziomowa integracja z innym oprogramowaniem, przydatne narzędzia - zwiększają produktywność. Te same zadania zrealizujesz wygodniej i tak samo elastycznie, jak kiedyś. Wykorzystanie Symfony w projekcie jeszcze nigdy nie było tak wygodne.

Jeśli dopiero zaczynasz swoją przygodę z Symfony, książka ta pokaże siłę fameworka i jego pozytywny wpływ na produktywność, krok po kroku.

Jeśli używasz już Symfony, dzięki tej książce odkryjesz ten framework na nowo. W ciągu ostatnich kilku lat framework zasadniczo się zmienił, co bardzo poprawiło wygodę jego użycia. Mam wrażenie, że wiele osób używających Symfony nadal tkwi przy starych nawykach i trudno jest im przyjąć nowe sposoby tworzenia aplikacji z Symfony. Rozumiem niektóre z tych powodów. Tempo ewolucji jest oszałamiające. Pracując w pełnym wymiarze godzin nad projektem, często nie ma czasu na śledzenie wszystkiego, co dzieje się w społeczności. Wiem o tym z pierwszej ręki. Sam nie będę udawał, że potrafię nadążyć za wszystkim.

I nie chodzi tu tylko o nowe sposoby tworzenia tych samych rzeczy. Chodzi również o nowe, przełomowe komponenty, takie jak: Klient HTTP, Mailer, Workflow, Messenger, które powinny zmienić sposób, w jaki myślisz o aplikacji Symfony.

Czuję również potrzebę wydania nowej książki, ponieważ sposób tworzenia aplikacji mocno się zmienił. Powinniśmy podejmować tematy takie jak `API`_, `SPA`_, `konteneryzacja`_, `Continuous Deployment`_ i wiele innych.

Czy książka jest wciąż warta Twojego czasu teraz, gdy LLM potrafi napisać kod za Ciebie? Zadałem sobie to pytanie, pracując nad tym wydaniem. Moja odpowiedź brzmi: zdecydowanie tak. Codziennie korzystam z narzędzi AI i to one piszą teraz niemal cały mój kod; ale nie zmieniły tego, co najważniejsze. Wciąż musisz rozumieć architekturę swojej aplikacji, decydować, co zbudować, przeglądać wygenerowany kod i wychwytywać to, co jest subtelnie błędne. Nie da się przejrzeć kodu, którego się nie rozumie; ani nie da się pokierować narzędziem ku architekturze, której nie potrafisz sobie wyobrazić. Argumentem za czytaniem książek jest osąd, a nie nostalgia.

Osąd wymaga zrozumienia, a zrozumienie to właśnie to, co buduje ta książka: kompletny model myślowy aplikacji Symfony, od pierwszego commita po produkcję. Mając ten model w głowie, następnym razem, gdy LLM wygeneruje dla Ciebie kontroler albo handler Messengera, będziesz wiedzieć, czy jest poprawny i dlaczego. Uruchomimy nawet jeden z nich wewnątrz samej aplikacji, aby moderował komentarze.

Twój czas jest cenny. Nie oczekuj długich akapitów, ani długich wyjaśnień na temat podstawowych pojęć. Ta książka jest formą zaproszenia do podróży: od czego zacząć? który kod napisać? kiedy? jak? Postaram się wzbudzić w Tobie zainteresowanie ważnymi tematami i to Ty zdecydujesz, czy chcesz się czegoś więcej nauczyć i drążyć dalej.

Nie chcę też powielać istniejącej dokumentacji, której jakość jest doskonała. W sekcji "Idąc dalej" na końcu każdego rozdziału zamieściłem obszerne odniesienia do dokumentacji. Myśl o tej książce, jako o zbiorze odnośników do większej liczby zasobów.

Książka opisuje tworzenie aplikacji krok po kroku. Nie zbudujemy jednak na jej kartach gotowej aplikacji. Nie pokażemy wszystkiego, zastosujemy skróty myślowe, być może pominiemy niektóre skrajne przypadki (ang. edge cases), walidację lub testy. Nie zawsze uwzględnimy najlepsze praktyki. Za to zajmiemy się niemal każdym aspektem nowoczesnego projektu Symfony.

Kiedy zaczynałem pracę nad tą książką, pierwszą rzeczą, jaką zrobiłem, było napisanie aplikacji końcowej. Byłem pod wrażeniem rezultatu i szybkości dodawania nowych funkcji, przy bardzo małym nakładzie pracy. To dzięki dokumentacji i temu, że Symfony wie, kiedy zejść Ci z drogi. Jestem pewien, że Symfony można jeszcze poprawić (i odnotowałem już kilka możliwych ulepszeń), ale wygoda kodowania jest o wiele większa niż kilka lat temu. Chcę się tym ze wszystkimi podzielić.

Książka jest podzielona na etapy. Każdy etap jest podzielony na podetapy. Etapy powinno dać się szybko czytać, ale, co ważniejsze, zachęcam Cię do kodowania w trakcie czytania. Napisz kod, wypróbuj go, wdróż i dopracuj.

Ostatnia rzecz, ale równie ważna - nie wahaj się poprosić o pomoc. Możesz trafić na skrajny przypadek lub literówkę w kodzie, która może być trudna do znalezienia i naprawienia. Zadawaj pytania. Mamy wspaniałą społeczność na `Slack`_ i `GitHub`_.

Przechodzimy do kodowania? Baw się dobrze!

.. _`API`: https://en.wikipedia.org/wiki/Application_programming_interface
.. _`SPA`: https://en.wikipedia.org/wiki/Single-page_application
.. _`konteneryzacja`: https://en.wikipedia.org/wiki/OS-level_virtualization
.. _`Continuous Deployment`: https://en.wikipedia.org/wiki/Continuous_deployment
.. _`Slack`: https://symfony.com/slack
.. _`GitHub`: https://github.com/symfony/symfony/discussions
