# gRPC: Moderní způsob komunikace mezi mikroslužbami

**gRPC (gRPC Remote Procedure Call)** je vysoce výkonný, open-source framework vyvinutý společností Google. Slouží pro efektivní, rychlou a typově bezpečnou komunikaci mezi distribuovanými systémy. Je postavený na protokolu **HTTP/2** a využívá **Protocol Buffers (Protobuf)** jako jazyk pro definici rozhraní (IDL) a serializaci dat.

gRPC nachází své uplatnění především v architekturách založených na **mikroslužbách**, kde je klíčová rychlost, nízká latence, škálovatelnost a interoperabilita mezi různými technologiemi.

---

## Výhody použití gRPC oproti REST API

Na rozdíl od klasických REST API, které standardně přenášejí textový formát JSON přes protokol HTTP/1.1, přináší gRPC do backendové komunikace zásadní evoluci:

* **Vysoký výkon a nízká latence:** Díky binární serializaci pomocí Protobuf jsou přenášená data výrazně menší než JSON. Protokol HTTP/2 navíc umožňuje plný multiplexing (posílání více požadavků současně přes jedno TCP spojení).
* **Plně typovaná komunikace (API jako kontrakt):** Kontrakty jsou striktně definovány v `.proto` souborech. Klient i server se musí tímto kontraktem řídit, což eliminuje chyby způsobené nekonzistentními datovými strukturami.
* **Nativní podpora pro streaming:** gRPC podporuje 4 typy komunikačních modelů:
* *Unary* (klasický požadavek/odpověď)
* *Server-streaming* (jeden požadavek, proud odpovědí)
* *Client-streaming* (proud požadavků, jedna odpověď)
* *Bidirectional streaming* (obousměrný proud dat)


* **Podpora pro více programovacích jazyků:** Z jednoho `.proto` souboru lze automaticky vygenerovat klientské kód (Stub) a serverovou kostru pro C++, Javu, Python, Go, Node.js, C#, Rust a další.

---

## Srovnání: gRPC vs. REST API

| Vlastnost | gRPC | REST API |
| --- | --- | --- |
| **Protokol** | HTTP/2 (striktně vyžadováno) | HTTP/1.1 nebo HTTP/2 |
| **Datový formát** | Protobuf (binární, nečitelný textově) | JSON, XML (textový, lidsky čitelný) |
| **Typovost** | Silná, definovaná kontraktem (`.proto`) | Volná (závislá na dokumentaci / OpenAPI) |
| **Komunikační model** | Volání procedur (RPC) | Manipulace se zdroji (CRUD přes HTTP metody) |
| **Streaming** | Client, Server, Obousměrný | Pouze Server-Sent Events (SSE) nebo WebSockets |
| **Generování kódu** | Integrované a nativní | Vyžaduje nástroje třetích stran (Swagger) |

---

## Jak funguje gRPC v praxi

Komunikace probíhá na základě volání vzdálených procedur. Pro vývojáře to vypadá, jako by volal lokální funkci v kódu, ale na pozadí se provede síťové volání na jinou mikroslužbu.

### 1. Definice rozhraní v `.proto` souboru

Vývojář nejprve definuje službu a strukturu zpráv:

```protobuf
syntax = "proto3";

package greeter;

// Definice gRPC služby
service Greeter {
  // Unary RPC: Klient pošle jeden požadavek a dostane jednu odpověď
  rpc SayHello (HelloRequest) returns (HelloReply);
}

// Struktura požadavku
message HelloRequest {
  string name = 1;
  int32 age = 2;
}

// Struktura odpovědi
message HelloReply {
  string message = 1;
}

```

### 2. Generování kódu a implementace

Následně se pomocí kompilátoru `protoc` automaticky vygenerují knihovny pro požadovaný jazyk. To výrazně urychluje vývoj, protože odpadá ruční psaní validací a parsování JSONu.

---

## Použití v mikroslužbách a cloudu

gRPC je ideální volbou pro **interní komunikaci (East-West traffic)** v distribuovaných systémech:

> **Architektonická poznámka:** Zatímco pro komunikaci s frontendem (prohlížeč, mobilní aplikace) se často kvůli kompatibilitě stále volí REST API nebo GraphQL, pro komunikaci mezi desítkami vnitřních mikroslužeb v cloudu je gRPC standardem.

V prostředích jako **Kubernetes** se gRPC prosazuje díky nativní podpoře pokročilého load balancingu, distribuovaného trasování (tracing) a health checků. Architektury typu Service Mesh (např. **Istio** nebo **Linkerd**) dokážou s gRPC provozem efektivně pracovat přímo na síťové vrstvě.

---

## Bezpečnost a autentizace

Bezpečnost je v gRPC navržena jako prvořadá součást frameworku:

* **Šifrování:** gRPC striktně doporučuje (a v mnoha implementacích vyžaduje) použití **TLS** nebo **mTLS** (mutual TLS) pro zabezpečení přenosu dat mezi službami.
* **Autentizace:** Podporuje mechanismus *Metadata*, přes který lze bezpečně přenášet **JWT (JSON Web Tokens)** nebo **OAuth2** tokeny pro ověření identity volající služby.

---

## Nevýhody a omezení gRPC

Přestože gRPC nabízí špičkový výkon, přináší i specifické výzvy:

* **Horší lidská čitelnost:** Binární formát Protobuf není bez speciálních nástrojů (např. *gRPCurl*) čitelný. Nemůžete se jednoduše podívat do síťového tabu v prohlížeči na surová data jako u JSONu.
* **Omezená podpora v prohlížečích:** Webové prohlížeče v současnosti neumožňují dostatečnou kontrolu nad HTTP/2 rámcemi, aby mohly gRPC volat přímo. Pro webové klienty je nutné použít proxy vrstvu **gRPC-Web**.
* **Nutnost správy kontraktů:** Jakákoli změna v `.proto` souborech vyžaduje přegenerování kódu na straně klienta i serveru. Je nutné striktně dodržovat pravidla zpětné kompatibility (např. neměnit indexy polí v Protobuf zprávách).

---

## Závěr

gRPC představuje moderní, vysoce efektivní a robustní způsob komunikace v distribuovaných systémech. Pokud stavíte komplexní systém založený na mikroslužbách, kde rozhodují milisekundy latence a propustnost sítě, je gRPC technologií, kterou byste rozhodně měli do své architektury zařadit.
