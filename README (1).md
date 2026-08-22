# QR-siirto

Yhden tiedoston HTML-sovellus, joka siirtää tiedostoja kahden puhelimen välillä QR-koodeista koostetun videon avulla — ilman verkkoyhteyttä, Bluetoothia tai kaapelia. Kaikki toimii selaimessa (`qr-siirto.html`).

## Miten se toimii pähkinänkuoressa

1. **Lähettäjä** valitsee tiedoston ja nopeuden (QR-koodia/s), ja sovellus **renderöi koko tiedoston etukäteen valmiiksi videoksi**, jossa jokainen kuva on yksi QR-koodi. Video jää talteen ja sen voi myös ladata `.webm`-tiedostona.
2. Lähettäjä näyttää tämän videon puhelimen ruudulta.
3. **Vastaanottaja** nauhoittaa oman kameransa lähettäjän ruudusta (nappi "Aloita nauhoitus" / "Lopeta nauhoitus").
4. Nauhoituksen loputtua sovellus **käy tallennetun videon läpi kuva kuvalta** ja purkaa jokaisesta löytyneestä QR-koodista datapalasen.
5. Kun kaikki palaset on kerätty, tiedosto kootaan ja vastaanottaja voi ladata sen.

Kahden vaiheen malli (kuvaa → sitten käsittele) on tarkoituksella eri kuin suora reaaliaikainen skannaus: se sietää paremmin kameran nykäyksiä, autofokusta ja pakkauksesta johtuvia häiriöitä, koska koko nauhoite käydään läpi jälkikäteen tarkasti, eikä yhtäkään kehystä tarvitse napata "lennosta".

## Käyttö — Lähettäjä

1. Avaa `qr-siirto.html`, pysy välilehdellä **Lähetä**.
2. Valitse tiedosto ja säädä liukusäätimellä nopeus (1–60 QR-koodia/s).
3. Paina **Luo QR-video**. Sovellus tekee tämän kahdessa vaiheessa:
   - **Vaihe 1/2 — Muodostetaan QR-kuvia:** kaikki QR-koodit lasketaan valmiiksi kuviksi etukäteen (ei aikakriittinen, nopea).
   - **Vaihe 2/2 — Nauhoitetaan videota:** valmiit kuvat näytetään tasatahtisesti valitulla nopeudella ja nauhoitetaan videoksi taustalla (`MediaRecorder` + `<canvas>.captureStream()`).
4. Kun video on valmis:
   - Se toistuu automaattisesti silmukassa esikatselussa — voit näyttää sen suoraan tästä.
   - **Lataa video (.webm)** tallentaa videon puhelimelle, jolloin sen voi toistaa esim. Kuvat/Galleria-sovelluksesta tai lähettää eteenpäin muulla tavalla.
   - **Koko näyttö** suurentaa videon koko ruudulle — helpottaa vastaanottajan kameraa lukemaan koodit.
5. **Tee uusi video** palaa asetuksiin, jos haluat vaihtaa tiedoston tai nopeuden.

## Käyttö — Vastaanottaja

1. Avaa `qr-siirto.html`, valitse välilehti **Vastaanota** ja salli kameran käyttö.
2. Osoita kamera lähettäjän ruutuun/videoon niin, että QR-koodi täyttää suurimman osan kuva-alasta.
3. Paina **Aloita nauhoitus**. Anna videon toistaa niin monta kertaa kuin haluat — ylimääräiset toistot eivät haittaa, sillä samat palat vain ohitetaan uudelleen löydettäessä.
4. Paina **Lopeta nauhoitus** kun uskot saaneesi koko sisällön läpi.
5. Sovellus käsittelee nauhoitteen automaattisesti (näet edistymispalkin), tunnistaa jokaisen kuvista löytyvän QR-koodin ja täyttää palikkaruudukkoa sitä mukaa kun uusia paloja löytyy.
6. Jos kaikki palat eivät löytyneet ensimmäisellä kerralla, paina **Nauhoita lisää puuttuvia paloja** ja nauhoita uudelleen — jo kerätyt palat säilyvät, vain puuttuvat täydentyvät.
7. Kun kaikki palat on kerätty, tiedosto kootaan automaattisesti ja **Lataa tiedosto** -painike ilmestyy.

## Tekninen kuvaus

### Datan pilkkominen

Tiedosto pilkotaan **900 tavun** palasiin (`CHUNK_BYTES`). Jokainen pala koodataan Base64:ksi ja upotetaan yhteen QR-koodiin seuraavassa muodossa (putkella `|` eroteltuna):

```
QRT1|<siirtoId>|<palanIndeksi>|<palojenMäärä>|<tiedostonimiBase64>|<mimeTyyppi>|<tiedostonKokoTavuina>|<CRC32(hex)>|<palanDataBase64>
```

- `QRT1` — muodon tunniste/versio.
- `siirtoId` — satunnainen tunniste, joka erottaa eri siirrot toisistaan.
- `CRC32` — palan eheystarkistus. Jos QR luetaan väärin (esim. pakkauksen tai kameran kohinan takia), tarkistussumma ei täsmää ja pala hylätään hiljaisesti — se yritetään saada myöhemmin uudesta nauhoituksesta tai videon myöhemmästä toistosta.

Chunk-koko (900 tavua) on valittu tasapainoksi: video pakataan kertaalleen lähetyspäässä (VP8/VP9) ja kuvataan vielä toisen kerran kameralla vastaanottopäässä — liian tiheä QR-koodi ei kestäisi kahta pakkauskierrosta yhtä luotettavasti.

### Video lähetyspäässä — miksi kehykset näkyvät nyt tasaisesti

Aiemmassa versiossa jokainen QR-koodi laskettiin (`makeCode()`) juuri ennen näyttämistä ja seuraava kehys ajastettiin `setTimeout`-ketjulla suhteessa edelliseen. Tämä aiheutti kaksi ongelmaa: raskas QR-laskenta saattoi itsessään kestää pidempään kuin kehyksen näyttöaika suurilla nopeuksilla, ja `setTimeout`-viiveet kertyivät (drifttäsivät) ajan myötä, koska seuraava ajastus laskettiin aina edellisestä eikä absoluuttisesta kellosta.

Tämä on korjattu kahdella tavalla:

1. **Kaikki QR-kuvat lasketaan ja piirretään bitmapeiksi etukäteen** (`createImageBitmap`), ennen kuin nauhoitus edes alkaa. Nauhoituksen aikana ei siis enää tehdä yhtään raskasta laskentaa — pelkkää valmiin kuvan kopiointia canvasille.
2. **Nauhoituksen ajastus perustuu absoluuttiseen kelloon**, ei edellisestä kehyksestä laskettuun viiveeseen: joka `requestAnimationFrame`-kierroksella lasketaan `oikea kehysindeksi = kulunut aika ÷ kehyksen kesto` suoraan aloitushetkestä. Jos yksi silmukan kierros myöhästyy hieman, seuraava kierros korjaa tilanteen heti oikeaan kohtaan sen sijaan, että viive kasvaisi kumulatiivisesti. `canvas.captureStream()` (ilman kiinteää fps-arvoa) lähettää uuden videokehyksen aina kun canvas piirretään uudelleen, jolloin video vastaa tarkasti sitä, mitä kulloinkin näytöllä oikeasti oli.
3. Video jää selaimen muistiin `Blob`-objektina, josta sekä esikatselu että latauslinkki ladataan. `MediaRecorder` tallentaa sen WebM-muotoon (VP9 tai VP8, riippuen selaimen tuesta), bittinopeudella ~6 Mbit/s pakkausartefaktien minimoimiseksi.

### Videon käsittely vastaanottopäässä

- Kameranauhoitus tallennetaan myös `MediaRecorder`-rajapinnalla `Blob`-objektiksi.
- Nauhoitteen käsittely tapahtuu **kelaamalla** piilotettua `<video>`-elementtiä 0,05 sekunnin (20 näytettä/s) välein, piirtämällä kunkin kohdan kuva `<canvas>`-elementtiin ja ajamalla se `jsQR`-kirjaston läpi.
- Tämä on tarkempi kuin suora reaaliaikainen `requestAnimationFrame`-skannaus, koska jokainen ajanhetki käydään läpi järjestelmällisesti sen sijaan, että luotettaisiin kameran ja selaimen ehtivän näyttää jokaisen ruudun ajallaan.

### Rajoitukset ja vinkit

- Suositeltu nopeus **5–15 QR-koodia/s**. Suuremmilla nopeuksilla (jopa 60/s) videosta tulee vaativampi lukea kahden pakkauskierroksen ja kameran suljinajan takia — kannattaa nauhoittaa useampi toistokierros.
- Hyvä, tasainen valaistus ja vakaa kuvakulma parantavat tunnistustarkkuutta merkittävästi.
- Isot tiedostot tuottavat paljon kehyksiä (`tiedoston koko / 900 tavua`) — pidä tämä mielessä nopeutta valitessa, jotta videon kesto (kehysten määrä / nopeus) pysyy järkevänä.
- Kaikki käsittely tapahtuu paikallisesti selaimessa; mitään ei lähetetä verkkoon.
