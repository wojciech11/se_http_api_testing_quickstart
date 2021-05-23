## Praca z API HTTP w pigułce

### Warto wiedzieć:

- Kody HTTP, minimum znać, patrz na [MDN web docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status):
  - Successful responses - 2xx: 200, 201
  - Redirects - 3xx: 301, 302
  - Client errors - 4xx: 400, 401, 403, 404, 405, 422, 429
  - Server errors - 5xx: 500, 502, 503 i 504

- Metody HTTP, podstawowe (więcej na [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)):
  - POST
  - GET
  - PUT
  - DELETE

- Co to jest REST / REST API? Jakie zasady powinno spełniać API? Na pewno bym zaczął od:
  - poleganie na HTTP w sposobie działania
  - bezstanowość - każdy request zawiera całą informację o stanie.

- [Dodatkowe] Przeczytaj co to jest GraphQL.

### Praca z API

- Curl & [httpie](https://httpie.io/) - szybkie sprawdzenie lub proste skrypty w bashu (czasami można się pokusić o proste checki z pomocą [BATS](https://github.com/sstephenson/bats)):

  ```bash
  $ curl -X POST -H "Content-Type: application/json" \
       -d '{"name":"natalia"}' https://httpbin.org/post

  $ http POST  https://httpbin.org/post "name"="natalia"
  ```

  ```bash
  # szybki smoke test

  $ curl -X GET https://httpbin.org/status/404 --fail
  # --fail, dzięki temu zwraca curl błąd
  $ echo $?

  # mamy teraz endpoint oczekujący get, a wysyłamy POST:
  $ curl -X POST https://httpbin.org/get --fail

  # porównaj z:
  $ curl -X GET https://httpbin.org/get --fail
  $ curl -X GET https://httpbin.org/status/404 --fail
  ```

  ```bash
  $ export API_ENDPOINT=127.0.0.1:5000
  $ curl -s -o /dev/null -w "%{http_code}" --fail ${API_ENDPOINT}
  ```

  A jak w skrypcie (zauważ z httpie byłoby zdecydowanie prościej :) ):

  ```bash
  # chcemy wyciąnąć imie z odpowiedzi na naszego POST
  $ curl -s --fail -X POST -H "Content-Type: application/json" \
      -d '{"name":"natalia"}' https://httpbin.org/post

  $ curl -s --fail -X POST -H "Content-Type: application/json" \
      -d '{"name":"natalia"}' https://httpbin.org/post \
      | jq '.json | .name'

  $ curl -s --fail -X POST -H "Content-Type: application/json" \
      -d '{"name":"natalia"}' https://httpbin.org/post \
      | jq '.json | .name' | tr -d '"'
  ```

  Teraz skrypt, `show_name.bash` (skąd te flagi? Przeczytaj [better bash in 15 minutes](http://robertmuth.blogspot.com/2012/08/better-bash-scripting-in-15-minutes.html)):

  ```bash
  #!/bin/bash
  set -o nounset
  set -o errexit
  set -o pipefail

  GENERATED_NAME=$(curl -s --fail -X POST -H "Content-Type: application/json" -d '{"name":"natalia"}' https://httpbin.org/post | jq '.json | .name' | tr -d '"  ')

  if [[ $GENERATED_NAME != "natalia" ]]; then
      echo "bledna wartosc: ${GENERATED_NAME}"
      exit 1
  else
    echo "OK"
  fi
  ```

- Praca z API, przygotowując skrypt wykorzystujący API jeśli popełniasz błędy zazwyczaj natkniesz się na następujące błędy:

  - [400](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/400)
  - [422](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/422) - serwer przyjął twoje żądanie ale serwer nie jest w stanie pracować z danymi, które właśnie przesłałaś
  - [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404)
  - [403](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/403)
  - [405](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/405) - method not allowed, próbujemy POST na endpoint-cie (ścieżce API) zamiast GET

  Zauważ, czasami możesz mieć problemy z rate-limiting, w zależności od API dostaniesz różne odpowiedz, patrz [rate limit Github z 403](https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting) i porównaj do [Stripe rate limiting posługującym się 429](https://stripe.com/docs/rate-limits)

- biblioteka [requests](https://docs.python-requests.org/en/master/user/quickstart/) (bread & butter):

  ```python
  import requests

  srv = "https://services.odata.org/v4/TripPinServiceRW/People('russellwhyte')"
  result = requests.get(srv)

  # moglibyśmy przerwać żądanie
  # result.raise_for_status()


  print("Status {0}".format(result.status_code))
  print("Result: {0}".format(result.json()))
  ```

- unittesty flask - patrz twoje testy w test_views.py na podstawie [test_views w projekcie do ćwiczeń](https://github.com/wojciech11/se_hello_printer_app/blob/master/test/test_views.py)

  - sprawdź czy `status_code` jest jaki oczekujesz
  - zawsze parsujemy wynik za pomocą odpowiedniej biblioteki, np., json czy [lxml](https://lxml.de/tutorial.html).
  - jeśli JSON to kapitalnie sprawdza się również biblioteka jmespath (jak XPATH ale dla JSONA!), np:

    ```python
    import jmespath
    import requests

    r = requests.get("http://127.0.0.1:5000?output=json")
    rj = r.json()

    expected = "Apolonia"
    actual = jmespath.search("imie", rj)
    if actual != expected:
        print("aaa!")
    ```

- `requests` i [behave](https://behave.readthedocs.io/en/stable/) lub Robot - dla czytelności naszych intencji możemy wykorzystać BDD do lepszego opisania naszych testów, patrz instrukcja Python część 2 z BDD.

- postman - manualne testy, większość firm w których pracowałem wykorzystywały to do manualnego testowania i współdzielenie know-how jak korzystać z danych endpointów. Warto zacząć skupić się na [building requests](https://learning.postman.com/docs/sending-requests/requests/), można szybko również spojrzeć na pozostałe tematy w [ich tutorialu](https://learning.postman.com/docs/getting-started/introduction/).

- zewnętrzne serwisy mogą być wolne lub możemy mieć problemy z połączeniem z nimi, więc warto zawsze określać timeouty, gdy wykonujemy zewnętrze requesty (patrz [requests timeouts](https://docs.python-requests.org/en/master/user/quickstart/#timeouts)):

  ```python
  requests.get('https://github.com/', timeout=0.001)
  ```

- Pamiętaj, że mamy wiele rodzajów API oraz metod RFC (remote function call) dla wywołania zdalnej funkcjonalności zdania przez, np., protokół HTTP lub inny.

### Dodatkowe

- Zapoznaj się z zadaniami dotyczącymi perf testing za pomocą [locust.io](https://locust.io/): https://github.com/wojciech11/se_perf_testing_basics
