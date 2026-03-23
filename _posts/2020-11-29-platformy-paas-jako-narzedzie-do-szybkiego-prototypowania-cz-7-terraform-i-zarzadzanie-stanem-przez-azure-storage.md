---
title: 'Platformy PaaS jako narzędzie do szybkiego prototypowania – cz. 7: Terraform i zarządzanie stanem przez Azure Storage'
date: '2020-11-29T13:46:24+01:00'
layout: single
permalink: /2020/11/29/platformy-paas-jako-narzedzie-do-szybkiego-prototypowania-cz-7-terraform-i-zarzadzanie-stanem-przez-azure-storage/
series: "Platformy PaaS jako narzędzie do szybkiego prototypowania"
excerpt: "Lokalny stan Terraform to problem bezpieczeństwa — sekrety lądują w tfstate czystym tekstem. Rozwiązanie: Azure Storage jako backend dla stanu. Przeniesienie stanu i security best practices w praktyce."
categories:
  - Cloud
tags:
  - cloud
  - azure
  - terraform
  - azure-storage
  - iac
  - security
  - devops
---

Obiecałem w poprzednich rozdziałach, że zadbamy o lekkie usprawnienia security. Lokalny stan w terraform, nie stanowi najlepszego rozwiązania wystarczy, że podejrzymy wygenerowane pliki .tfstate i niezależnie od sposobu przekazania sekretów – lądują one tam czystym tekstem. Dodatkowo jeśli więcej niż jedna osoba pracuje nad zmianami infrastruktury – stan byłby ciągle trzymany niezależnie na różnych systemach. Potencjalnymi rozwiązaniami tego problemu mógłby być wspóldzielony dysk lub nawet lepiej inny rodzaj bezpiecznego składowiska.

## Składowanie stanu w Azure

W tym artykule omówimy sobie podejście wykorzystujące Azure Storage. Zaczynamy od przygotowanie dedykowanej grupy zasobów, jak i konta magazynu (Storage account).

```bash
% az group create --name terraform-state --location westeurope
% az storage account create --resource-group terraform-state --name tstatesgdevpl --sku Standard_LRS --encryption-services blob
```

Przygotowane w ten sposób konto magazynu da nam dostęp do stanu z każdego miejsca na świecie. Klucze dostępowe uzyskamy poprzez polecenie:

```bash
% az storage account keys list --resource-group terraform-state --account-name tstatesgdevpl                           
[
  {
    "keyName": "key1",
    "permissions": "Full",
    "value": "xxxxxxxx"
  },
  {
    "keyName": "key2",
    "permissions": "Full",
    "value": "yyyyyyyyyy"
  }
]
```

Nieco interesującym może wydać się fakt, że w wyniku zostały zwrócone od razu dwa klucze (key1 i key2). Którego z nich powinniśmy zatem używać? Czy różnią się one zakresem uprawnień?   
Otóż… możemy korzystać z obu. Istnienie dwóch kluczy drastycznie ułatwia ich emigrowanie w razie skompromitowania jednego z nich. W przypadku, gdy jeden z nich wyciekł powinniśmy zadziałać w następujący sposób. W każdej aplikacji korzystającej z naszego konta magazynu, podmieniamy klucz na drugi z dostępnych. Następnie rotujemy pierwszy z nich. Istnienie dwóch aktywnych kluczy umożliwia rotację w banalny sposób chroniący nas przed jakimkolwiek downtime’m aplikacji.

Pozyskany w ten sposób klucz dla potrzeb Terraform powinniśmy umieścić w zmiennej środowiskowej:

```bash
export ARM_ACCESS_KEY=xxxxxxxx
```

Następnie przygotowujemy magazyn pod nasz stan termaformowy.

```bash
% az storage container create --name tstate-marvel-aggregator --account-name tstatesgdevpl --account-key $ARM_ACCESS_KEY
{
  "created": true
}
```

Jeśli chcemy obejrzeć efekt powyższych działań możemy zajrzeć np. na Portal Azure. Warto dodać, że jeżeli pominiemy wstrzyknięcie klucza do konta magazynu, a jesteśmy zalogowani z poziomu AzureCLI, polecenie i tak wykona się poprawnie po wyświetleniu nieprzyjemnego ostrzeżenia.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-13-1024x245.png)

Pozostał nam do ułożenia ostatni klocek naszej układanki. Powinniśmy przestawić domyślne zachowanie Terraform, aby potrafił skorzystać z przygotowanego składowiska.  
Mechanizmy obsługujące zarządzanie stanem nazywane są „backendami” i są one w pełni konfigurowalne i podmienialne. Domyślne zachowanie i jego wady poznaliśmy w poprzednich artykułach, jednak ich podmiana okazuje się banalną sprawą – wykorzystuje się do tego specjalną sekcję na początku pliku (dodajmy ją do naszego config.tf):

```hcl
terraform {
  backend "azurerm" {
    resource_group_name   = "terraform-state"
    storage_account_name  = "tstatesgdevpl"
    container_name        = "tstate-marvel-aggregator"
    key                   = "terraform.tfstate"
  }
}
```

Jako, że mamy przygotowaną już całą konfigurację możemy wykonać zestaw poleceń init (reinicjalizacja), a następnie plan i apply, a jako efekt ich wykonania powinniśmy zobaczyć, że stan wylądował w naszym magazynie.

![](https://sgdev.pl/wp-content/uploads/2020/11/image-14-1024x260.png)

## Podsumowanie

W tym artykule przenieśliśmy zarządzanie stanem do chmury, od tego momentu w wygodny sposób i niewielkim kosztem możemy pracować infrastrukturą naszego projektu. Oczywiście planując wdrażanie większych rozwiązań warto rozważyć inne sposoby zarządzania stanem.