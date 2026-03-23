---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 6: Terraform'
date: '2020-11-24T21:55:35+01:00'
layout: single
permalink: /2020/11/24/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-6-terraform/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Koniec z klikaniem w portalu Azure — czas na Infrastrukturę jako Kod. Terraform pozwala zdefiniować całą infrastrukturę w plikach tekstowych i zarządzać nią jak kodem. Praktyczny przykład dla Azure."
categories:
  - Cloud
tags:
  - cloud
  - azure
  - terraform
  - iac
  - paas
  - devops
---

W poprzednich częściach cyklu przeszliśmy szybką drogę do stworzenia aplikacji mobilnej w oparciu o Azure App Service. Wszystkie przykłady przedstawiałem w oparciu o interfejs graficzny, którego intuicyjność pozwala poznać ogrom możliwości chmury. W pewnych okolicznościach wyklikiwanie wszystkiego może okazać się niepraktyczne, a co gorsza może prowadzić do poważnych błędów.

Terraform jest narzędziem pozwalającym wdrażać infrastrukturę jako kod (Infrastructure as a code). Opiera swoje działanie o ogromną ilość wtyczek (nazywanych provider’ami) do wszystkich popularnych dostawców chmurowych i nie tylko. W zasadzie każda usługa posiadająca swoje publiczne API może posiadać swojego provider’a. Niepełną ich listę można znaleźć pod adresem: https://www.terraform.io/docs/providers/index.html

Wszystko co chcemy zautomatyzować za pomocą terraform musi zostać zaimplementowane w języku HCL. Język ten ma charakter deklaratywny i ma wyrazić docelowy stan, a w momencie aplikacji zmian powoduje szereg wywołań API używanych providerów. Aplikacja zmian powoduje też odłożenie stanu jaki udało się osiągnąć np. w plikach terraform.tfstate, co pozwala na synchronizację i planowanie kolejnych zmian w zasobach.

Czy dałoby się jednak uniknąć przechowywania stanu? Cóż w pewnych scenariuszach byłoby to możliwe, jednak bardzo komplikowałoby to całą ideę szczególnie podczas operacji aktualizacji/zastąpienia zasobów, świadomości stworzonych metadanych i relacji między nimi oraz z powodów czysto wydajnościowych.

## Praktyka

Spróbujmy przejść od tej suchej teorii do praktyki. Do utworzenia zasobów w Azure będzie nam potrzebny provider azurerm. Sprobujmy też przygotować dedykowaną grupę zasobów:

```hcl
provider "azurerm" {
  version = "~> 2.1.0"
  features {}
}

resource "azurerm_resource_group" "example" {
  name     = "sgdevpl_blog_via_tf"
  location = "westeurope"
}
```


Aby zaaplikować zmiany powinniśmy wykonać:

```bash
% terraform plan
% terraform apply
```

Przetworzenie całości trochę potrwa i powinniśmy być zalogowani za pomocą lokalnego klienta Azure dostępnego na wszystkie platformy (<https://docs.microsoft.com/pl-pl/cli/azure/install-azure-cli>). Do tak utworzonego pliku możemy przygotować definicję planu usługi określawjącą typ i rodzaj zasobów konsumowanych przez naszą aplikację:

```hcl
resource "azurerm_app_service_plan" "test" {
  name                = "example-appserviceplan"
  location            = "${azurerm_resource_group.example.location}"
  resource_group_name = "${azurerm_resource_group.example.name}"
  kind                = "Linux"
  reserved            = true

  sku {
    tier = "Free"
    size = "F1"
  }
}
```


Proszę zwróćcie uwagę w jaki sposób połączono lokalizację oraz grupę zasobów z uprzednio utworzonym planem.

Oczywiście po edycji pliku terraform ponownie wykonujemy polecenia plan i apply. W pliku stanu oraz w logach możemy podejrzeć addytywne zachowanie terraforma.

Po zdefiniowaniu planu przyszedł czas na przygotowanie usługi, która będzie z nego korzystać i pomieści naszą aplikację. Poniższe linijki mieszczą właściwie wszystkie ekrany związane z konfiguracją naszej aplikacji, a także musiały się w niej znaleźć zmienne środowiskowe przechowujące wszystkie sekrety. O tym w jaki sposób powinniśmy zarządzać sekretami opowiemy sobie w innym artykule tego cyklu.

```hcl
resource "azurerm_app_service" "main" {
  name                = "sg-blog-appservice"
  location            = "${azurerm_resource_group.example.location}"
  resource_group_name = "${azurerm_resource_group.example.name}"
  app_service_plan_id = "${azurerm_app_service_plan.test.id}"

  site_config {
    app_command_line          = ""
    use_32_bit_worker_process = true
    linux_fx_version          = "DOCKER|registry.gitlab.com/sgdev_pl/marvelaggregator:latest"
  }

  app_settings = {
    "WEBSITES_ENABLE_APP_SERVICE_STORAGE" = "false"
    "DOCKER_REGISTRY_SERVER_URL"          = "https://registry.gitlab.com"
    "WEBSITES_CONTAINER_START_TIME_LIMIT" = "230"
    "WEBSITES_ENABLE_APP_SERVICE_STORAGE" = "false"
    "WEBSITES_PORT"                       = "8080"
    "DOCKER_REGISTRY_SERVER_USERNAME"     = "${var.docker_registry_server_username}"
    "DOCKER_REGISTRY_SERVER_PASSWORD"     = "${var.docker_registry_server_password}"
    "DATABASE_URL"                        = "${var.database_url}"
    "marvelclient_privateKey"             = "${var.marvel_privateKey}"
    "marvelclient_publicKey"              = "${var.marvel_publicKey}"
    "SPRING_DATASOURCE_URL"               = "${var.spring_datasource_url}"
    "SPRING_DATASOURCE_USERNAME"          = "${var.spring_datasource_username}"
    "SPRING_DATASOURCE_PASSWORD"          = "${var.spring_datasource_password}"
  }
}

```

Kiedy podejmiemy próbę aplikacji zmian okaże się że nasz kod jest niepoprawny:

```bash
Error: Reference to undeclared input variable

```

Oczywiście zmienne które zdefiniowaliśmy powinniśmy gdzieś zadeklarować. Utarło się że zmienne w terraform powinno umieścić się w plikach **variables.tf** lub **vars.tf**. Deklaracja wszystkich zmiennych wygląda w zbliżony sposób jak w innych językach programowania, a dobry opis ułatwia zarządzanie nimi.

```hcl
variable "marvel_privateKey" {
  description = "marvel_privateKey"
  type        = string
}
variable "marvel_publicKey" {
  description = "marvel_publicKey"
  type        = string
}

variable "database_url" {
  description = "DATABASE_URL as stated in Heroku"
  type        = string
}

variable "spring_datasource_url" {
  description = "datasoure url as stated in Heroku"
  type        = string
}

variable "spring_datasource_username" {
  description = "datasoure username as stated in Heroku"
  type        = string
}

variable "spring_datasource_password" {
  description = "datasoure password as stated in Heroku"
  type        = string
}

variable "docker_registry_server_username" {
  description = "docker registry server username"
  type        = string
}

variable "docker_registry_server_password" {
  description = "docker registry server password"
  type        = string
}

```

Po dodaniu powyższych definicji przy próbie wykonania planu pojawi się ładne zapytanie o kolejne wartości:

```bash
% terraform plan
var.database_url
  DATABASE_URL as stated in Heroku

  Enter a value:

```

Takie zachowania można obejść na przykład poprzez zmienne środowiskowe. Muszą one przyjąć konkretny format tj. posiadać prefiks **TF\_VAR\_**. Jeśli uda nam się zestawić pełny zestaw zmiennych, mozolnie wpisując je w konsoli utworzenie aplikacji powinno udać się bez większych przeszkód.

## Podsumowanie

W tym artykule udało nam się ująć definicję naszej aplikacji w dwóch plikach. Początkowo takie usprawnienie może wydawać się nadmiarowe, jednak zarządzając większą ilością usług możemy ujrzeć zalety automatyzacji i algorytmizacji naszej pracy.