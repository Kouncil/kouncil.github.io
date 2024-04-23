---
layout:    post
title:     "Kouncil podłączenie pod Amazon MSK"
date:      2024-04-24 7:00:00 +0100
published: true
didyouknow: false
lang: pl
lang-ref:  kafka-aws-msk

author:    pbelke
image:     /assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk.png
description: "Czy kiedykolwiek zastanawiałeś się, jak utworzyć i podłączyć się pod klaster Amazon MSK? W tym poście pokażemy Ci jak to zrobić na dwa sposoby."
tags:
- kouncil
- programming
- kafka
- amazon
- aws msk
---

Sposobów na uruchomienie Kafki jest wiele. Od najprostszych, takich jak uruchomienie lokalne, aż po te oparte o technologie cloudowe. W niniejszym wpisie przyjrzymy się jednemu z rozwiązań chmurowych, jakim jest Amazon Managed Streaming for Apache Kafka, w skrócie Amazon MSK. Przejdziemy krok po kroku od momentu utworzenia klastra Kafki, aż po podpięcie go pod Kouncil. W chmurze uruchomiony zostanie również Kouncil. W tym artykule pokażemy najbardziej niskopoziomy sposób, wystartujemy od pustego Centosa, ale jeśli masz juz AKS albo Beanstalka to możesz uruchomić Kouncil jako element architektury, jako ze sprowadza się to do uruchomienia kontenera dockerowego.

## Utworzenie klastra
Utworzenie klastra Kafki na Amazon MSK jest bardzo proste. Musimy przejść do strony [Amazon MSK](https://console.aws.amazon.com/msk/home) i dalej wybrać **Create cluster**. Pojawi się konfigurator klastra. Na potrzeby tego artykułu wybiorę opcję **Quick create** oraz typ klastra **Provisioned**. W przypadku takiej konfiguracji trzeba odczekać kilkanaście minut, aby klaster był dostępny.

![Konfiguracja klastra Kafki](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-1.png)
<span class="img-legend">Konfiguracja klastra Kafki</span>

![Utworzony klaster Kafki](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-2.png)
<span class="img-legend">Utworzony klaster Kafki</span>

## Utworzenie uprawnień - IAM
Aby utworzyć regułę uprawnień musimy przejść do strony [IAM](https://console.aws.amazon.com/iam/home#/home) i z menu bocznego wybrać **Policies**, a następnie **Create policy**. 

![Dodanie reguły uprawnień](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-3.png)
<span class="img-legend">Dodanie reguły uprawnień</span>

W przypadku edytora proponuję przejść na wersję JSON. Ułatwi to konfigurację uprawnień. Poniższe uprawnienia dają możliwość wykonywania wszystkich operacji na klastrze Kafki. Pamiętaj tylko, żeby zmienić region, **Account-ID** oraz **ClusterName**.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:*"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:cluster/ClusterName/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:*"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:topic/ClusterName/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:*"
            ],
            "Resource": [
                "arn:aws:kafka:region:Account-ID:group/ClusterName/*"
            ]
        }
    ]
}
```

Następnie utworzoną powyżej regułę musimy podpiąć pod rolę IAM. W tym celu z menu bocznego wybieramy **Roles**, a następnie **Create role**. Jako **Use case** wybieramy **EC2** i na kolejnym ekranie wybieramy utworzoną wcześniej regułę uprawnień. Klikamy **Next**, nadajemy naszej roli nazwę i tworzymy rolę za pomocą **Create role**.

Po powyższej konfiguracji mamy dwie możliwości. Pierwsza z nich polega na uruchomieniu Kouncil w infrastrukturze AWS, podpięciu pod niego Kafka Amazon MSK i ustawieniu publicznego dostępu do Kouncil. Druga opcja, naszym zdaniem mniej bezpieczna, to ustawienie publicznego dostępu do Kafki uruchomionej na AWS MSK.

## Opcja 1 - uruchomienie Kouncil w infrastrukturze AWS
Aby uruchomić Kouncil w infrastrukturze AWS będziemy potrzebowali instancji Amazon EC2. Możemy ją stworzyć [tutaj](https://console.aws.amazon.com/ec2/), klikając w **Launch instance**. Nadajemy nazwę. Dla Amazon Machine Image (AMI) możemy pozostawić domyślną wartość. Jako typ instancji wybieramy **t2.micro**. Następnie, jeśli nie posiadamy, tworzymy nową parę kluczy logowania i je pobieramy. Po rozwinięciu **Advanced Details** wybieramy w **IAM instances profile** utworzoną wcześniej rolę IAM. Następnie klikamy **Launch instance**.

### Konfiguracja grupy bezpieczeństwa
Aby Kouncil był dostępny z zewnątrz, musimy skonfigurować grupę bezpieczeństwa, a dokładniej pozwolić na dostęp do portu, na którym Kouncil zostanie uruchomiony. W tym celu wybieramy z listy naszą instancję i przechodzimy do zakładki **Security**.

![Szczegóły bezpieczeństwa naszej instancji](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-4.png)
<span class="img-legend">Szczegóły bezpieczeństwa naszej instancji</span>

Następnie przechodzimy do grupy bezpieczeństwa podpiętej pod naszą instancję. W szczegółach grupy klikamy **Edit inbound rules**, aby dodać nową regułę.
Klikamy **Add rule** i wybieramy typ **HTTP**, dla którego zostanie ustawiony domyślny port 80. Jako **Source** wybieramy **Anywhere-IPv4**. Przy takiej konfiguracji, do Kouncil będzie można się dostać z dowolnego adresu IP. Jeśli chcemy, możemy ograniczyć dostęp podając zakres lub konkretny adres IP. 

![Dodanie reguły umożliwiającej połączenie się z Kouncil](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-5.png)
<span class="img-legend">Dodanie reguły umożliwiającej połączenie się z Kouncil</span>

Musimy jeszcze zmodyfikować grupę bezpieczeństwa, z której korzysta nasz klaster Kafki. W tym celu przechodzimy najpierw do szczegółów klastra, a następnie do podpiętej pod niego grupy bezpieczeństwa. Klikamy **Edit inbound rules**.

Dalej klikamy **Add rule**. Tym razem wybieramy opcję **Custom TCP**. Jako port podajemy port, na którym dostępni są brokerzy, domyślnie jest to 9098. W **Source** wybieramy **Custom** i wskazujemy grupę bezpieczeństwa wykorzystywaną przez instancję, na której zostanie uruchomiony Kouncil. Na koniec klikamy **Save rules**.

![Dodanie reguły umożliwiającej komunikację z brokerami Kafki](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-6.png)
<span class="img-legend">Dodanie reguły umożliwiającej komunikację z brokerami Kafki</span>

### Uruchomienie instancji
Aby uruchomić instancję wracamy do listy instancji, wybieramy ją z listy i klikamy **Connect**. Na następnym ekranie zostawiamy domyślne wartości i wybieramy **Connect**.

### Utworzenie konfiguracji Kouncil
Musimy utworzyć plik zawierający konfigurację Kouncil, w którym podamy namiary na brokerów klastra Kafki, których uruchomiliśmy na początku. Namiary na brokerów naszej Kafki znajdziemy w szczegółach naszego klastra klikając w przycisk **View client information**.

![Adresy prywatne brokerów Kafki (dostępne w ramach infrastruktury AWS)](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-7.png)
<span class="img-legend">Adresy prywatne brokerów Kafki (dostępne w ramach infrastruktury AWS)</span>

Zawartość pliku konfiguracyjnego Kouncil powinna wyglądać następująco:
```yaml
kouncil:
  clusters:
    - name: aws-msk-cluster
      brokers:
        - host: b-1.kafkacluster.a4x88r.c25.kafka.us-east-1.amazonaws.com
          port: 9098
          saslMechanism: "AWS_MSK_IAM"
          saslProtocol: "SASL_SSL"
          saslJassConfig: software.amazon.msk.auth.iam.IAMLoginModule required;
          saslCallbackHandler: "software.amazon.msk.auth.iam.IAMClientCallbackHandler"
```

### Wystartowanie Kouncil
Kiedy już utworzyliśmy plik konfiguracyjny, pozostaje nam tylko zainstalować Dockera i będziemy mogli uruchomić aplikację.

W pierwszej kolejności odświeżmy zainstalowane paczki  i cache paczek na naszej instancji poleceniem:
```bash
sudo yum update -y
```

Aby zainstalować Dockera musimy wykonać poniższe polecenie:
```bash
sudo yum install docker
```

Kiedy Docker zostanie zainstalowany, uruchamiamy usługę Dockera poleceniem:
```bash
sudo service docker start
```

Na koniec dodajmy jeszcze naszego użytkownika (w moim przypadku ec2-user) do grupy Dockera, abyśmy mogli wykonywać polecenia Dockera bez używania **sudo**.
```bash
sudo usermod -a -G docker ec2-user.
```

Następnie musimy wylogować się z instancji i zalogować się do niej ponownie.
Poprawność instalacji Dockera możemy sprawdzić poleceniem docker info, które wylistuje nam konfigurację uruchomionego Dockera.

Aby uruchomić Kouncil musimy wykonać poniższe polecenie:
```bash
docker run -m 1G -d -p 80:8080 --restart always -v /home/ec2-user/:/config/ --name kouncil consdata/kouncil:latest_snapshot
```
Teraz pozostaje nam tylko sprawdzić publiczny DNS naszej instancji (**Public IPv4 DNS**). 

![Szczegóły instancji, na których możemy sprawdzić publiczny adres DNS](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-8.png)
<span class="img-legend">Szczegóły instancji, na których możemy sprawdzić publiczny adres DNS</span>

Otwieramy go w nowym oknie przeglądarki. Powinniśmy zobaczyć stronę logowania do Kouncil.

![Strona logowania w aplikacji Kouncil](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-9.png)
<span class="img-legend">Strona logowania w aplikacji Kouncil</span>

Po zalogowaniu na użytkownika o uprawnieniach administratora zobaczymy listę dostępnych brokerów Kafki.

![Dostępni brokerzy na uruchomionej Kafce](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-10.png)
<span class="img-legend">Dostępni brokerzy na uruchomionej Kafce</span>

## Opcja 2 - Wystawienie Kafki “na świat”
Aby nasz klaster był dostępny spoza infrastruktury AWS, musimy spełnić kilka warunków.

Po pierwsze włączamy publiczny dostęp do klastra. W tym celu należy przejść do ustawień sieci (**Networking settings**), rozwinąć opcje dostępne przy **Edit** i wybrać **Edit public access**. Dalej pozostaje tylko zaznaczyć **Turn on** i zapisać zmiany.

![Ustawienia sieci klastra](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-11.png)
<span class="img-legend">Ustawienia sieci klastra</span>

![Włączenie publicznego dostępu do klastra](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-12.png)
<span class="img-legend">Włączenie publicznego dostępu do klastra</span>

Kolejną ważną kwestią jest sprawdzenie konfiguracji dostępu do naszego klastra spoza infrastruktury AWS. Wszystkie podsieci, z których korzysta klaster muszą być publiczne, tzn. muszą mieć zdefiniowaną w tablicy routingu bramę internetową z dostępem zewnętrznym.

![Podsieć ze zdefiniowaną w tablicy routingu bramą internetową z dostępem zewnętrznym](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-13.png)
<span class="img-legend">Podsieć ze zdefiniowaną w tablicy routingu bramą internetową z dostępem zewnętrznym</span>

Ponadto, grupa bezpieczeństwa, z której korzysta nasz klaster musi pozwalać na ruch **TCP** z adresu IP, z którego będziemy chcieli się podłączyć pod Kafkę. Grupy bezpieczeństwa, z których korzysta nasz klaster są wylistowane w jego ustawieniach sieciowych. W szczegółach naszej grupy bezpieczeństwa widzimy reguły, które są używane dla ruchu przychodzącego i wychodzącego.

Zedytujmy regułę ruchu przychodzącego (**Edit inbound rules**). Jako typ wybieramy **Custom TCP**, natomiast w polu **Port range** wpisujemy port, na którym będą dostępni brokerzy Kafki, 9198.

Dla źródła możemy wybierać spośród kilku opcji:
* Custom pozwala zdefiniować dokładne adresy lub zakresy adresów, które mają mieć dostęp do klastra
* Anywhere-IPv4 pozwala na dostęp do klastra każdemu z zewnątrz
* Anywhere-IPv6 pozwala na dostęp do klastra każdemu z zewnątrz
* My IP - pozwala na dostęp do klastra tylko z adresu IP maszyny, z której konfigurujemy klaster

![Edycja reguły ruchu przychodzącego do klastra](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-14.png)
<span class="img-legend">Edycja reguły ruchu przychodzącego do klastra</span>

Na koniec zapisujemy zmiany klikając w **Save rules**.

Ostatnim warunkiem jest utworzenie użytkownika, za pomocą którego będziemy mogli uwierzytelnić się przy połączeniu do Kafki. Przechodzimy na główną stronę IAM i z menu bocznego wybieramy **Users**, a następnie **Create user**. 

Przy tworzeniu użytkownika mamy kilka możliwości podpięcia wyżej zdefiniowanych uprawnień. Możemy to zrobić za pomocą grupy zdefiniowanej dla użytkowników, albo podpiąć uprawnienia bezpośrednio pod użytkownika. Po utworzeniu użytkownika należy wygenerować dla niego klucz dostępu. Proponuję go sobie zapisać, ponieważ będziemy go później potrzebować przy podłączeniu się do Amazon MSK z Kouncil.

![Utworzony użytkownik z wygenerowanym kluczem dostępu](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-15.png)
<span class="img-legend">Utworzony użytkownik z wygenerowanym kluczem dostępu</span>

### Podpięcie pod Kouncil
Samo podpięcie pod Kouncil sprowadza się do podania publicznego adresu brokera w konfiguracji klastra Kafki oraz pozostałych parametrów połączenia. Przykładową konfigurację można znaleźć tutaj [Advanced config - Amazon MSK Kafka cluster](https://docs.kouncil.io/getting-started/deployment#advanced-config-amazon-msk-kafka-cluster).

![Publiczne adresy brokerów Kafki](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-16.png)
<span class="img-legend">Publiczne adresy brokerów Kafki</span>


Zapisany wcześniej klucz dostępu możemy wykorzystać jako zmienne środowiskowe. W przypadku uruchomienia Kouncil przy użyciu Dockera możemy je podać przy pomocy flagi **-e**:
* -e AWS_SECRET_ACCESS_KEY=<secret_access_key>
* -e AWS_ACCESS_KEY_ID=<access_key_id>

Po zalogowaniu się do Kouncil powinniśmy zobaczyć listę topiców, którą mamy utworzoną na wybranym brokerze Kafki.

![Lista brokerów z podpiętej Kafki Amazon MSK](/assets/img/posts/2024-04-24-kafka-aws-msk/kafka-aws-msk-17.png)
<span class="img-legend">Lista brokerów z podpiętej Kafki Amazon MSK</span>

## Podsumowanie
W powyższym wpisie poznaliśmy dwa sposoby, dzięki którym możemy podpiąć Kouncil pod klaster Kafki postawiony na Amazon Managed Streaming for Apache Kafka (Amazon MSK).

