# Fronty zpráv s RabbitMQ

**Předmět:** Analýza IS – NoSQL / Big Data  
**Autor:** Kryštof Pavlů, Daniel Borovička 

---

**Fronty zpráv**

Jak oddělit služby, zvládnout špičky a neztratit rozpracovanou úlohu

`Producent → broker → konzument`


Dnes budu mluvit o frontách zpráv a systému RabbitMQ. Hlavní princip je jednoduchý: aplikace, která vytvoří nějaký úkol, nemusí čekat, až ho jiná aplikace dokončí. Úkol předá prostředníkovi, kterému říkáme broker, a může pokračovat dál. RabbitMQ se postará o uložení a předání úkolu správnému příjemci.

Příkladem může být e-shop. Po vytvoření objednávky je potřeba poslat e-mail, rezervovat zboží nebo zapsat údaje do analytiky. Tyto činnosti nemusí proběhnout všechny ve stejné sekundě.

**More u know:** RabbitMQ je prostředník mezi aplikací, která úkol vytvoří, a aplikací, která ho zpracuje.

---

## Jeden pomalý krok může zablokovat celou objednávku

**Synchronní řetězec:**

`Web → Platba → Sklad → E-mail → Výsledek`

- Web přijme objednávku.
- Platební služba čeká na bránu.
- Sklad rezervuje zboží.
- E-mailová služba posílá potvrzení.
- Uživatel stále čeká.

Pokud e-mail nebo sklad vypadne, chyba se může přenést až k uživateli, i když samotná objednávka už vznikla.

Nejdřív si ukážeme problém. Uživatel odešle objednávku a web začne postupně volat další služby. Čeká na platbu, sklad a nakonec na odeslání e-mailu. Tomu říkáme synchronní komunikace, protože jedna část čeká na odpověď druhé.

Když je například e-mailová služba pomalá, čeká kvůli ní celý požadavek. Pokud úplně spadne, může uživatel dostat chybu, i když objednávka už byla správně vytvořena. Odeslání potvrzovacího e-mailu přitom může klidně proběhnout o několik sekund později.

**Jednoduché přirovnání:** Jeden člověk nosí dokument postupně přes několik kanceláří a v každé čeká na razítko.

---

## Fronta umožní bezpečně předat úkol

| Bez fronty | S frontou |
|---|---|
| Web volá e-mailovou službu přímo. | Web předá zprávu brokeru. |
| Obě služby musí současně fungovat. | Konzument může pracovat vlastním tempem. |
| Web čeká na dokončení. | Web nemusí čekat na dokončení. |
| Špička okamžitě zatíží příjemce. | Zprávy se dočasně seřadí ve frontě. |
| Chyba příjemce se vrátí webu. | Nedokončenou zprávu lze doručit znovu. |

**Asynchronní neznamená okamžité. Znamená, že odesílatel nemusí čekat na výsledek.**

Fronta vloží mezi web a e-mailovou službu prostředníka. Web už neposílá úkol přímo e-mailové službě. Vytvoří zprávu, například „pošli potvrzení objednávky“, a předá ji RabbitMQ.

RabbitMQ zprávu podrží. E-mailová služba si ji vezme, až bude mít kapacitu. Web díky tomu nemusí čekat na samotné odeslání e-mailu. Stačí mu vědět, že úkol byl přijat.

Fronta pomáhá také při krátkodobé špičce. Když přijde tisíc objednávek najednou, e-mailová služba je nemusí zpracovat ve stejné sekundě. Zprávy čekají ve frontě a zpracují se postupně.

**Pozor:** Uživatel rychle dostane informaci, že úkol byl přijat. To ale ještě nemusí znamenat, že byl dokončen.

---

## Jak zpráva prochází RabbitMQ


`PRODUCENT → EXCHANGE → FRONTA → KONZUMENT → ACK`

- **Producent** vytvoří zprávu.
- **Exchange** rozhodne, kam zpráva půjde.
- **Fronta** zprávu dočasně drží.
- **Konzument** zprávu převezme a zpracuje.
- **ACK** potvrzuje úspěšné dokončení.

Jednu zprávu z jedné pracovní fronty zpracuje jeden konzument.


Producent je aplikace, která zprávu vytváří. Může to být například web e-shopu. Zprávu neodesílá přímo konzumentovi, ale publikuje ji do exchange.

Exchange si můžeme představit jako poštovní třídírnu. Podle pravidel a hodnoty zvané routing key rozhodne, do které fronty zpráva patří. Fronta je jako přihrádka, ve které zpráva čeká.

Konzument je pracovník nebo aplikace, která zprávu vezme a provede požadovanou práci. Po úspěšném dokončení odešle potvrzení ACK. Teprve potom broker ví, že zprávu může odstranit.

**Přirovnání k restauraci:** Číšník je producent, objednávkový lístek je zpráva, lišta s lístky je fronta, kuchař je konzument a označení „hotovo“ je ACK.

---

## Jedna objednávka může spustit více úloh

**Událost: objednávka byla vytvořena**

- Sklad rezervuje zboží.
- E-mailová služba pošle potvrzení.
- Analytika zapíše událost do reportingu.

Každá služba má vlastní frontu a může fungovat nezávisle.

Jedna událost může zajímat více služeb. Když vznikne objednávka, sklad musí rezervovat zboží, e-mailová služba musí poslat potvrzení a analytická služba chce událost zapsat do reportu.

Exchange může stejnou událost nasměrovat do několika samostatných front. Každá služba tak dostane vlastní kopii zprávy. Pokud analytika hodinu nefunguje, neměla by kvůli tomu přestat fungovat rezervace zboží ani posílání e-mailů. Po restartu si analytika čekající zprávy postupně zpracuje.

Tomuto způsobu distribuce se říká publish/subscribe. Jeden producent publikuje událost a více odběratelů o ni může mít zájem.

**Pozor:** Pokud mají tři různé služby dostat stejnou událost, obvykle potřebují tři samostatné fronty napojené na exchange.

---

## Minimální architektura

`Python producer → RabbitMQ broker → Python consumer`

- **Producer:** Python a knihovna `pika`; publikuje zprávu.
- **RabbitMQ:** broker; port `5672` pro AMQP a `15672` pro administrační rozhraní.
- **Consumer:** Python a knihovna `pika`; zprávu zpracuje a potvrdí.
- Vše může spouštět Docker Compose na společné síti.
- RabbitMQ může používat perzistentní volume.

Nejjednodušší praktická architektura má tři části. První je producent napsaný v Pythonu. Ten vytváří zprávy. Uprostřed běží RabbitMQ a na druhé straně je konzument, také v Pythonu.

Pythonové aplikace používají knihovnu pika, která umí komunikovat s RabbitMQ. Port 5672 slouží pro komunikaci aplikací pomocí protokolu AMQP. Port 15672 slouží pro webové administrační rozhraní RabbitMQ, pokud je zapnutý management plugin.

Docker Compose umožňuje všechny části spustit společně. Perzistentní volume uchovává data RabbitMQ mimo život jednoho kontejneru.

**Pozor:** Docker ani Python nejsou principem fronty. Jsou to pouze technologie použité v této konkrétní implementaci.

---

## Co dělá producent

```python
channel.queue_declare(
    queue="hello_queue",
    durable=True
)

channel.basic_publish(
    exchange="",
    routing_key="hello_queue",
    body=message_body,
    properties=pika.BasicProperties(
        delivery_mode=2
    )
)
```

- `durable=True` – definice fronty přežije restart brokeru.
- `delivery_mode=2` – zpráva je označena jako perzistentní.
- `routing_key="hello_queue"` – určuje cílovou frontu při použití výchozího exchange.

Producent nejprve deklaruje frontu s názvem hello_queue. Parametr durable nastavený na true říká, že definice této fronty má přežít restart RabbitMQ.

Potom producent publikuje zprávu. V jednoduchém příkladu používá výchozí exchange, proto je routing key přímo názvem cílové fronty. Delivery mode 2 označuje zprávu jako perzistentní, takže ji RabbitMQ ukládá odolněji vůči restartu.

Durable fronta a perzistentní zpráva nejsou stejná věc. Durable se týká existence fronty. Persistent se týká konkrétní zprávy. V produkčním systému se pro potvrzení, že broker zprávu skutečně převzal, používají také publisher confirms.

**Jednoduchá pomůcka:** Durable je trvalá poštovní schránka. Persistent je dopis uložený tak, aby lépe přežil restart. Publisher confirm je potvrzení pošty, že dopis přijala.

---

## Co dělá konzument


```python
channel.basic_qos(prefetch_count=1)

channel.basic_consume(
    queue="hello_queue",
    on_message_callback=callback,
    auto_ack=False
)

def callback(ch, method, props, body):
    process(body)
    ch.basic_ack(
        delivery_tag=method.delivery_tag
    )
```

- `auto_ack=False` – zpráva se nepotvrdí automaticky při doručení.
- `basic_ack(...)` – konzument zprávu potvrdí až po dokončení práce.
- `prefetch_count=1` – konzument dostane nejvýše jednu nepotvrzenou zprávu.

Konzument má automatické potvrzení vypnuté. To je důležité, protože samotné převzetí zprávy ještě neznamená, že práce byla dokončena.

Konzument nejprve zavolá funkci process a provede požadovanou práci. Až potom odešle basic_ack. Pokud by spadl před odesláním ACK, RabbitMQ ví, že zpráva nebyla potvrzena.

Prefetch jedna znamená, že konzument nedostane další nepotvrzenou zprávu, dokud nedokončí první. Když běží více konzumentů, práce se díky tomu rozděluje férověji.

**Přirovnání:** Kuchař dostane jeden objednávkový lístek. Dokud neřekne „hotovo“, vedoucí mu nedá další.

---

## Co se stane při pádu

| Konzument spadne před ACK | Konzument odešle ACK |
|---|---|
| Zpráva zůstane nepotvrzená. | Broker považuje zprávu za dokončenou. |
| Broker ji může vrátit do fronty. | Zpráva se odstraní z fronty. |
| Zpráva může být doručena znovu. | Znovu se běžně nedoručuje. |

**Důsledek:** Konzument musí být idempotentní.

Když konzument spadne před odesláním ACK, broker považuje úkol za nedokončený. Po ztrátě spojení může zprávu vrátit do fronty a doručit ji stejnému nebo jinému konzumentovi.

To znamená, že jedna zpráva může být zpracována vícekrát. Proto musí být konzument idempotentní. Idempotence znamená, že opakované zpracování stejné zprávy nevytvoří nežádoucí dvojí výsledek.

Příkladem je platba s identifikátorem PAY-55. Konzument platbu provede, ale spadne těsně před ACK. Zpráva přijde znovu. Konzument musí podle identifikátoru poznat, že platba PAY-55 už proběhla, a nesmí peníze strhnout podruhé.

**Zapamatuj si:** RabbitMQ běžně pomáhá zajistit doručení alespoň jednou. Duplicitní doručení je proto možné.

---

## Kdy RabbitMQ použít a kdy ne

| RabbitMQ se hodí | Je vhodné zvážit jiný přístup |
|---|---|
| Odesílání e-mailů a notifikací | Uživatel potřebuje okamžitou odpověď |
| Zpracování obrázků a dokumentů | Jednoduchá malá aplikace |
| Integrace mikroslužeb | Dlouhodobý event log a opakované čtení ve velkém |
| Vyrovnání krátkodobých špiček | Není vyřešen monitoring, retry a idempotence |
| Rozdělení práce mezi více workerů | Broker by přinesl více složitosti než užitku |

RabbitMQ je vhodný hlavně pro úlohy, které mohou proběhnout asynchronně. Typickým příkladem je odesílání e-mailů, generování dokumentů, zpracování obrázků nebo rozdělení práce mezi více pracovníků.

Není ale automaticky nejlepší pro každou komunikaci. Pokud uživatel potřebuje okamžitou odpověď, může být jednodušší přímé synchronní REST API. Pro dlouhodobé uchovávání proudu událostí a jejich opakované čtení se často zvažuje například Kafka.

Fronta přidává odolnost, ale také další složitost. Je potřeba provozovat broker, sledovat velikost front, nastavit retry pravidla a řešit duplicitní zprávy.

**Hlavní myšlenka:** RabbitMQ použijeme tehdy, když je pro nás užitečné oddělit vznik úkolu od jeho pozdějšího zpracování.

---

## Závěr

> **Fronta nekouzlí. Dává systému čas.**

### Tři hlavní přínosy

1. **Oddělení** – producent nemusí čekat na konzumenta.
2. **Odolnost** – nepotvrzený úkol lze doručit znovu.
3. **Škálování** – více konzumentů si může rozdělit práci.

**Kontrolní otázka:** Co se stane se zprávou, když konzument spadne před ACK?

Na závěr zopakuji tři hlavní přínosy. Fronta odděluje služby, takže producent nemusí čekat na konzumenta. Pomáhá s odolností, protože nepotvrzenou zprávu lze doručit znovu. A umožňuje škálování, protože více konzumentů si může rozdělit práci z jedné fronty.

RabbitMQ ale samo o sobě nezaručuje, že se všechno provede přesně jednou. Musíme správně používat potvrzení, perzistenci, retry a idempotenci.

Pokud konzument spadne před ACK, zpráva zůstane nepotvrzená a RabbitMQ ji může doručit znovu. To je nejdůležitější odpověď celé prezentace.


# Zdroje

- Kurzový repozitář: <https://github.com/TomasRacil/analyza-IS-NoSQL>
- RabbitMQ Work Queues: <https://www.rabbitmq.com/tutorials/tutorial-two-python>
- RabbitMQ Consumer Acknowledgements: <https://www.rabbitmq.com/docs/confirms>
- RabbitMQ Consumer Prefetch: <https://www.rabbitmq.com/docs/consumer-prefetch>
