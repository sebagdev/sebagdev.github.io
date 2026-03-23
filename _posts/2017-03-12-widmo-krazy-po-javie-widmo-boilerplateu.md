---
title: "Widmo krąży po Javie... widmo boilerplate'u"
date: '2017-03-12T19:00:15+01:00'
layout: single
permalink: /2017/03/12/widmo-krazy-po-javie-widmo-boilerplateu/
excerpt: "Boilerplate w Javie to problem, który od lat frustruje programistów. Lombok to biblioteka, która eliminuje gettery, settery, konstruktory i wiele więcej za pomocą adnotacji — zerowy narzut, maksymalny efekt."
categories:
  - Java
tags:
  - java
  - boilerplate
  - lombok
  - czysty-kod
---

#### Problem (?)

W zasadzie ktokolwiek zajmujący się Javą, a mający w swoim życiorysie romans z platformą .NET przyzna, że Java nie jest zwięzłym językiem. Już sam kod akcesorów, które trzeba tworzyć/generować za każdym razem potrafi doskonale zaciemnić nam obraz klasy. Dla przykładu znany z konkurencyjnej platformy mechanizm Properties doskonale adresuje ten problem, skracając boilerplate do minimum, jak i pozostawia programistę w pełnej kontroli:

```java
class Entity {

    public String Name {get; set;}

    private String surname;
    public String Surname {  
        get { 
            Console.WriteLine("Accessing Surname"); 
            return this.surname;
        }  
        set {
            Console.WriteLine("Setting Surname"); 
            this.surname = value;
        }
    }    


    public void Print(){
        Console.WriteLine("My name is: {0} {1}", this.Name, this.Surname);
    }

}
```

Dla przykładu odpowiadający kawałek kodu w Javie:

```java
public class Entity {

    private String name;
    private String surname;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getSurname() {
        System.out.println("Accesing surname");
        return surname;
    }

    public void setSurname(String surname) {
        System.out.println("Setting surname");
        this.surname = surname;
    }

    public void print(){
        System.out.printf("My name is: %s %s", getName(), getSurname() );
    }
}
```

Kolejną rzeczą, na którą dość często tracimy czas jest „implementowanie” .equals() i .hashCode(). Doskonałym wsparciem w tym zakresie jest IDE – mamy rozbudowane możliwości generacji, jak i biblioteki, jednakże gdy przychodzi do utrzymania takiego kodu cóż… wtedy robi się mniej ciekawie. Dodajmy że przy większych encjach programista jest zmuszony do wpatrywania się w ścianę kodu, po czym no właśnie… pozostaje dylemat. *dodałem/usunąłem/zmieniłem pole, przegenerować to wszystko, ale zaraz czyż to nie jest jakieś paskudne legacy code? i co mam teraz z tym zrobić ? czemu właściwie te 10 testów eksplodowało ?*

```java
// Classic style
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;

    Entity entity = (Entity) o;

    if (name != null ? !name.equals(entity.name) : entity.name != null) return false;
    return surname != null ? surname.equals(entity.surname) : entity.surname == null;
}

@Override
public int hashCode() {
    int result = name != null ? name.hashCode() : 0;
    result = 31 * result + (surname != null ? surname.hashCode() : 0);
    return result;
}

// Java 7+/Guava Style
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Entity entity = (Entity) o;
    return Objects.equals(name, entity.name) &&
            Objects.equals(surname, entity.surname);
}

@Override
public int hashCode() {
    return Objects.hash(name, surname);
}
```

#### Potencjalne rozwiązania ?

Powyżej zamieściłem dwa szybkie przykłady na najbardziej pospolity typ boilerplate kodu wynikający z samej natury języka. W Javie jak to w Javie zawsze istnieje więcej niż jeden sposób na rozwiązanie problemu. Osobiście słyszałem o dwóch: [AutoValue](https://github.com/google/auto/tree/master/value) od Google’a, którego główne założenie było trochę inne, choć rozwiązuje po drodze ten sam problem. Przyznaję nie używałem, ani nie znam nikogo, kto korzystał niestety, może w następnym projekcie zapróbkuję i dam znać jak poszło.

Drugie rozwiązanie któremu właściwie chciałem poświęcić ten artykuł jest [ProjectLombok](https://projectlombok.org/). Opiera się on podobnie jak i AutoValue o wykorzystanie adnotacji pozwalających na ograniczenie ilości kodu, jakie trzeba napisać/wygenerować. Biblioteka wstrzykuje podczas kompilacji opierając się o adnotacje kod i w zasadzie tyle. Warto zwrócić uwagę, że pozbawiony lomboka kod będzie niekompilowalny. Co do czytelności, myślę że sami osądzicie to po poniższym przykładzie – cały kod mieści się na jednym ekranie 🙂

```java
@ToString(exclude = {"dontWantThisInToString"})
@EqualsAndHashCode(exclude = {"dontWantThisInHashCode"})
public class EntityLombok {

    @Getter @Setter
    private String name;
    private String surname;
    @Getter @Setter
    private String dontWantThisInHashCode;
    @Getter @Setter
    private String dontWantThisInToString;


    public String getSurname() {
        System.out.println("Accesing surname");
        return surname;
    }

    public void setSurname(String surname) {
        System.out.println("Setting surname");
        this.surname = surname;
    }

    public void print(){
        System.out.printf("My name is: %s %s", getName(), getSurname() );
    }

    public static void main(String[] args){
        EntityLombok entity = new EntityLombok();
        entity.setName("Testname");
        entity.setSurname("Testsurname");
        entity.setDontWantThisInHashCode("not wanted in hash code");
        entity.setDontWantThisInToString("not wanted in to string");
        entity.print();
        System.out.println(entity);
    }
}
```

Uruchomienie powyższej klasy spowoduje wypisanie następujących treści na konsolę:

```java
Setting surname
Accesing surname
My name is: Testname TestsurnameAccesing surname
EntityLombok(name=Testname, surname=Testsurname, dontWantThisInHashCode=not wanted in hash code)
```

Jednakże jak ta magia jest możliwa ? Cóż… lombok.jar musi być dostępny w classpath. Ponadto musi być zainstalowana wtyczka do IDE (istnieją wtyczki do IntelliJ, Eclipse i Netbeans). Tylko dzięki temu z poziomu edytora będzie można skorzystać z wygenerowanych metod, a także (np. wybierając z Outline w Eclipse lub poprzez find usages z IntelliJ) wyszukać wszystkie miejsca użycia danej metody. Co ciekawe lombok posiada także wiele innych przydatnych rzeczy jak np. autoimplementacją wzorca Builder:

```java
@Builder
@ToString(exclude = {"dontWantThisInToString"})
@EqualsAndHashCode(exclude = {"dontWantThisInHashCode"})
public class EntityLombok {

//....

EntityLombok entityLombok = EntityLombok.builder()
        .name("Testname")
        .surname("Testsurname")
        .dontWantThisInHashCode("no no no")
        .dontWantThisInToString("no not in toString")
        .build();
entityLombok.print();
System.out.println(entityLombok);
```

Co więcej można nawet wprowadzić nowe słowo kluczowe 🙂

```java
val list = new ArrayList<String>();
// stanie się odpowiednikiem:
final ArrayList<String> list = new ArrayList<String>();
```

Lombok oczywiście posiada dużo więcej możliwości, stąd zachęcam do samodzielnej eksploracji tematu na stronie projektu. Jak ze wszystkim sugeruję korzystanie z ostrożnością i wybranie tylko tych feature’ów które naprawdę mogą wam pomóc w danym projekcie. Dobrze byłoby ustalić listę tolerowanych i pożądanych elementów i bezwzględnie przestrzegać tych zasad podczas implementacji. Ostatni przykład (*val*) był swoistym rodzajem ostrzeżenia, mówiącego, że jest cienka granica przed tym kiedy przestajemy programować w Javie, a zaczynamy programować w Lomboku.

#### Kilka luźnych myśli na koniec

Nie chcę pisać, że lombok jest rozwiązaniem pozbawionym wad. Wprost przeciwnie – wnosi całą gamę problemów, które bez niego by nie istniały, ponadto wnosi również zagrożenia, a sam fakt, że jego implementacja opiera się o niepubliczne API pozwala wątpić w sensowność tego rozwiązania. Nie będę też pisał, że stosuję go w każdym projekcie bo tak nie jest. Moim zdaniem jest ciekawym sposobem na łatanie pewnych „braków” językowych, a przede wszystkim braku lukru składniowego jaki wszyscy kochamy. Co ciekawe na jednym ze spotkań SJUG’a ([Silesia Java User Group](https://www.meetup.com/Silesia-JUG/)) w luźnej dyskusji poruszyłem ten temat (stosowanie/niestosowanie) i przyznam, że większość znających używających wyrażała mieszane uczucia. Tak czy inaczej myślę, że spróbować warto. Wnioski i ewentualne spostrzeżenia wyciągniecie sami.

#### Dla zainteresowanych

W środowisku toczy się od lat debata odnośnie używania adnotacji. Są wszędzie, że traktujemy je jak coś naturalnego, ale czy na pewno tak powinno być ? Lombok, Spring, EJB, JPA wszystko opiera się o adnotacje. Poniżej krótki wykład przedstawiający nieco inne spojrzenie:  
<iframe allowfullscreen="allowfullscreen" frameborder="0" height="315" loading="lazy" src="https://www.youtube.com/embed/-6zT60l5hDc?ecver=1" width="560"></iframe>