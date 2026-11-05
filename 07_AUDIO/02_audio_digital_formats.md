## 02. Àudio Digital i Formats

### Del So Analògic al So Digital

El so és una ona de pressió que varia de manera contínua (senyal **analògic**). Per guardar-lo o enviar-lo per la xarxa, cal convertir-lo en números (senyal **digital**). La conversió la fa la targeta de so en dos passos:

1. **Mostreig:** mesura l'amplitud de l'ona a intervals regulars. Per exemple, 44.100 vegades per segon.
2. **Quantificació:** cada mesura (mostra) es guarda amb un nombre fix de bits. Per exemple, 16 bits.

```
 Amplitud
    │        ●  ●
    │     ●        ●                         ● = mostra: el valor de l'ona
    │   ●            ●                           en un instant
 ───┼─●────────────────●──────────────●───► temps
    │                    ●          ●
    │                       ●  ●  ●
         |←→| interval entre mostres = 1 / freqüència de mostreig
```

| Paràmetre                     | Què indica                          | Valors habituals                                                       |
| ----------------------------- | ----------------------------------- | ---------------------------------------------------------------------- |
| **Freqüència de mostreig**    | Mostres per segon (Hz)              | 8 kHz (telèfon), 16 kHz (veu per IP), **44,1 kHz** (CD, música), **48 kHz** (vídeo), 96 kHz (estudi) |
| **Profunditat de bits**       | Bits de cada mostra                 | **16 bits** (CD), 24 bits (estudi de gravació)                         |
| **Canals**                    | Senyals independents                | 1 (mono: veu), 2 (estèreo: música), 6 (so envolvent 5.1)               |

- **Teorema de Nyquist:** per reproduir una freqüència, cal mostrejar a més del doble d'aquesta freqüència. L'oïda humana arriba fins a uns 20 kHz; per això el CD fa servir 44,1 kHz. La veu humana pot anar a 8 o 16 kHz.
- **Profunditat de bits:** cada bit afegeix uns 6 dB de marge dinàmic (diferència entre el so més fluix i el més fort). 16 bits donen uns 96 dB.

### Taxa de Bits i Mida dels Fitxers

La **taxa de bits** (*bitrate*) és la quantitat de dades per segon:

```
taxa de bits (bps) = freqüència de mostreig × profunditat de bits × canals
mida (bytes)       = taxa de bits × durada (s) / 8
```

**Exemple: qualitat CD**

- Taxa: 44.100 × 16 × 2 = 1.411.200 bps ≈ **1.411 kbps**.
- 1 minut: 1.411.200 × 60 / 8 = 10.584.000 bytes ≈ **10,6 MB**. Una hora ocupa uns 635 MB.
- El mateix minut en **MP3 a 128 kbps**: 128.000 × 60 / 8 = 960.000 bytes ≈ **0,96 MB**, unes 11 vegades menys.

| Ús                                  | Format                              | Taxa de bits   | Mida d'1 hora  |
| ----------------------------------- | ----------------------------------- | -------------- | -------------- |
| Qualitat CD sense comprimir         | PCM 44,1 kHz, 16 bits, estèreo      | 1.411 kbps     | 635 MB         |
| Música d'alta qualitat              | MP3 o AAC                           | 256-320 kbps   | 115-144 MB     |
| Ràdio en línia                      | MP3                                 | 128 kbps       | 57,6 MB        |
| Ràdio per a mòbils                  | Opus o AAC                          | 48-64 kbps     | 21,6-28,8 MB   |
| Podcast de veu                      | MP3 mono                            | 64 kbps        | 28,8 MB        |
| Telefonia IP                        | Opus                                | 16-32 kbps     | 7,2-14,4 MB    |

En aquests càlculs, 1 kbps = 1.000 bits per segon i 1 MB = 1.000.000 bytes.

### Compressió: Sense Pèrdua i amb Pèrdua

| Tipus              | Com funciona                                                                  | Reducció habitual      | Formats              | Ús                                 |
| ------------------ | ----------------------------------------------------------------------------- | ---------------------- | -------------------- | ---------------------------------- |
| **Sense pèrdua**   | Com un ZIP: es recupera exactament el so original                             | Ocupa un 50-70 % de l'original | FLAC, ALAC  | Arxiu, edició, música d'alta qualitat |
| **Amb pèrdua**     | Elimina el que l'oïda percep menys (model psicoacústic): sons tapats per altres més forts, freqüències gairebé inaudibles | 10 vegades o més | MP3, AAC, Vorbis, Opus | Distribució per Internet |

- Cada vegada que es torna a comprimir amb pèrdua un fitxer que ja ho estava, es perd més qualitat. Per això, **els originals es guarden en WAV o FLAC** i es comprimeixen una sola vegada per a cada format de distribució.
- **Taxa constant (CBR)**: sempre els mateixos bits per segon. L'ample de banda és previsible, per això es fa servir en streaming en directe.
- **Taxa variable (VBR)**: més bits als fragments complexos i menys als senzills. Dona més qualitat amb la mateixa mida; per això es fa servir en fitxers.

### Còdec i Contenidor

- **Còdec** (codificador-descodificador): l'algorisme que comprimeix i descomprimeix el so (MP3, AAC, Opus).
- **Contenidor:** el format del fitxer, que guarda el so codificat i les metadades (MP4, Ogg, WebM, WAV).

Un mateix còdec pot anar en contenidors diferents (Opus en Ogg o en WebM), i un contenidor pot portar còdecs diferents (Ogg pot portar Vorbis, Opus o FLAC). L'extensió del fitxer indica sobretot el contenidor. Per saber quin còdec porta un fitxer, cal mirar-ho amb `ffprobe` (apartat 03).

### Formats d'Àudio Més Habituals

| Format    | Compressió        | Contenidor i extensió        | Taxa habitual      | Llicència                                  | Ús típic                                      |
| --------- | ----------------- | ---------------------------- | ------------------ | ------------------------------------------ | --------------------------------------------- |
| **PCM**   | Cap               | WAV (`.wav`)                 | 1.411 kbps (CD)    | Lliure                                     | Gravació i edició                             |
| **FLAC**  | Sense pèrdua      | `.flac`                      | 700-1.000 kbps     | Lliure                                     | Arxiu, música d'alta qualitat                 |
| **ALAC**  | Sense pèrdua      | MP4 (`.m4a`)                 | Com FLAC           | Lliure des del 2011                        | Dispositius d'Apple                           |
| **MP3**   | Amb pèrdua        | `.mp3`                       | 64-320 kbps        | Lliure (les patents van caducar el 2017)   | Podcast, ràdio; funciona a tot arreu          |
| **AAC**   | Amb pèrdua        | MP4 (`.m4a`), `.aac`         | 64-256 kbps        | Amb patents                                | Dispositius mòbils, plataformes de vídeo      |
| **Vorbis** | Amb pèrdua       | Ogg (`.ogg`)                 | 96-320 kbps        | Lliure                                     | Jocs, programari lliure                       |
| **Opus**  | Amb pèrdua        | Ogg (`.opus`), WebM          | 6-510 kbps         | Lliure (estàndard de l'IETF, RFC 6716)     | Veu i música en temps real: WebRTC, videotrucades, streaming |

**Opus** és el còdec més eficient: a 64 kbps sona millor que l'MP3 a la mateixa taxa, i també funciona amb taxes molt baixes per a la veu. L'**MP3**, però, continua sent el format més compatible.

### Compatibilitat i Tipus MIME

Els servidors indiquen el format de cada fitxer amb el **tipus MIME** (capçalera `Content-Type`), i el navegador decideix com reproduir-lo a partir d'aquesta informació:

| Format              | Tipus MIME          | Navegadors                                        |
| ------------------- | ------------------- | ------------------------------------------------- |
| MP3                 | `audio/mpeg`        | Tots                                              |
| AAC (M4A)           | `audio/mp4`         | Tots                                              |
| Ogg (Vorbis, Opus)  | `audio/ogg`         | Firefox i Chrome; Safari només en versions recents |
| WebM (Opus)         | `audio/webm`        | Firefox i Chrome; Safari només en versions recents |
| FLAC                | `audio/flac`        | Tots els actuals                                  |
| WAV                 | `audio/wav`         | Tots                                              |
| Llista M3U          | `audio/x-mpegurl`   | L'obren els reproductors (VLC, mpv)               |

Per això, una web d'àudio ofereix sempre **MP3 o AAC** com a opció segura i, si vol, Opus com a opció més eficient (apartat 07).

### Metadades

Els fitxers d'àudio guarden informació sobre el contingut: títol, artista, àlbum, any, gènere i imatge de portada.

- **MP3:** etiquetes **ID3** (ID3v2 pot incloure la portada).
- **Ogg, Opus i FLAC:** *Vorbis comments*.
- **MP4 (AAC):** àtoms de metadades.

En una emissió en directe, el títol del que sona en cada moment viatja dins del mateix flux d'àudio, i els reproductors el mostren.

### Quin Format per a Cada Ús

| Necessitat                        | Format recomanat                                   | Motiu                                                     |
| --------------------------------- | -------------------------------------------------- | --------------------------------------------------------- |
| Guardar l'original o editar-lo    | WAV o FLAC                                         | Sense pèrdua: es pot tornar a codificar sense perdre qualitat |
| Podcast                           | MP3: 64 kbps mono (veu) o 128 kbps estèreo (música) | L'accepten tots els reproductors i directoris de podcasts |
| Ràdio en línia                    | MP3 a 128 kbps i, a més, Opus o AAC a 64 kbps      | Compatibilitat, i una opció per a connexions lentes        |
| Àudio en una pàgina web           | MP3 o AAC, i Opus com a alternativa                | Funciona a tots els navegadors                             |
| Veu en temps real                 | Opus                                               | Poc retard i bona qualitat amb poca taxa de bits           |
