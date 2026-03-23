---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 8: zarządzanie infrastrukturą z poziomu GitLab'
date: '2020-11-30T18:49:07+01:00'
layout: single
permalink: /2020/11/30/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-8-zarzadzanie-infrastruktura-z-poziomu-gitlab/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Finał serii — zamykamy pętlę i pozwalamy GitLab CI/CD zarządzać nie tylko kodem, ale i infrastrukturą. Service Principal, pipeline Terraformowy i pełna automatyzacja od commita do chmury."
categories:
  - Cloud
tags:
  - cloud
  - azure
  - terraform
  - gitlab-ci
  - iac
  - devops
  - paas
---

W poprzednich częściach pokazaliśmy jak możemy w naszej prototypowanej aplikacji szybko zbudować pełny pipeline CI/CD. Następnie udało nam się przeprowadzić deployment na chmurach Heroku i Azure, oraz cały kod przedstawić w postaci Terraformowego skryptu o stanie współdzielonym w zewnętrznym magazynie. Moglibyśmy teraz ruszyć o krok dalej, pozwalając naszemu repozytorium nie tylko zarządzać kodem, ale też dbać o aplikowanie zmian w chmurze.

W celu użycia nie-personalnego konta i posiadania pewnej kontroli nad tworzoną aplikacją powinniśmy utworzyć dedykowaną jednostkę usługi (Service Principal). Na potrzeby artykułu możemy traktować ją jako rodzaj niespersonalizowanej tożsamości.

```bash
% az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/{SubscribtionID}"
Creating a role assignment under the scope of "/subscriptions/<SubscriptionID>"
  Retrying role assignment creation: 1/36
{
  "appId": "<UUID>",
  "displayName": "azure-cli-2020-11-29-13-22-27",
  "name": "http://azure-cli-2020-11-29-13-22-27",
  "password": "<PASSWORD>",
  "tenant": "<UUID>"
}
```

Przypisanie jednostce usługi roli „Contributor” pozwoli jej na zarządzanie dostępem do zasobów, a jednocześnie uniemożliwi zmianę w ustawieniach ról AzureRBAC i nadawanie nowych uprawnień.

Następnie możemy przygotować zarys Pipeline’u (.gitlab-ci.yml), którego ogólna idea pozwoli na przygotowanie i przetestowanie zmian w infrastrukturze naszego projektu.

```yaml
image:
  name: hashicorp/terraform:0.13.5
before_script:
  - rm -rf .terraform
  - terraform --version
  - terraform init
 
stages:
  - validate
  - plan
  - apply

validate:
  stage: validate
  script:
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan
  dependencies:
    - validate

apply:
  stage: apply
  script:
    - terraform apply -input=false
  only:
    - master
  dependencies:
    - plan

```

Pierwsze etapy pozwolą na weryfikację i wywołanie planowania zmian, aplikacja nastąpi dopiero na pipeline master. Jako, że będziemy działali w ramach automatycznego zadania, musimy wyłączyć tryb interaktywny poprzez dodanie parametru (-input=false). Dodatkowo powinniśmy uzupełnić wszystkie zmienne środowiskowe wymagane przez aplikację.

Zatem począwszy od zmiennych aplikacyjnych TF\_VARS\_\* możemy rozpocząć przygotowywanie naszych sekretów. Po raz kolejny chciałbym zwrócić uwagę, że wciąż nie zaadresowaliśmy wielu problemów związanych z bezpieczeństwem, a jedynie skupiamy na się na poprawie automatyzacji tworzenia infrastruktury.

Przy próbie wywołania zadania planowania nawet po ustawieniu zmiennych, możemy zobaczyć następujący błąd:

```bash
Error: Error building AzureRM Client: Azure CLI Authorization Profile was not found. Please ensure the Azure CLI is installed and then log-in with `az login`.
```

Nadszedł czas na wykorzystanie uprzednio przygotowanego Service Principal. Provider Azure wymaga pięciu zmiennych:

![](/wp-content/uploads/2020/11/image-15-1024x372.png)
Zasadniczo wszystkie poza id naszej subskrybcji i ACCESS\_KEY znanym z poprzedniego artykułu można pozyskać po utworzeniu roli. ARM\_CLIENT\_ID to tak naprawdę appId, a ARM\_CLIENT\_SECRET to otrzymane w odpowiedzi hasło.

Gdy zaaplikujemy wszystkie wspomniane zmiany – nasz Pipeline powinien się zazielenić 🙂

## Podsumowanie

Czas chyba odpowiedzieć sobie na dwa pytania. Czy tak naszpikowany zmiennymi pipeline może być w ogóle zarządzalny? Czy to w ogóle bezpieczne?

Cóż – na pewno zmienne aplikacyjne nie powinny być ręcznie konfigurowane poprzez Gitlab’a i być tam przechowywanymi. Powinniśmy korzystać z bezpiecznego składowiska typu Vault i z niego na żądanie pobierać wartości.   
Co do ogólnego bezpieczeństwa… Zadania na publicznym GitLab uruchamiane są w Dockerze i nie współdzielą stanu pomiędzy wywołaniami. Pozwalamy jednak zewnętrznemu dostawcy na trzymanie danych naszej jednostki usługi. W planie darmowym, niewiele jest sposobów na obejście tej wady, a więc musielibyśmy posłużyć się integracją z Vault. Wszystko zależy od rozmiaru projektu i naszej determinacji w celu zabezpieczenia go.