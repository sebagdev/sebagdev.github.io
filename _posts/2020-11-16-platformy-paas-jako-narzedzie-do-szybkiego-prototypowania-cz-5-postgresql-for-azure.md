---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 5: PostgreSQL for Azure'
date: '2020-11-16T21:35:32+01:00'
layout: single
permalink: /2020/11/16/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-5-postgresql-for-azure/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Dodajemy bazę danych do naszej chmurowej aplikacji — Azure Database for PostgreSQL. Konfiguracja, połączenie z aplikacją Spring Boot, Spring Data JPA i zarządzanie migracjami w środowisku chmurowym."
categories:
  - Cloud
tags:
  - cloud
  - azure
  - postgresql
  - java
  - spring-boot
  - paas
---

W poprzedniej części naszego cyklu udało nam się z sukcesem przenieść naszą aplikację pod postacią obrazu Docker’owego na Azure. Jednak na Heroku wciąż pozostała baza danych, co sugeruje, że nasza migracja jest wciąż niepełna. W tym artykule chciałbym skupić się na postawieniu dedykowanej bazy danych w chmurze Microsoft’u.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-9-1024x820.png)

W portalu po wpisaniu frazy Azure Database for PostgreSQL będziemy mogli wybrać jeden z czterech rodzajów wdrożeń. Każdy z nich różni się skalą, możliwościami i kosztami.

Dla przykładowej aplikacji zrealizowanej przez nas w tym cyklu artykułów interesuje nas pojedynczy serwer, który powinien wytrzymać obciążenie aplikacji prototypowej.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-10-1024x936.png)
Kolejne ekrany pozwolą na dokonfigurowanie wybranej opcji.Pamiętajmy, że w naszym przykładzie zagadnienia security spłycam do minimum, zatem starajcie się chronić hasła które ustawiacie.

Aprowizacja bazy potrwa chwilę, jednak jak się można domyślić utworzona przez nas baza będzie posiadała tylko użytkownika administracyjnego. Odwiedzenie bazy danych pozwoli na lekkie poluzowanie polityki bezpieczeństwa. Zacznijmy od lekkiej liberalizacji ustawień Firewall’a poprzez zezwolenie na dostęp z innych usług Azure. Jeśli chcemy pracować z naszym klientem bazy danych(psql) powinniśmy także odblokować nasz adres IP.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-11-1024x642.png)
Do dalszej konfiguracji konieczny już będzie klient psql, ale jeśli nie chcemy go instalować wystarczy, że skorzystamy z AzureCLI otrzymamy wygodny terminal zaszyty w przeglądarce internetowej.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-12-1024x36.png)
Nie ma znaczenia czy dalszą konfigurację wykonamy z terminala, czy z przeglądarki, do połączenia potrzebujemy jednak klienta psql:

```bash
% psql --host=<mój host>.postgres.database.azure.com --port=5432 --username=<mój admin>@<mój host> --dbname=postgres
```

Następnie należy utworzyć kolejno bazę danych, rolę i odpowiednie przywileje:

```sql
postgres=> CREATE DATABASE sgdevblog;
CREATE DATABASE
postgres=> CREATE ROLE sgdevblogger WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD '<moje hasło>';
CREATE ROLE
postgres=> GRANT ALL PRIVILEGES ON DATABASE sgdevblog TO sgdevblogger;
GRANT
```

Po utworzeniu nowej roli, możemy zaktualizować nasze połączenia. Przykładowy connection string będzie wyglądał następująco:

```text
jdbc:postgresql://<nasza instancja bazy>.postgres.database.azure.com:5432/<baza danych>
```

I tyle, po restarcie nasza aplikacja powinna już uruchamiać się i korzystać z bazy świeżo utworzonej w Azure.

Jeśli dotrwaliście aż dotąd, można uznać że to ostatni punkt migracji. Jedyną pozostałością jaka została po Heroku, to dług w postaci dziwnie podzielonych properties zawierających poświadczenia bazodanowe.