# Rakenna oma SMTP-postinlähetyspalvelin

## johdanto

SMTP voi ostaa suoraan palveluita pilvitoimittajilta, kuten:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali pilvi sähköpostin push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Voit myös rakentaa oman sähköpostipalvelimesi – rajoittamaton lähetys, alhaiset kokonaiskustannukset.

Alla esittelemme askel askeleelta oman sähköpostipalvelimen rakentamisen.

## Palvelimen valinta

Itseisännöity SMTP-palvelin vaatii julkisen IP-osoitteen, jonka portit 25, 456 ja 587 ovat avoinna.

Yleisesti käytetyt julkiset pilvet ovat oletuksena estäneet nämä portit, ja ne voi olla mahdollista avata antamalla työkäsky, mutta se on loppujen lopuksi erittäin hankalaa.

Suosittelen ostamaan isännältä, jolla on nämä portit auki ja joka tukee käänteisten verkkotunnusten määrittämistä.

Täällä suosittelen [Contaboa](https://contabo.com) .

Contabo on Münchenissä, Saksassa sijaitseva hosting-palveluntarjoaja, joka perustettiin vuonna 2003 erittäin kilpailukykyisin hinnoin.

Jos valitset ostovaluutaksi euron, hinta on halvempi (8 Gt muistilla ja 4 prosessorilla varustettu palvelin maksaa noin 529 yuania vuodessa ja alkuasennusmaksu on ilmainen vuodeksi).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Kun teet tilauksen, muista valita `prefer AMD` , jolloin AMD-suorittimella varustetun palvelimen suorituskyky on parempi.

Seuraavassa otan Contabon VPS:n esimerkkinä havainnollistaakseni oman sähköpostipalvelimen rakentamista.

## Ubuntu-järjestelmän kokoonpano

Käyttöjärjestelmä tässä on Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Jos ssh-palvelin näyttää `Welcome to TinyCore 13!` (kuten alla olevasta kuvasta näkyy), se tarkoittaa, että järjestelmää ei ole vielä asennettu. Irrota ssh-yhteys ja odota muutama minuutti ennen kuin kirjaudut uudelleen sisään.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Kun `Welcome to Ubuntu 22.04.1 LTS` tulee näkyviin, alustus on valmis ja voit jatkaa seuraavilla vaiheilla.

### [Valinnainen] Alusta kehitysympäristö

Tämä vaihe on valinnainen.

Mukavuuden vuoksi laitoin ubuntu-ohjelmiston asennuksen ja järjestelmämääritykset osoitteeseen [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Suorita seuraava komento asentaaksesi yhdellä napsautuksella.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kiinalaiset käyttäjät, käytä seuraavaa komentoa sen sijaan, jolloin kieli, aikavyöhyke jne. asetetaan automaattisesti.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo mahdollistaa IPV6:n

Ota IPV6 käyttöön, jotta SMTP voi lähettää myös sähköpostiviestejä IPV6-osoitteilla.

muokkaa `/etc/sysctl.conf`

Muokkaa tai lisää seuraavia rivejä

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Jatka [contabo-opetusohjelmaa: IPv6-yhteyden lisääminen palvelimeesi](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Muokkaa `/etc/netplan/01-netcfg.yaml` , lisää muutama rivi alla olevan kuvan mukaisesti (Contabo VPS:n oletusasetustiedostossa on jo nämä rivit, poista vain kommentit).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Tämän jälkeen `netplan apply` , jotta muokatut asetukset tulevat voimaan.

Kun määritys on onnistunut, voit tarkastella ulkoisen verkkosi ipv6-osoitetta `curl 6.ipw.cn` avulla.

## Kloonaa määritysarkisto ops

```
git clone https://github.com/wactax/ops.soft.git
```

## Luo ilmainen SSL-varmenne verkkotunnuksellesi

Sähköpostin lähettäminen vaatii SSL-varmenteen salausta ja allekirjoittamista varten.

Käytämme [acme.sh-](https://github.com/acmesh-official/acme.sh) tiedostoa varmenteiden luomiseen.

acme.sh on avoimen lähdekoodin automaattinen varmenteen allekirjoitustyökalu,

Siirry asetusvarastoon ops.soft, suorita `./ssl.sh` ja **ylempään hakemistoon** luodaan `conf` kansio.

Etsi DNS-palveluntarjoajasi osoitteesta [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , muokkaa `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Suorita sitten `./ssl.sh 123.com` luodaksesi `123.com` ja `*.123.com` varmenteet verkkotunnuksellesi.

Ensimmäinen ajo asentaa automaattisesti [acme.sh-tiedoston](https://github.com/acmesh-official/acme.sh) ja lisää ajoitetun tehtävän automaattista uusimista varten. Näet `crontab -l` , siellä on tällainen rivi.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Luodun varmenteen polku on esimerkiksi `/mnt/www/.acme.sh/123.com_ecc。`

Varmenteen uusiminen kutsuu `conf/reload/123.com.sh` -skriptin, muokkaa tätä komentosarjaa, voit lisätä komentoja, kuten `nginx -s reload` päivittääksesi liittyvien sovellusten varmennevälimuistin.

## Rakenna SMTP-palvelin chasquidillä

[chasquid](https://github.com/albertito/chasquid) on avoimen lähdekoodin SMTP-palvelin, joka on kirjoitettu Go-kielellä.

Vanhojen sähköpostipalvelinohjelmien, kuten Postfixin ja Sendmailin, korvikkeena chasquid on yksinkertaisempi ja helpompi käyttää, ja se on myös helpompi toissijaisessa kehittämisessä.

Suorita `./chasquid/init.sh 123.com` asennetaan automaattisesti yhdellä napsautuksella (korvaa 123.com lähettävällä verkkotunnuksellasi).

## Määritä sähköpostin allekirjoitus DKIM

DKIM:ää käytetään sähköpostien allekirjoitusten lähettämiseen, jotta kirjeitä ei käsitellä roskapostina.

Kun komento on suoritettu onnistuneesti, sinua pyydetään asettamaan DKIM-tietue (kuten alla on esitetty).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Lisää vain TXT-tietue DNS:ään (kuten alla).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Tarkastele palvelun tilaa ja lokeja

 `systemctl status chasquid` Näytä palvelun tila.

Normaalin toiminnan tila on alla olevan kuvan mukainen

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` tai `journalctl -xeu chasquid` voi tarkastella virhelokia.

## Käänteinen verkkotunnuksen määritys

Käänteinen verkkotunnus on mahdollistaa IP-osoitteen ratkaisemisen vastaavaksi verkkotunnukseksi.

Käänteisen verkkotunnuksen nimen asettaminen voi estää sähköpostien tunnistamisen roskapostiksi.

Kun sähköposti vastaanotetaan, vastaanottava palvelin suorittaa käänteisen verkkotunnuksen analyysin lähettävän palvelimen IP-osoitteelle varmistaakseen, onko lähettävällä palvelimella kelvollinen käänteinen toimialueen nimi.

Jos lähettävällä palvelimella ei ole käänteistä verkkotunnusta tai jos käänteinen toimialueen nimi ei vastaa lähettävän palvelimen IP-osoitetta, vastaanottava palvelin voi tunnistaa sähköpostin roskapostiksi tai hylätä sen.

Käy osoitteessa [https://my.contabo.com/rdns](https://my.contabo.com/rdns) ja määritä alla olevan kuvan mukaisesti

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Kun olet asettanut käänteisen toimialueen nimen, muista määrittää toimialueen nimen ipv4 ja ipv6 välitystarkkuus palvelimelle.

## Muokkaa chasquid.conf-palvelimen nimeä

Muokkaa `conf/chasquid/chasquid.conf` käänteisen toimialueen nimen arvoksi.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Käynnistä sitten palvelu uudelleen suorittamalla `systemctl restart chasquid` .

## Varmuuskopioi conf git-arkistoon

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Varmuuskopioin esimerkiksi conf-kansion omaan github-prosessiini seuraavasti

Luo ensin yksityinen varasto

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Anna conf-hakemisto ja lähetä se varastoon

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Lisää lähettäjä

juosta

```
chasquid-util user-add i@wac.tax
```

Voi lisätä lähettäjän

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Varmista, että salasana on asetettu oikein

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Käyttäjän lisäämisen jälkeen `chasquid/domains/wac.tax/users` päivitetään, muista lähettää se varastoon.

## DNS lisää SPF-tietue

SPF (Sender Policy Framework) on sähköpostin vahvistustekniikka, jota käytetään estämään sähköpostipetoksia.

Se varmistaa sähköpostin lähettäjän henkilöllisyyden tarkistamalla, että lähettäjän IP-osoite vastaa sen väitetyn verkkotunnuksen DNS-tietueita, mikä estää huijareita lähettämästä vääriä sähköposteja.

SPF-tietueiden lisääminen voi estää sähköpostien tunnistamisen roskapostiksi mahdollisimman paljon.

Jos verkkotunnuksesi nimipalvelimesi ei tue SPF-tyyppiä, lisää vain TXT-tyyppinen tietue.

Esimerkiksi `wac.tax` SPF on seuraava

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF for `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Huomaa, että olen `include:_spf.google.com` . Tämä johtuu siitä, että määritän `i@wac.tax` lähetysosoitteeksi Googlen postilaatikossa myöhemmin.

## DNS-määritys DMARC

DMARC on lyhenne sanoista (Domain-based Message Authentication, Reporting & Conformance).

Sitä käytetään sieppaamaan SPF-pomppuja (voivat johtua määritysvirheistä tai joku muu esiintyy sinuna lähettääkseen roskapostia).

Lisää TXT-tietue `_dmarc` ,

Sisältö on seuraava

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Kunkin parametrin merkitys on seuraava

### p (käytäntö)

Osoittaa, kuinka käsitellä sähköposteja, jotka epäonnistuvat SPF (Sender Policy Framework) tai DKIM (DomainKeys Identified Mail) -vahvistuksessa. Parametri p voidaan asettaa johonkin kolmesta arvosta:

* ei mitään: Mitään toimenpiteitä ei tehdä, vain vahvistustulos palautetaan lähettäjälle sähköpostiraportointimekanismin kautta.
* Karanteeni: Laita sähköposti, joka ei ole läpäissyt vahvistusta, roskapostikansioon, mutta se ei hylkää postia suoraan.
* hylkää: Hylkää suoraan sähköpostit, joiden vahvistus epäonnistuu.

### fo (vikavaihtoehdot)

Määrittää raportointimekanismin palauttamien tietojen määrän. Se voidaan asettaa johonkin seuraavista arvoista:

* 0: Raportoi kaikkien viestien vahvistustulokset
* 1: Ilmoita vain viesteistä, joiden vahvistus epäonnistuu
* d: Raportoi vain verkkotunnuksen varmistusvirheistä
* s: raportoi vain SPF-vahvistusvirheistä
* l: Ilmoita vain DKIM-vahvistusvirheistä

### rua & ruf

* rua (Raportointi-URI koontiraporteille): Sähköpostiosoite koottujen raporttien vastaanottamista varten
* ruf (Reporting URI for Forensic reports): sähköpostiosoite yksityiskohtaisten raporttien vastaanottamiseen

## Lisää MX-tietueita lähettääksesi sähköpostit Google Mailiin

Koska en löytänyt ilmaista yrityspostilaatikkoa, joka tukisi yleisiä osoitteita (Catch-All, voi vastaanottaa kaikki tähän verkkotunnukseen lähetetyt sähköpostit ilman etuliitteitä), välitin kaikki sähköpostit edelleen Gmail-postilaatikkooni chasquidilla.

**Jos sinulla on oma maksullinen yrityspostilaatikko, älä muuta MX:ää ja ohita tämä vaihe.**

Muokkaa `conf/chasquid/domains/wac.tax/aliases` , aseta edelleenlähetyspostilaatikko

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` tarkoittaa kaikkia sähköposteja, `i` on edellä luotu lähettävän käyttäjän sähköpostiosoitteen etuliite. Postin edelleenlähettämiseksi jokaisen käyttäjän on lisättävä rivi.

Lisää sitten MX-tietue (osoitan tässä suoraan käänteisen verkkotunnuksen osoitteeseen, kuten alla olevan kuvan ensimmäisellä rivillä näkyy).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Kun määritys on valmis, voit käyttää muita sähköpostiosoitteita sähköpostien lähettämiseen osoitteisiin `i@wac.tax` ja `any123@wac.tax` nähdäksesi, voitko vastaanottaa sähköpostiviestejä Gmailissa.

Jos ei, tarkista chasquid-loki ( `grep chasquid /var/log/syslog` ).

## Lähetä sähköposti osoitteeseen i@wac.tax Google Maililla

Kun Google Mail oli vastaanottanut sähköpostin, toivoin luonnollisesti voivani vastata osoitteella `i@wac.tax` i.wac.tax@gmail.com sijaan.

Siirry osoitteeseen [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) ja napsauta Lisää toinen sähköpostiosoite.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Syötä sitten vahvistuskoodi, joka vastaanotettiin sähköpostiin, johon lähetettiin edelleen.

Lopuksi se voidaan asettaa oletuslähettäjäosoitteeksi (sekä mahdollisuus vastata samalla osoitteella).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Tällä tavalla olemme saaneet päätökseen SMTP-sähköpostipalvelimen perustamisen ja samalla käytämme Google Mailia sähköpostien lähettämiseen ja vastaanottamiseen.

## Lähetä testisähköposti tarkistaaksesi, onko määritys onnistunut

Syötä `ops/chasquid`

Suorita `direnv allow` riippuvuuksien asentaminen (direnv on asennettu edellisessä yhden avaimen alustusprosessissa ja komentotulkkiin on lisätty koukku)

sitten juokse

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Parametrien merkitys on seuraava

* käyttäjä: SMTP-käyttäjätunnus
* pass: SMTP-salasana
* vastaanottaja: vastaanottaja

Voit lähettää testisähköpostin.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

On suositeltavaa käyttää Gmailia testisähköpostien vastaanottamiseen, jotta voidaan tarkistaa, onnistuvatko asetukset.

### TLS-standardin mukainen salaus

Kuten alla olevasta kuvasta näkyy, siinä on tämä pieni lukko, mikä tarkoittaa, että SSL-varmenne on otettu käyttöön onnistuneesti.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Napsauta sitten "Näytä alkuperäinen sähköposti"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Kuten alla olevasta kuvasta näkyy, Gmailin alkuperäisen sähköpostin sivulla näkyy DKIM, mikä tarkoittaa, että DKIM-määritys onnistui.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Tarkista alkuperäisen sähköpostin otsikosta Vastaanotettu, niin näet, että lähettäjän osoite on IPV6, mikä tarkoittaa, että myös IPV6 on määritetty onnistuneesti.
