# gRPC: Moderní a efektivní komunikace mezi mikroslužbami

**gRPC (gRPC Remote Procedure Call)** je vysoce výkonný, open-source framework vyvinutý společností Google a uvolněný v březnu 2015. Za více než deset let své existence se stal časem prověřenou technologií, kterou pro svou interní komunikaci adoptovali technologičtí giganti jako **Netflix** nebo **Spotify**.

Kolem gRPC vznikla masivní komunita a dnes se jedná o standard s podporou prakticky všech myslitelných platforem. gRPC služby jsou orientované **procedurálně**, což z nich dělá ideální volbu pro vývojáře, kterým nevyhovuje architektonický styl REST API. V prostředí **.NET** se gRPC ustálilo jako přímý myšlenkový nástupce dříve populárního, ale již zastaralého frameworku **WCF (Windows Communication Foundation)**.

---

## Architektura a efektivní komunikace

Maximální výkonnost gRPC je zajištěna synergickým spojením dvou klíčových technologií: **protokolu HTTP/2** a **binární serializace Protobuf**.

### 1. Protokol HTTP/2 jako základní pilíř

HTTP/2 přineslo revoluci v síťové komunikaci, ze které gRPC plně těží:

* **Framing (Rámování):** Komunikace je rozdělena na malé, samostatné binární rámce, což eliminuje textový režim starších protokolů.
* **Multiplexing:** Na jednom jediném TCP spojení lze otevřít stovky paralelních streamů mezi klientem a serverem. Data tak mohou proudit kontinuálně oběma směry bez nutnosti režie na neustálé otevírání nových spojení.

### 2. Multiplatformní a úsporný Protobuf

**Protocol Buffers (Protobuf)** představují jazykově neutrální mechanismus pro binární serializaci dat. Namísto formátu JSON nebo XML se data komprimují do minimálního binárního streamu. Bohatá nabídka nástrojů umožňuje ze společné specifikace generovat klientské knihovny i dokumentaci pro libovolný programovací jazyk.

---

## Srovnání: gRPC vs. REST API

| Vlastnost | gRPC | REST API |
| --- | --- | --- |
| **Protokol** | HTTP/2 (striktně vyžadováno) | HTTP/1.1 nebo HTTP/2 |
| **Datový formát** | Protobuf (binární, úsporný) | JSON, XML (textový, lidsky čitelný) |
| **Komunikační model** | Unary, Client/Server/Obousměrný stream | Request-Response (případně SSE/WebSockets) |

---

## Podpora komunikačních modelů (Streaming)

Díky plně duplexnímu streamování v HTTP/2 podporuje gRPC čtyři základní scénáře, které lze efektivně využít pro synchronní komunikaci v mikroslužbách, dávkové operace i přenosy v reálném čase:

* **Unary (Požadavek/Odpověď):** Klasický model, kdy klient pošle jeden požadavek a čeká na jednu odpověď.
* **Server-streaming:** Klient pošle jeden požadavek a server kontinuálně posílá proud dat (např. continuous feed, sledování logů).
* **Client-streaming:** Klient posílá proud dat na server, který po zpracování vrátí jednu finální odpověď (např. nahrávání velkého souboru po částech).
* **Bidirectional (Obousměrné) streaming:** Obě strany posílají data nezávisle na sobě v reálném čase. Ideální pro **kontinuální čtení dat ze senzorů (IoT), notifikační huby nebo chatovací aplikace**.

---

## Návrh gRPC služeb a podpora v .NET

Podpora gRPC sahá v ekosystému Microsoftu až do tradičního .NET Frameworku, avšak skutečně nativní, snadné a vysoce optimalizované použití přinesl až **.NET Core 3.1** a novější verze (.NET 6/8/9). Skvělou zprávou pro enterprise vývojáře je, že gRPC se rozvíjí stabilně a pomalu – není třeba se obávat neustálých breaking changes a zpětně nekompatibilních změn.

V .NET prostředí existují dva hlavní přístupy k návrhu:

### A. Contract-First (Přístup přes `.proto` soubory)

Standardní cesta, kdy se nejprve definuje rozhraní v neutrálním jazyce Protobuf:

```protobuf
syntax = "proto3";
package greeter;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}

```

### B. Code-First (Přístup bez `.proto` souborů)

Tento přístup vyhovuje zejména týmům přecházejícím z **WCF** a firmám, které vyvíjejí čistě v rámci .NET platformy. Služby a kontrakty se navrhují výhradně pomocí C# rozhraní a datových tříd. Kontrakty lze následně mezi klientskými a serverovými projekty sdílet extrémně pohodlně formou vnitrofiremních **NuGet balíčků**, což celý vývoj výrazně zjednodušuje.

---

## Reflection a nástroje pro vývoj

Pro účely testování a automatického objevování endpointů (Discovery Service) disponuje gRPC mechanismem **gRPC Reflection**.

* **Podobnost se Swaggerem:** Reflection funguje velmi podobně jako Swashbuckle Swagger u REST API. Dokáže za běhu aplikace zpětně analyzovat endpointy i kontrakty a vystavit je okolnímu světu.
* **Univerzálnost:** Tento mechanismus funguje jak v režimu s `.proto` soubory, tak v případě přístupu Code-First.

Díky Reflection lze gRPC služby snadno testovat, zkoumat z příkazové řádky (**CLI**) nebo pomocí moderních grafických nástrojů (např. *Postman*, *Insomnia* nebo *gRPCUI*), a to včetně komplexního testování asynchronních streamů přímo na lokálním počítači.

---

## Web a gRPC: Propojení s klientským UI

Původně bylo gRPC určeno výhradně pro komunikaci typu *backend-to-backend* (tzv. East-West traffic). Dnes už to však neplatí a gRPC proniká i do scénářů *klient-server*.

Standardní webové prohlížeče sice neumí kvůli omezením v API přirozeně manipulovat s HTTP/2 rámci, tento problém je však vyřešen pomocí **gRPC-Web proxy**. Sám Microsoft je autorem proxy implementované přímo v podobě **nativního middlewaru** v .NETu, která zajišťuje zpětnou kompatibilitu pro **HTTP/1.1**.

### Výhody pro Blazor WebAssembly

Tuto integraci ocení zejména vývojáři stavějící **Blazor WebAssembly** aplikace. Klientské UI v prohlížeči nemusí komunikovat přes klasické REST rozhraní, ale komunikuje přímo s procedurálně orientovanou gRPC službou. Výsledkem je:

1. Drastické zrychlení komunikace mezi UI a backendem.
2. Možnost provádět vysoce efektivní dávkové operace v reálném čase.
3. Sdílení datových modelů a typová bezpečnost napříč celou aplikací (od databáze až po prohlížeč).

---

## Nevýhody a omezení gRPC

I přes špičkové vlastnosti má gRPC specifická úskalí:

* **Lidská nečitelnost:** Surová binární data proudící po síti nelze bez speciálních nástrojů (např. *gRPCurl*) číst v síťové záložce prohlížeče, což ztěžuje rychlý debugging.
* **Režie se správou kontraktů:** Každá strukturální změna v rozhraní vyžaduje distribuci nových kontraktů (přegenerování kódu nebo aktualizaci NuGet balíčku) a striktní dodržování pravidel zpětné kompatibility.
* **Nevhodné pro veřejná API:** Pro otevřená API určená široké veřejnosti nebo třetím stranám zůstává kvůli snadné integraci a univerzálnosti standardem REST API (JSON).

