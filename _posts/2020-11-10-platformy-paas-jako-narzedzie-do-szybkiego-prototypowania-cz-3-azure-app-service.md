---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 3: Azure App Service'
date: '2020-11-10T19:50:05+01:00'
layout: single
permalink: /2020/11/10/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-3-azure-app-service/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Przenosimy się na Azure. Azure App Service jako platforma dla kontenerów Dockerowych — pierwsze kroki, konfiguracja resource group, App Service Plan i wdrożenie aplikacji Spring Boot z GitLab."
categories:
  - Cloud
tags:
  - cloud
  - azure
  - azure-app-service
  - java
  - docker
  - paas
---

W poprzednim artykułach opisywałem Heroku, który idealnie pokrywa przypakdi hobbystyczne, co jednak kiedy chcemy przeprowadzić pewien mniej lub bardziej zaawansowany proof-of-concept w firmie, a management nie specjalnie ma ochotę na zawieranie nowych umów i sugeruje skorzystanie z już posiadanych przez nas subskrybcji.

W środowiskach korporacyjnych dominuje głównie dwóch dostawców pokrywających większość rynku i wyznaczających standardy dla konkurencji. Pierwszy z nich – AWS zapoczątkował wielką rewolucję chmurową, która odmieniła cały świat IT, zmuszając nawet najbardziej konserwatywne firmy jak banki do opracowania swoich rozwiązań (private cloud) lub wykorzystania chmury publicznej w części prowadzonych projektów (hybrid cloud).  
Microsoft Azure powstał jako odpowiedź na rosnącą popularność chmur obliczeniowych i szybko stał się faworytem korporacji ze względu na ogromną ilość oferowanych usług w modelu SaaS (jak Office 365 i Dynamics). Jak już wspominałem wcześniej nawet jeśli twoja korporacja nie posiada jeszcze projektów realizowanych w chmurze Azure – jest duża szansa, że posiada subskrybcję Office365, czy Microsoft Dynamics.

W tym artykule chciałbym kontynuować naszą zabawę z modelem PaaS. Spróbujmy przenieść aplikację, którą zbudowaliśmy pod chmurę Heroku na Azure App Service. Jeśli nie popełniliśmy zasadniczych błędów projektowych, to nie powinniśmy stać się uzależnieni od dostawcy (ang. vendor lock-in) i migracja całości nie powinna kosztować nas zbyt dużo wysiłku.

Większość serwisów PaaS można uniezależnić całkowicie od języka programowania i platformy, poprzez odpowiednią konteneryzację. Jeśli pamiętacie heroku addons jak i buildpack’i wspomniane w poprzedniej części – to uniezależnienie się od nich stanowi klucz do uwolnienia naszego serwisu od określonego dostawcy i wspieranych przez niego technologii.

Przykładowa architektura:

![](https://azurecomcdn.azureedge.net/mediahandler/acomblog/media/Default/blog/AppServiceOnLinux.png)
#### Na czym zatem polega konteneryzacja?

Jest to rodzaj wirtualizacji na poziomie systemu operacyjnego (OS-level virtualization) i wbrew powszechnej opinii nie stanowi nowego wynalazku – jej wczesnym pierwowzorem były jail’e i chroot znany z systemów uniksowych (1982). W świecie backend developerów dopiero pojawienie się Docker’a upowszechniło konteneryzację. Od tej pory tworzone usługi i oprogramowanie mogły w całości uniezależnić się stack’u systemowego czy technologicznego.

Obraz Dockerowy można złożyć z kilku warstw za pomocą kilku wbudowanych poleceń takich jak FROM, CMD, COPY, ARG. Dla przykładu:

```dockerfile
FROM ubuntu:18.04
CMD echo "lipa"
```

Po utworzeniu pliku Docker file jw. zbudowanie obrazu i uruchomienie kontenera sprowadza się do dwóch poleceń:

```bash
% docker build -t hellow .
Sending build context to Docker daemon   43.7MB
Step 1/2 : FROM ubuntu:18.04
18.04: Pulling from library/ubuntu
171857c49d0f: Pull complete 
419640447d26: Pull complete 
61e52f862619: Pull complete 
Digest: sha256:646942475da61b4ce9cc5b3fadb42642ea90e5d0de46111458e100ff2c7031e6
Status: Downloaded newer image for ubuntu:18.04
 ---> 56def654ec22
Step 2/2 : CMD echo "lipa"
 ---> Running in 1b4ba8ccc056
Removing intermediate container 1b4ba8ccc056
 ---> 61967ac5ca38
Successfully built 61967ac5ca38
Successfully tagged hellow:latest
% docker run hellow       
lipa
```

## Rozbudowa aplikacji

W przypadku naszej aplikacji moglibyśmy wyodrębnić całą logikę poprzez przekopiowanie utworzonych artefaktów w następujący sposób:

```dockerfile
FROM openjdk:14-jdk-alpine
RUN addgroup -S marvelaggregator && adduser -S marvelaggregator -G marvelaggregator
USER marvelaggregator:marvelaggregator
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

W zasadzie poza skopiowaniem artefaktów i uruchomieniem naszej aplikacji zmieniamy jeszcze kontekst użytkownika na pozbawionego przywilejów ROOT’a (dobra praktyka).  
Po próbie budowy, możemy niestety zaobserwować dość nieprzyjemny wyjątek:

```bash
% docker build -t sgdevpl/marvelapp .
% docker run sgdevpl/marvelapp:latest
% .....
....
Caused by: java.lang.IllegalArgumentException: Could not resolve placeholder 'DATABASE_URL' in value "r2dbc:${DATABASE_URL}"
        at org.springframework.util.PropertyPlaceholderHelper.parseStringValue(PropertyPlaceholderHelper.java:178) ~[spring-core-5.2.7.RELEASE.jar!/:5.2.7.RELEASE]

```

W dość wymowny sposób pokazuje o czym zapomnieliśmy podczas uruchamiania/tworzenia obrazu.   
Do uruchomienia potrzebujemy kilku dodatkowych zmiennych środowiskowych – w końcu w Heroku użyliśmy aż pięciu (!). Powinny one zostać przekazane do kontenera. Przygotujmy zatem plik w następującym formacie:

```text
DATABASE_URL=postgres://dbuser:dbpass@ec2-11-11-11-11.eu-west-1.compute.amazonaws.com:5432/dfdbf
marvelclient_privateKey=123456abcdef.....
marvelclient_publicKey=43567890abcdef....
SPRING_DATASOURCE_URL=jdbc:postgresql://ec2-11-11-11-11.eu-west-1.compute.amazonaws.com:5432/dfdbf
SPRING_DATASOURCE_USERNAME=dbuser
SPRING_DATASOURCE_PASSWORD=dbpass
```

Wartości można oczywiście zabrać z naszej wcześniej utworzonej aplikacji na Heroku. Na powyższym przykładzie wychodzi też cena pokazanej wcześniej automagii.  
Czy projektując aplikację nie pod konkretnego dostawcę podjęlibyśmy identyczną decyzję, jeśli chodzi o konfigurację podłączenia do bazy danych?  
Zapewne zależałoby nam na rozdzieleniu pewnych elementów, dodatkowo moglibyśmy uniknąć zaprezentowanego wcześniej hack’a.  
Kolejna próba z uwzględnieniem zmiennych środowiskowych i przekierowaniem portów pozwala już na lokalne uruchomienie skonteneryzowanej aplikacji:

```bash
% docker run --env-file ./env.list -p 8080:8080  sgdevpl/marvelapp:latest
```

Teraz wypadałoby poszerzyć nasz projekt o budowanie obrazu w pipelinie. Na szczęście przy oparciu naszego amatorskiego projektu o Gitlaba uzyskaliśmy możliwość skorzystania z ichniejszego rejestru obrazów (rozmiar całego repozytorium jak i rejestru posiada ograniczenie 10GB).  
Użycie pipeline’ów w Gitlab sprawia, że uzyskanie do niego dostępu również jest banalnie proste poprzez predefiniowane zmienne:

```yaml
docker-build:
  stage: dockerize
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_PIPELINE_ID
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - master
```

Przygotowany etap wykonuje budowanie obrazu przy założeniu, że wcześniej odłożyliśmy je:

```yaml
build:
  image: maven:3.6.3-jdk-14
  stage: build
  script:
    - mvn clean package
  artifacts:
    paths:
      - target/*.jar
```

![](https://sgdev.pl/wp-content/uploads/2020/11/image-1024x403.png)
Po uruchomieniu kilku pipeline’ów zobaczymy, że rejestr zapełnia się kolejnymi tagami z kolejnymi wersjami aplikacji. Jak zatem powinniśmy decydować, która z nich powinna zostać zdepluyowan’a na nasze docelowe produkcyjne środowisko?

Można dodać do pipeline’u kolejny etap, którego celem będzie wykonanie właśnie tej operacji tj. odpowiednie tagowanie ostatniego build’a.

```yaml
deploy (Azure):
  stage: deploy latest
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_PIPELINE_ID
    RELEASE_TAG: $CI_REGISTRY_IMAGE:latest
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u $CI_REGISTRY_USER $CI_REGISTRY --password-stdin
    - docker tag $IMAGE_TAG $RELEASE_TAG
  only:
    - master
```

Et voilla. Nowy stage sprawi, że pod tagiem latest zawsze będzie się ukrywał najnowszy obraz pipeline’u masterowego.

Ktoś mógłby zadać pytanie, ale gdzie w tym wszystkim Azure? Dlaczego przygotowywać aplikację w ten sposób?   
W następnej części cyklu zobaczymy uzysk z tych wszystkich operacji, ponadto zastanowimy się czy dałoby się podejść do tematu nieco inaczej.