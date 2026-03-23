---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 2: Heroku cd.'
date: '2020-09-24T08:31:21+02:00'
layout: single
permalink: /2020/09/24/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-2/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Kontynuacja serii o Heroku — baza danych, zmienne środowiskowe, add-ony i bardziej zaawansowana konfiguracja pipeline'u. Jak postawić kompletną aplikację produkcyjną za darmo."
categories:
  - Cloud
tags:
  - cloud
  - heroku
  - java
  - paas
  - gitlab-ci
    - java
    - paas
---

W poprzednim artykule cyklu zaprezentowaliśmy prostą aplikację, którą w ekspresowy sposób można wdrożyć z wykorzystaniem Heroku. Co w przypadku, gdy oczekujemy czegoś więcej niż tylko połączenia z zewnętrznym API jak np. dodatkowej warstwy persystencji? Oczywiście dostawcy PaaS, aby dostarczyć dojrzałe rozwiązania musieli udostępnić odpowiednie oprzyrządowanie. W przypadku Heroku w sukurs przychodzą nam Add-ony (dodatki), które można w dowolny sposób podłączać do już stworzonej aplikacji. Oczywiście w modelu PaaS nie mamy takiej dowolności w porównaniu do własnoręcznego zarządzania, ale w zamian otrzymujemy prekonfigurowane komponenty, które działają właściwie od razu.  
Jednak każdy z dodatków jakie możemy użyć na platformie ma swoją cenę i pewne ograniczenia regionalne, ale w zamian odpadają wszelkie dodatkowe koszty związane z administracją i utrzymaniem.

Przykładowo dodając bazę postgres na platformie Heroku wystarczy wywołać:

```bash
Creating heroku-postgresql on ⬢ guarded-hollows-81209... free
Database has been created and is available
 ! This database is empty. If upgrading, you can transfer
 ! data from another database with pg:copy
Created postgresql-globular-66581 as DATABASE_URL
Use heroku addons:docs heroku-postgresql to view documentation
```

Jak widzimy utworzenie nowej bazy danych jest bajecznie proste, a podobne działanie naturalnie można podjąć także z poziomu UI. Chciałbym tu zwrócić uwagę na pewną rzecz – mianowicie jak dostać się do tak skonfigurowanej bazy z poziomu naszej aplikacji?

```bash
<pre class="lang:sh decode:true ">Created postgresql-globular-66581 as DATABASE_URL
```

Rąbka tajemnicy uchyla powyższa linijka – „klucz” do bazy znany również jako „connection string” ląduje w zmiennej konfiguracyjnej DATABASE\_URL

```text
postgres://<RANDOMIZED_USERNAME>:<SO_SECRET_PASSWORD>@ec2-12-345-678-901.eu-west-1.compute.amazonaws.com:5432/<RANDOM_DB_NAME>
```

Już na pierwszy rzut oka widać, że baza została posadowiona na silniku EC2 od Amazona. Dodatkowo wszystko co z nią związane zostało wygenerowane sposób pseudolosowy. Spróbujmy zatem wykorzystać tą świeżo przygotowaną bazę danych w naszej testowej aplikacji.

Zacznijmy od przygotowania modelu – chcielibyśmy przetrzymywać w niej użytkownika:

```java

@Data
@NoArgsConstructor
@AllArgsConstructor
@Table("user_")
class User {
    @Id
    Long id;
    String username;
}
interface UserRepository extends ReactiveCrudRepository<User, Long> {
}
```

Aby przygotować tabelę, w tym przykładzie chciałem posłużyć się technologią wersjonowania bazy danych Flyway, która stanowi głównego konkurenta do już dość dojrzałego Liquibase. Osobiście nigdy nie miałem preferencji w kierunku jakiejkolwiek z tych technologii, jednak zawsze odnosiłem wrażenie, że Flyway pozwala szybciej wystartować, a o to po części chodzi w tym przykładzie. Zatem zacznijmy od utworzenia skryptu V1\_\_Create\_new\_table\_user.sql:

```sql
DROP TABLE IF EXISTS "user_";
CREATE TABLE "user_" ( id SERIAL PRIMARY KEY, username VARCHAR(100) NOT NULL);
```

Wydaje się, że jeśli chodzi o warstwę dostępu do danych to mamy wszystko, żeby pójść dalej z naszym przykładem. Obsłużmy zatem w kodzie zapis nowego użytkownika:

```java
@Component
@RequiredArgsConstructor
public class UserHandlers {

    private final UserRepository userRepository;

    public Mono<ServerResponse> createUser(ServerRequest serverRequest) {
        Mono<User> productToSave = serverRequest.bodyToMono(User.class);
        return ServerResponse.status(HttpStatus.CREATED)
                .contentType(MediaType.APPLICATION_JSON)
                .body(productToSave.flatMap(userRepository::save), User.class);
    }
}
```

Powyższy handler należy spiąć z istniejącym routingiem w naszej aplikacji, po szybkim refactoringu otrzymujemy:

```java
@Bean
public RouterFunction<ServerResponse> route(MarvelHeroesHandlers marvelHeroesHandlers, UserHandlers userHandlers) {
    return RouterFunctions
            .route(RequestPredicates.GET("/marvelheroes")
                    .and(RequestPredicates.accept(MediaType.APPLICATION_JSON)), marvelHeroesHandlers::marvelHeroes)
            .andRoute(RequestPredicates.POST("/user")
                    .and(RequestPredicates.accept(MediaType.APPLICATION_JSON)), userHandlers::createUser);
}
```

Zwróćcie proszę uwagę, że w tym przypadku podobnie jak i w poprzednim przykładzie, gdzie odpytywaliśmy Marvel’owskie API staram posługiwać się paradygmatem reaktywnym, który nieśmiało pojawia się w coraz większej ilości projektów. Nie zawsze też w miejscach w których rzeczywiście jest wymagany i potrzebny (jak np. tutaj :-)).

Celem przetestowania repozytorium przygotujemy prosty test oparty o TestContainers, technologia ta wymaga od was utrzymywania od was demona Docker’owego zarówno na stacji roboczej jak i na pipeline’ach, jednak w odróżnieniu od zagnieżdżonych baz danych pozwala na przeprowadzenie testów integracyjnych w otoczeniu praktycznie tożsamym z produkcją.

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    @Container
    public static PostgreSQLContainer postgreSQLContainer = new PostgreSQLContainer(DockerImageName.parse("postgres:9.6"))
            .withDatabaseName("integration-tests-db")
            .withUsername("sa")
            .withPassword("sa");

    @DynamicPropertySource
    static void jdbcProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.r2dbc.url", () -> postgreSQLContainer.getJdbcUrl().replace("jdbc:", "r2dbc:"));
        registry.add("spring.r2dbc.username", postgreSQLContainer::getUsername);
        registry.add("spring.r2dbc.password", postgreSQLContainer::getPassword);
        registry.add("spring.flyway.url", postgreSQLContainer::getJdbcUrl);
        registry.add("spring.flyway.user", postgreSQLContainer::getUsername);
        registry.add("spring.flyway.password", postgreSQLContainer::getPassword);
    }

    @Autowired
    UserRepository userRepository;

    @Test
    void testInsertUser() {
        User user = new User();
        user.setUsername("lipa");
        this.userRepository.saveAll(Arrays.asList(user, user))
                .as(StepVerifier::create)
                .expectNextCount(2)
                .verifyComplete();
    }
}
```

Metoda oznaczona @DynamicPropertySource umożliwia przeciążenie konfiguracji danymi w runtime pozyskanymi ze świeżo postawionego kontenera bazodanowego. Alternatywnie można do tego wykorzystać inicjalizator kontekstu:

```java
@ContextConfiguration(initializers = PostgresContainerInitializer.class).
```

Gdy nasze testy już działają należałoby spróbować połączyć wszystko z bazą danych. Uważny czytelnik zapewne zauważył, że DATABASE\_URL dostarczany przez Heroku ma jednak nieco inny format niż formaty obsługiwane przez Spring’a (i JDBC), jednak i ten problem został poniekąd zaadresowany, ale po kolei…

W przypadku gdy korzystamy z aplikacji Javowej na platformie (po autodetekcji przy pierwszym deploy’u), Heroku automatycznie wzbogaca ją o „buildpack”, czyli niezbędny zbiór skryptów i narzędzi takich jak np. maven. Szczegóły można znaleźć w dokumentacji: <https://devcenter.heroku.com/articles/java-support>.

Dodatkowo buildpack automatycznie będzie próbował utworzyć zmienne środowiskowe SPRING\_DATASOURCE\_USERNAME, SPRING\_DATASOURCE\_PASSWORD, SPRING\_DATASOURCE\_URL.  
 Oczywiście nawet w przypadku gdy korzystamy w nieco inny sposób z JDBC jesteśmy w stanie sobie poradzić np. przetwarzając początkowy DATABASE\_URL. [https://devcenter.heroku.com/articles/connecting-to-relational-databases-on-heroku-with-java#using-the-database\_url-in-plain-jdbc](https://devcenter.heroku.com/articles/connecting-to-relational-databases-on-heroku-with-java#using-the-database_url-in-plain-jdbc)

Oczywiście możemy wykorzystać polecenie heroku config:get i ręcznie przeciążyć ustawienia aby osiągnąć konfigurację „pod nas”, jednak w pewnym sensie byłaby to forma tightcoupling’u, której raczej chcemy unikać.

Wzbogaceni o tą wiedzę spróbujmy skonfigurować Flyway, tym co daje nam platforma Heroku:

```yaml
spring:
  flyway:
    user: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}
    url: ${SPRING_DATASOURCE_URL}
    baseline-on-migrate: true
    locations: classpath:/db/migration
    check-location: true
```

Próba przygotowania tabeli powinna zakończyć się powodzeniem, co jednak z konfiguracją naszego reaktywnego repozytorium? URL’e różnią się od tych oczekiwanych – nawet w przypadku testu integracyjnego kłuje w oczy linijka w którym podmieniłem jdbc na r2dbc. Dla uproszczenia przykładu i nie tworzenia wszystkiego manualnie po prostu nadpisałem connectionFactory i obsługę properties, co nie jest być może rozwiązaniem idealnym, ale szybkim.

```java
@Configuration
public class ApplicationConfiguration extends AbstractR2dbcConfiguration {

    @Override
    @Bean
    public ConnectionFactory connectionFactory() {
        return ConnectionFactoryBuilder
                .of(new HerokuR2dbcProperties(r2dbcProperties), () -> null)
                .build();
    }

    @RequiredArgsConstructor
    private class HerokuR2dbcProperties extends R2dbcProperties {
        private final R2dbcProperties r2dbcProperties;

        public String getName() {
            return r2dbcProperties.getName();
        }

        public String getUrl() {
            if (r2dbcProperties.getUrl().startsWith("r2dbc:postgres:")) {
                return r2dbcProperties.getUrl().replace("r2dbc:postgres:", "r2dbc:postgresql:");
            }
            return r2dbcProperties.getUrl();
        }
}
```


Mając przygotowaną aplikację – możemy zaobserwować, że pipeline kończy się już na etapie testów:\[ERROR\] UserRepositoryIntegrationTest ? ContainerLaunch Container startup fai

```bash
[ERROR]   UserRepositoryIntegrationTest ? ContainerLaunch Container startup failed
```

Testcontainers które wykorzystaliśmy w projekcie wymagają Docker’a a zatem musimy wzbogacić nasz pipeline (.gitlab-ci.yml) o obraz Docker in Docker.

```yaml
services:
  - docker:dind

variables:
  DOCKER_HOST: "tcp://docker:2375"
  DOCKER_DRIVER: overlay2
```

I voill’a 🙂 Celem potwierdzenia, że nasza baza danych już działa:

```bash
% curl -verbose -X POST https://sgdev-pl-marvelaggregator-XXXX.herokuapp.com/user -H 'Content-Type: application/json' -d '{ "username": "Kopytko 123!"}'
```

w odpowiedzi powinniśmy otrzymać naszego świeżo utworzonego użytkownika z nowo nadanym id’kiem:

```json
{"id":6,"username":"Kopytko 123!"}
```

## Podsumowanie  


 W tych dwóch krótkich artykułach opowiadających o platformie Heroku poruszyliśmy całą gamę tematów związaną z modelem PaaS i utworzyliśmy szkielet aplikacji która:

- posiada podstawowy pipeline CI/CD
- działa w sposób reaktywny
- swoje testy integracyjne opiera o kontenery testowe

I to wszystko w zaledwie w dwóch krótkich artykułach – co jest całkiem niezłym wynikiem. Niestety wszystko ma swoją cenę i koszt(poza rachunkiem) – obnażyliśmy też jedną z największych słabości PaaS, czyli konieczność dostosowania się do waszego dostawcy. Planując budowę i obsługę aplikacji w tym modelu należy dokładnie przeanalizować co dostawcy oferują i czy jesteśmy w stanie poradzić sobie z ograniczeniami.