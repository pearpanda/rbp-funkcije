# Uvod u funkcije u DB2

Svrha ovog dokumenta je da ukratko predstavi način kako se kreiraju
korisnički definisane funkcije u DB2.

## Skelet

Funkcije u DB2 SQL-u se prave naredbom `CREATE FUNCTION`. Celokupan
izgled naredbe izgleda nalik ovome:

```sql
CREATE OR REPLACE FUNCTION <ime_funkcije>(<lista_parametara>)
RETURNS <povratni tip>
LANGUAGE SQL
NOT DETERMINISTIC
READS SQL DATA
<naredba>
```

Parametri u listi parametara se navode razdvojeni zarezom, u obliku
`<ime> <tip>`, na primer `broj_indeksa INTEGER`.

Objasnimo dodatne aspekte skeleta:

- `LANGUAGE SQL`: DB2 podržava mogućnost pisanja procedura i funkcija
  u tipičnim programskim jezicima, poput Jave i C-a. Ovim naglašavamo
  da pišemo naredbe u *dijalektu* SQL jezika zvanom SQL PL, koji je
  fokus ovog dokumenta.
- `NOT DETERMINISTIC`: Može se navesti `DETERMINISTIC` ili `NOT
  DETERMINISTIC`. Ukoliko je funkcija `DETERMINISTIC`, onda za
  određene ulazne argumente uvek vraća istu vrednost (možemo je
  posmatrati kao *matematičku funkciju*). U našem slučaju, rezultat će
  nam verovatno zavisiti od pročitanih podataka, pa naša funkcija nije
  deterministična. Koristi se za optimizaciju upita.
- `READS SQL DATA`: Može biti `CONTAINS SQL`, `READS SQL DATA` i
  `WRITES SQL DATA`. Nisam 100% siguran tačno u čemu se razlikuju. Po mom
  razumevanju, ograničavaju koje naredbe su dozvoljene unutar
  funkcije, pa samim tim su bitne za kontrolu dozvola.

## Telo

Može biti jedna naredba, ili može biti niz naredbi u `BEGIN ... END`
bloku, ili `BEGIN ATOMIC ... END` bloku.

Ukoliko se vraća rezultat SQL upita direktno, mislim da je obavezno
`BEGIN ATOMIC ... END` ili da je jedina naredba (bez bloka) `RETURN`. Kao
kontrast, obe opcije onemogućavaju određene operacije sa promenljivama.

Lični savet: ukoliko vraćate ceo rezultat unutar jednog SQL upita
koristite oblik samo sa jednom naredbom i neka ta naredba bude oblika

```sql
RETURN
SELECT ...
FROM ...
```

### Uvod u SQL PL

Mi ćemo se opredeliti za `BEGIN ... END` varijantu. Naša funkcije će
otprilike izgledati ovako:

```sql
CREATE OR REPLACE FUNCTION ime(parametri)
RETURNS tip
LANGUAGE SQL
NOT DETERMINISTIC
READS SQL DATA
BEGIN
    -- Sekcija deklaracija
    -- Sekcija dodele
        -- Glavni upit
    -- Vraćanje vrednosti
END @
```

#### Deklaracije u SQL PL

DB2 nam omogućava da u našim funkcijama koristimo promenljive, i da ih
koristimo i u uslovima SQL upita. Kao primer, neka se traže svi studenti
čiji je prosek veći od proseka studenta sa nekim zadatim indeksom.
Tipičan način bi bio da taj prosek računamo preko podupita, ali
prirodnije je izračunati ga pre i smestiti u promenljivu, i onda
koristiti taj rezultat u glavnom upitu.

Deklaracija promenljive izgleda ovako:

```sql
DECLARE <ime> <tip>;
```

Lična preporuka: imena promenljivih započnite sa `v_`, da bi se
razlikovale od imena kolona u kasnijim upitima.

**Napomena**: I parametri funkcije su isto promenljive, pa se mogu
koristiti svuda gde i deklarisane promenljive.

**Napomene:**
-   Promenljive (koje nisu parametri) je potrebno deklarisati pre
    upotrebe.
-   Sekcija deklaracija nema nikakvu naznaku da je odvoji od ostatka
    koda. Možete koristiti praznu liniju da razdvojite sekcije.
-   Sve deklaracije, i ništa sem deklaracija ne ide u sekciju
    deklaracije. Ako koristite bilo koju naredbu pre neke `DECLARE`
    naredbe, dobićete grešku! *Setite se da su deklaracije u C-u uvek
    išle na početku bloka.*

#### Dodela vrednosti u SQL PL

Postoje dve naredbe za dodelu: jedna je `SET`, a druga je `SELECT INTO`.

```sql
SET <ime> = <vrednost>;
```

Tipična upotreba `SET` naredbe je za dodela konstante, ili nekog izraza
drugih promenljivih.

```sql
SELECT kolona
   INTO promenljiva
FROM ...
WHERE ...
...
```

`SELECT INTO` vam omogućava da koristite SQL na isti način kao što ste
navikli, da biste dodelili vrenost promenljivoj.

**Napomena:** Povratna vrednost za promenljive skalarnog tipa (što je
jedino nama od interesa) su obavezno **samo jedna kolona**, i upit **ne
sme da vrati više od jednog reda**!

Primer:

Pronaći prosek studenta čiji je indeks dat promenljivom `v_indeks`.
Rezultat smestiti u `v_prosek`.

```sql
SELECT AVG(1.0 * ocena)
    INTO v_prosek
FROM da.dosije AS d JOIN da.ispit AS i
     ON d.indeks = i.indeks
        AND status = 'o'
        AND ocena > 5
```

Glavni upit je poseban tip dodele: dodeljujete vrednost promenljivoj
koju ćete kasnije vratiti `RETURN` naredbom.

#### Vraćanje vrednosti

```sql
RETURN <vrednost>;
```

#### Grananje

Koristi se `IF ... ELSEIF .. ELSE ... END IF;` konstrukcija:

```sql
IF uslov THEN
    naredbe
ELSEIF uslov THEN
    naredbe
ELSE
    naredbe
END IF;
```

Primer:

```sql
IF v_broj_polozenih_predmeta = 0 THEN
   RETURN 0;
ELSE
   -- Racunaj prosek
   SELECT ...
      INTO v_prosek
   FROM ...
   ...;

   RETURN v_prosek;
END IF;
```

## Saveti za ispit

- Tipično će se tražiti da se vrati skalar koji je lista. To znači da
  će povratna vrednosti biti `VARCHAR(32672)`. (32672 je najveća
  moguća vrednost dužine `VARCHAR` niske). Podatak se može naći u SQL
  Reference.
- Da bi povratak liste funkcionisao kako treba, povratak treba da bude
  u obliku `LISTAGG(CAST(... AS VARCHAR(32672)), <separator>)`.
- Za generisanje liste će tipično biti neophodna pomoćna tabela. Data
  Studio ne prihvata korišćenje `SELECT` upita sa `WITH` delom, iako
  DB2 prihvata i funkcija radi normalno. Lični savet: upit za pomoćnu
  tabelu smestite u `FROM` klauzulu glavnog upita.

## Primer zadatka sa ispita

Tekst: 4. zadatak pod v), Januar 2019.

Rešenje:

```sql
CREATE OR REPLACE FUNCTION spisak(indeks INTEGER, br_predmeta INTEGER)
RETURNS VARCHAR(32672)
LANGUAGE SQL
READS SQL DATA
NOT DETERMINISTIC
BEGIN
    DECLARE v_indeks INTEGER;
    DECLARE v_prosek DOUBLE;
    DECLARE v_result VARCHAR(32672);

    SET v_indeks = indeks;
    SELECT AVG(CASE
           WHEN status = 'o' AND ocena > 5 THEN 1.0 * ocena
           ELSE NULL
       END)
    INTO v_prosek
    FROM da.ispit
    WHERE indeks = v_indeks;

    SELECT LISTAGG(CAST(indeks AS VARCHAR(32672)), ', ')
    INTO v_result
    FROM (
      SELECT indeks
      FROM da.ispit
      GROUP BY indeks
      HAVING AVG(CASE
             WHEN status = 'o' AND ocena > 5 THEN 1.0 * ocena
             ELSE NULL
         END) < v_prosek
         AND SUM(CASE
                WHEN status = 'o' AND ocena > 5 THEN 1
                ELSE 0
             END) > br_predmeta
     );

    RETURN v_result;
END @
```
