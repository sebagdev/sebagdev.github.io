---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 4: Azure App Service cd.'
date: '2020-11-14T12:24:01+01:00'
layout: single
permalink: /2020/11/14/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-4-azure-app-service/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Rozbudowa aplikacji na Azure App Service — zmienne środowiskowe, monitorowanie, skalowanie i integracja z Azure Container Registry. Jak zautomatyzować cały cykl deploy przez GitLab CI/CD."
categories:
  - Cloud
tags:
  - cloud
  - azure
  - azure-app-service
  - java
  - docker
  - gitlab-ci
  - paas
---

Zanim ruszymy dalej z artykułami chmurowymi, krótkie ostrzeżenie -korzystając z AWS, czy Azure powinniśmy pamiętać, że subskrybujemy usługi płatne – do założenia konta potrzebna jest karta debetowa lub kredytowa (i niestety wszelkiego rodzaju prepaidy odpadają). Dodatkowo wiele oferowanych usług klasy Enterprise po prostu kosztuje krocie, należy więc za każdym razem sprzątać testowane przez nas usługi i kontrolować koszty. Powinniśmy też zadbać o ochronę naszego konta i dostępu do zasobów tj. włączyć MFA, chronić tokeny, stosować zasadę minimalnych uprawnień itd.  
Jakiekolwiek zaniedbanie w tej dziedzinie może narazić nas na spore koszty.  
Sami dostawcy oferują na swoich stronach zestawy najlepszych praktyk dot. bezpieczeństwa <https://aws.amazon.com/architecture/security-identity-compliance/> i <https://azure.microsoft.com/mediahandler/files/resourcefiles/security-best-practices-for-azure-solutions/Azure%20Security%20Best%20Practices.pdf> z którymi zdecydowanie warto się zapoznać.

Zarówno w Azure jak i w AWS jako nowi użytkownicy możemy liczyć na pakiet powitalny w postaci sporego zestawu darmowych usług (w pewnych granicach oczywiście). Bazowa oferta pozwoli jednak na spokojny deployment prototypowanej aplikacji w modelu PaaS za free (<https://azure.microsoft.com/pl-pl/pricing/details/app-service/windows/>). Darmowa oferta nie jest tak dobra w porównaniu do Heroku, ale jest to poniekąd cena za ogromną uniwersalność całej platformy. Ufff… wydaje mi się, że na tym etapie możemy zakończyć informacje wstępne i spróbować założyć konto. Nie będę opisywał tego etapu, ale posiadanie już konta w usługach Microsoft skraca drastycznie cały proces – wystarczy uregulować kwestie dot. płatności i możemy działać.

## Konfiguracja

Po pierwszym zalogowaniu do portalu azure (http://portal.azure.com/) zobaczymy następujące wręcz „komiksowe” menu do zarządzania zasobami.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-1-1024x233.png)
W wyszukiwarce znajdujemy App Services i klikamy + Add. Pojawia się menu w którym możemy właściwie wyklikać całą naszą aplikację:

![](https://sgdev.pl/wp-content/uploads/2020/11/image-2-1024x803.png)
Gdy dopiero zaczynamy naszą przygodę z Azure zapewne będziemy musieli utworzyć pierwszą grupę zasobów. Z początku ten obiekt może wydawać się czymś nadmiarowym, bo dlaczego nie utworzyć zasobu bezpośrednio na naszej subskrybcji? Spójrzmy na konto Azure z perspektywy na przykład dużej firmy – możemy mieć w niej zdefiniowane, wiele aplikacji i środowiska, często użytkowanych przez rozmaite zespoły. W takim heterogonicznym świecie łatwo stracić możliwość zarządzania całością i jakiejkolwiek kontroli kosztów. Wydzielenie grup zasobów np. z podziałem na aplikacje pozwala uzyskać nam pełną kontrolę nad naszą chmurą i płatnościami.

Z ustawień dotyczących deploymedntu pozostaje odpowiedni dobór nazwy jak i typu samego deploymentu:

![](https://sgdev.pl/wp-content/uploads/2020/11/image-3-1024x327.png)
Czym różni się sposób publikacji *Code* vs *Docker Container*? Code pozwala na zdefiniowanie wdrożenia w analogiczny sposób jak w przypadku Heroku – opieralibyśmy się o zbudowanie aplikacji ze źródeł po stronie Azure’a. Przygotowaliśmy sobie obraz Dockerowy po to by maksymalnie uniezależnić się od konkretnego dostawcy, zatem skorzystamy z tego i wybierzemy *Docker Container.*

Kolejny ekran konfiguracji oczekuje wartości, które powinniśmy już znać z poprzedniej odsłony naszego cyklu:

![](https://sgdev.pl/wp-content/uploads/2020/11/image-4-1024x526.png)
Przygotowaliśmy już obraz, posiadamy rejestr w Gitlab, wypadałoby wprowadzić właściwy adres rejestru, nazwę użytkownika, hasło, a także obraz – zdobyć je można w Gitlabie w Settings -&gt; Repository -&gt; Deploy Tokens.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-5-1024x792.png)
W scope tworzonego token’a interesuje nas tylko możliwość odczytu obrazów w repozytorium – z punktu widzenia deploymentu jest to zupełnie wystarczające. Hasło pojawi się jednorazowo, jeśli nie skopiujemy go pozostaje nam utworzenie kolejnego token’u.

Pełną nazwę obrazu jak i adres repozytorium, uzyskamy na zakładce **Container Registry**.

Po wypełnieniu formatki i kliknięciu na URL możemy zauważyć, że nasza aplikacja nie startuje.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-7-1024x153.png)
Oglądając **Container Settings** możemy zobaczyć logi ze startu – brakuje nam zmiennych środowiskowych. Na szczęście odłożyliśmy potrzebne dane budując kontener.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-8-1024x1012.png)
Uzupełnienie konfiguracji powinniśmy wykonać w sekcji **Configuration**. Oprócz zmiennych środowiskowych, które wykorzystywaliśmy uruchamiając obraz Dockerowy powinniśmy dodać jeszcze jedną: **WEBSITES\_PORT**. W skrócie wskazuje ona na którym porcie nasłuchuje nasza aplikacja – w tym przypadku należy ustawić ją na 8080.

W zasadzie to tyle – ponowne odwiedziny URL z naszą aplikacją sprawią, że będziemy w stanie już korzystać z naszego API.