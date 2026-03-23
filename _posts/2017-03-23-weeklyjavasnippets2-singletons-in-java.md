---
title: 'Java Snippets #2 – Singletony w Javie'
date: '2017-03-23T20:00:00+01:00'
layout: single
permalink: /2017/03/23/weeklyjavasnippets2-singletons-in-java/
series: "Java Snippets"
excerpt: "Singleton to jeden z najczęściej stosowanych wzorców GoF, ale też jeden z najczęściej źle implementowanych. Przegląd podejść: od prostego, przez thread-safe, aż po implementację z enum — i dlaczego ta ostatnia bywa najlepsza."
categories:
  - Java
tags:
  - java
  - wzorce
  - singleton
  - gof
  - snippet
    - snippets
    - wzorce
---

#### Singleton – wzorzec czy antywzorzec?

Wśród programistów, często spotykam się z opinią, że Singleton stanowi antywzorzec. Niestety, rzadko też słyszę dobre argumenty podpierające tą tezę. Fakt, jest on jednym z najpowszechniejszej stosowanych wzorców projektowych i jednym z pierwszych jaki poznaje każdy programista. Konsekwencjami tego jest olbrzymia ilość błędnych implementacji, które krążą po repozytoriach i potrafią przyprawić o prawdziwy zawrót głowy podczas czytania kodu i poszukiwania przyczyny problemów.

Do czego właściwie to służy ? Opisując najkrócej jak się da posłużę się cytatem z książki GoF (*Gang of Four*):

> Gwarantuje, że klasa będzie miała tylko jeden egzemplarz, i zapewnia globalny dostęp do niego.

I to tyle… proste prawda? Okazuje się, że nie. Przede wszystkim wzorzec jest masowo nadużywany, a o tym kiedy go stosować, a kiedy unikać można napisać spory artykuł na pewno nie nadający się do WeeklyJavaSnippets. Gdy coś implementujesz, a nie masz pewności czy powinieneś/powinnaś użyć singletona skontaktuj się z lekarzem lub farmac… Wróć! Z seniorem lub architektem oczywiście 🙂

#### Jak wygląda zatem poprawna i szybka implementacja wzorca Singleton?

```java
public class MySingleton {

    private MySingleton(){ } // wyłączamy konstruktor domyślny

    private static class SingletonHolder {
        private static final MySingleton INSTANCE = new MySingleton();
    }

    public static MySingleton getInstance(){
        return SingletonHolder.INSTANCE;
    }

}
```

Prywatna, wewnętrzna i statyczna klasa SingletonHolder sprawia, że obiekt klasy MySingleton jest tworzony dopiero przy pierwszym wywołaniu getInstance(). Ponadto w przypadku tej implementacji nie ma możliwości utworzenia dwóch instancji przy współbieżnym dostępie, nie ma też dodatkowych opóźnień związanych z synchronizacją.  
Wśród błędnych implementacji często pojawiają się różne wariacje checked-locked, double-checked locked, a można je łatwo rozpoznać po pewnej ilości if’ów mających zapewnić lazy-loading. Zainteresowanych tymi wariantami odsyłam np. do [wiki](https://en.wikipedia.org/wiki/Double-checked_locking). Z góry ostrzegam, że obecnie nawet automaty często oznaczają je jako błędne.

Inną ciekawą implementacją singletona jest oparcie go o **enum**. Wersję taką zaproponował Joshua Bloch w swojej książce:

```java
public enum MyEnumSingleton {
    INSTANCE;

    private MyEnumSingleton(){ }
    public void doWork(){ }
}
```

To tyle na dzisiaj. Bawcie się dobrze używając singletona i przede wszystkim pamiętajcie o tym, aby go nie nadużywać! Standardowo, dzięki wszystkim którzy dotarli aż tutaj 🙂