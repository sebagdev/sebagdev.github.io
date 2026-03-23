---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 1: Heroku'
date: '2020-08-07T20:20:09+02:00'
layout: single
permalink: /2020/08/07/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-1/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Czym jest PaaS i dlaczego warto po niego sięgnąć? W pierwszej części serii budujemy Spring Boot + WebFlux PoC i wdrażamy go na Heroku przez GitLab CI/CD w kilku prostych krokach."
categories:
  - Cloud
tags:
  - cloud
  - heroku
  - java
  - spring-boot
  - paas
  - gitlab-ci
---

Koncepcja PaaS stanowi jeden z modeli chmury obliczeniowej opierającej się o usługi związane z uruchamianiem i zarządzaniem aplikacjami w ramach określonego ekosystemu. Wśród dostawców występuje ogromne zróżnicowanie oferowanych usług i rodzajów wsparcia. W ramach krótkiej serii artykułów chciałbym pokazać jak „ugryźć” temat i jak możemy za w miarę niewielkie pieniądze lub całkowicie za darmo z naszej lokalnej aplikacji wylądować z projektem działającym gdzieś w sieci.

## Heroku

Jako pierwszego dostawcę w tym cyklu chciałbym przedstawić Heroku. Nie posiadają oni własnej infrastruktury, natomiast korzystają z Amazonowej platformy EC2. Od początku Heroku posiadało doskonałe wsparcie dla języka Ruby, z czasem jednak paletę wspieranych technologii poszerzono w tym także o Javę i NodeJS. Podobnie jak większość dostawców także i w tym przypadku mamy podstawowy zestaw narzędzi zupełnie darmowy.

Wdrażana aplikacja uruchamiana jest w oparciu o skonteneryzowane środowisko Linuxowe. Kontener taki w przypadku tego dostawcy nazywany jest „dyno” i to jego użytkowanie i typ stanowi podstawę rozliczenia. Dla przykładu najtańsze „dyno” posiada ograniczenie do 512 MB RAM, wyłącza się po 30 minut nieaktywności, ale… pozwala nam ekspresowo przetestować małego POC’a lub stanowić podstawę prezentacji (jako alternatywa dla ngrok’a).

Konto można założyć za pomocą dość prostego formularza. Warto zainstalować także lokalnego klienta heroku pozwalającego na wygodne zarządzanie całą platformą

```bash
% heroku login
heroku: Press any key to open up the browser to login or q to exit:
Opening browser to https://cli-auth.heroku.com/auth/cli/browser/408b489f-6e5b-4397-ab86-6064cda3fb5c
Logging in... done
Logged in as twoj@mail.com
% heroku apps
You have no apps.
```

W paru krokach spróbuję przedstawić w jaki sposób można zbudować aplikację opartą o framework SpringBoot odpytującą zewnętrzne API. Dość interesujące publiczne API znajdziemy \[[tutaj](https://developer.marvel.com/docs#!/public/getCreatorCollection_get_0)\]. W ramach małego PoC’a spróbujemy odpytać jeden z endpointów.   
 Na początek proponuję wykorzystać \[[Spring initializr](https://start.spring.io)\] i wybrać Javę 14. Z zależności użyję Reactive Web i Lombok’a – tak przygotowany projekt można umieścić w repozytorium Gitlab’owym, co daje dostęp do dość wygodnych i bardzo zaawansowanych pipeline’ów. W podstawowej konfiguracji otrzymujemy również 2000 minut procesora – w sam raz dla mniejszych i mniej aktywnych projektów realizowanych hobbystycznie.

Przygotowujemy źródło niezbędnej konfiguracji (application.yml):

```yaml
marvelclient:
  url: https://gateway.marvel.com:443/
  publicKey: 11111
  privateKey: 22222

```

Zarówno klucz prywatny jak i publiczny możemy uzyskać rejestrując się w serwisie Marvel’a. Pamiętajcie, żeby traktować owe dane jako poufne i nie commitować ich do publicznego repozytorium. Pracując w środowisku chmurowym warto wyrabiać w sobie dobre nawyki związane z security także dla własnego bezpieczeństwa. Przykładowo uzyskanie niepowołanego dostępu do naszych kluczy AWS’owych może skutkować narażeniem na poważne koszty.

Przykładowy kod kliencki – odpytanie jednego z endpointów API:

```java
public class MarvelClient {

    private final WebClient webClient;
    private final MarvelClientProperties marvelClientProperties;

    public MarvelClient(MarvelClientProperties marvelClientProperties) {
        this.marvelClientProperties = marvelClientProperties;
        this.webClient = WebClient.builder()
                .exchangeStrategies(ExchangeStrategies.builder()
                        .codecs(configurer -> configurer
                                .defaultCodecs()
                                .maxInMemorySize(16 * 1024 * 1024))
                        .build())
                .baseUrl(marvelClientProperties.getUrl())
                .build();
    }


    public Mono<marvelresult> getHeroes() {
        return webClient
                .get()
                .uri(uriBuilder -> {
                    long ts = System.currentTimeMillis();
                    return uriBuilder.path("/v1/public/characters")
                            .queryParam("apikey", marvelClientProperties.getPublicKey())
                            .queryParam("limit", 100)
                            .queryParam("ts", ts)
                            .queryParam("hash", HashGenerator.md5(ts + "" + marvelClientProperties.getPrivateKey() + marvelClientProperties.getPublicKey()))
                            .build();
                })
                .accept(MediaType.APPLICATION_JSON)
                .retrieve()
                .bodyToMono(MarvelResult.class);
    }
}</marvelresult>
```

Aby sprawdzić czy wszystko działa i czy z naszej aplikacji da się połączyć z Marvelowym API musimy przygotować najprostszy endpoint:

```java
@Configuration
class MarvelHeroesRouter {

    @Bean
    public RouterFunction<serverresponse> route(MarvelHeroesHandlers marvelHeroesHandlers) {
        return RouterFunctions
                .route(RequestPredicates.GET("/marvelheroes")
                        .and(RequestPredicates.accept(MediaType.APPLICATION_JSON)), marvelHeroesHandlers::marvelHeroes);
    }
}</serverresponse>
```

```java
@Component
@RequiredArgsConstructor
class MarvelHeroesHandlers {

    private final MarvelClient marvelClient;

    public Mono<serverresponse> marvelHeroes(ServerRequest request) {
        return ServerResponse
                .ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(marvelClient.getHeroes(), MarvelResult.class);
    }
}</serverresponse>
```

Przygotowaną aplikację możemy potestować lokalnie i jeśli wszystko działa jak powinno, możemy spróbować podjąć próbę deploymentu.

## Wdrożenie

Gdy mamy przygotowany kod – powinniśmy się zająć miejscem na którym zdeployujemy naszą aplikację. Po zalogowaniu się do panelu Heroku możemy stworzyć nową aplikację:

![](https://sgdev.pl/wp-content/uploads/2020/08/image-1.png)
Kolejna formatka pozwala na podjęcie decyzji, gdzie chcemy by fizycznie znajdował się nasz kontener z aplikacją. Z wszystkich regionów AWS do wyboru są dwa (Stany Zjednoczone i Europa). Możemy też choć nie musimy nadać naszej aplikacji nazwę, po przejściu tego kroku w odpowiedzi mamy gotową aplikację typu Hello World czekającą na zastąpienie ciekawszym (bo naszym) projektem.

![](https://sgdev.pl/wp-content/uploads/2020/08/image-4-1024x583.png)
Wróćmy na chwilę do naszej aplikacji. Sugerowałem użycie gitlaba, ze względu na doskonałe CI/CD, które dostarcza. Dodatkowo Gitlab Pipelines posiadają bardzo niski punkt wejścia.

Przykładowo poniższy kawałek kodu pozwala nam na zdefiniowanie etapu (**stage**) build z jednym zadaniem (**job**) o nazwie *build*. Gitlabowy runner w pierwszym kroku pobiera obraz dockerowy posiadający maven i Javę 14, a następnie uruchamiany jest na nim build (clean package).

```yaml
stages:
  - build

build:
  image: maven:3.6.3-jdk-14
  stage: build
  script:
    - mvn clean package
```

Jak zatem powiązać nasz build z przygotowaną aplikacją w chmurze Heroku? Wystarczy dobrać odpowiednie narzędzie do deploymentu. Jednym z tego typu narzędzi jest dpl używany także przez Travis’a – posiadający wsparcie dla kilkunastu największych dostawców chmurowych (min. Azure i AWS).

Rozbudowa naszego pipeline’u o deployment na Heroku staje się banalnie prosta:

```yaml
stages:
  - build
  - deploy (staging)

build:
  image: maven:3.6.3-jdk-14
  stage: build
  script:
    - mvn clean package

deploy (staging):
  stage: deploy (staging)
  script:
    - apt-get update -yq
    - apt-get install -y ruby-dev
    - gem install dpl
    - dpl --provider=heroku --api-key=$HEROKU_API_KEY --app=$HEROKU_APP_NAME
  only:
    - master
```

Cała magia dzieje się w linijce:

```bash
dpl --provider=heroku --api-key=$HEROKU_API_KEY --app=$HEROKU_APP_NAME
```

HEROKU\_APP\_NAME i HEROKU\_API\_KEY są zmiennymi środowiskowymi, które możemy zdefiniować w ramach naszego projektu na Gitlabie. Nazwę aplikacji na Heroku znajdziemy bez trudu po zalogowaniu do głównego dashboard. Klucz (token) można stworzyć wywołując odpowiednie polecenie z CLI.

```bash
% heroku authorizations:create
Creating OAuth Authorization... done
Client:      <none>
ID:          bc299ac7-a14f-4609-a1cc-bfdf8eebdfa9
Description: Long-lived user authorization
Scope:       global
Token:       11111111-0000-4444-bbbb-4c74376bf966
Updated at:  Fri Aug 07 2020 18:57:15 GMT+0200 (GMT+02:00) (less than a minute ago)</none>
```

Tutaj bardzo ważna uwaga. Powyższy klucz daje pełny dostęp do konta Heroku – nie należy go udostępniać, ani commbitować do publicznych repozytoriów. W przypadku Gitlab – możemy umieścić go w chronionej zmiennej środowiskowej:

![](https://sgdev.pl/wp-content/uploads/2020/08/image-3-1024x253.png)
Chroniona zmienna środowiskowa w Gitlab, sprawi że będzie ona przekazywana ona tylko i wyłącznie do pipeline’ów uruchamianych na chronionych gałęziach (np. master – stąd zadanie deploymentu zostało w zawężone właśnie do tej gałęzi). Dodatkowo dodanie maskowania sprawi, że wartość nie zostanie wypisana w logach runnerów. Oczywiście nie jest to idealne zabezpieczenie, a zatem **nie traktujcie powyższego rozwiązania jako wzorca do powielenia w produkcyjnej aplikacji.**

Po uzupełnieniu zmiennych środowiskowych. Nasz pipeline powinien już działać i wdrożenie aplikacji do chmury Heroku powinno się wykonać.

W logach runner’a na GitLab można podpatrzeć działanie maskowania zmiennych środowiskowych:

```bash
[32;1m$ dpl --provider=heroku --api-key=$HEROKU_API_KEY --app=$HEROKU_APP_NAME[0;m
[33mInstalling deploy dependencies[0m
Successfully installed multipart-post-2.1.1
Successfully installed faraday-1.0.1
Successfully installed rendezvous-0.1.3
Successfully installed netrc-0.11.0
Successfully installed dpl-heroku-1.10.15
5 gems installed
authentication succeeded
checking for app [MASKED]
found app [MASKED]
[33mPreparing deploy[0m
Cleaning up git repository with `git stash --all`. If you need build artifacts for deployment, set `deploy.skip_cleanup: true`. See https://docs.travis-....
```

Ostatecznie aplikacja powinna być dostępna pod URLem **https://&lt;nazwa naszej aplikacji&gt;.herokuapp.com/marvelheroes.** Na tym etapie możemy nie być jeszcze w stanie połączyć się z API Marvel’a które definiowaliśmy na początku. Czego brakuje? Oczywiście kluczy. W ustawieniu ich dla naszej aplikacji pomoże nieoceniony Heroku CLI.

Poniższe polecenia pozwalają na wyświetlenie istniejących zmiennych środowiskowych i ustawienie interesujących nas kluczy dla aplikacji:

```
% heroku -a nazwa-naszej-aplikacji config 
% heroku -a nazwa-naszej-aplikacji config:set marvelclient_privateKey=xxxx 
% heroku -a nazwa-naszej-aplikacji config:set marvelclient_publicKey=yyyy
```

Teraz pozostaje nam tylko odpytać naszą aplikację za pomocą curl’a i powinniśmy uzyskać interesującą nas odpowiedź.

```bash
% curl https://nasza-aplikacji.herokuapp.com/marvelheroes
{"code":200,"status":"Ok","etag":"fff2e8270a046cf31b2b6b62dad51f266328ebf4","data":{"offset":0,"limit":100,"total":1493,"count":100,"results":[{"id":1011334,"name":"3-D Man","description":""},{"id":1017100,"name":"A-Bomb (HAS)","description":"Rick Jones has been Hulk's best bud since day one, but now he's more than a friend...he's a teammate! Transformed by a Gamma energy explosion, A-Bomb's thick, armored skin is just as strong and powerful as it is blue. And when he curls into action, he uses it like a giant bowling ball of destruction! "}
```


## Podsumowanie

W tym pozornie krótkim artykule udało się nam przejść przez wszystkie kroki wymagane do postawienia prostego hobbystycznego projektu.

Od wygenerowania aplikacji, poprzez przygotowanie i skonfigurowanie miejsca w chmurze Heroku aż po zestawienie prostego pipeline’u. Warto zwrócić uwagę, że najwięcej kodu i wysiłku nawet w tak banalnym przykładzie zajęło stworzenie naszej aplikacji. To też ukazuje potęgę rozwiązań typu PaaS -&gt; developer może skupiać się na tym co dla niego najważniejsze, czyli tworzeniu własnego projektu.