## 02. Imatge i Vídeo Digital

### La Imatge Digital

Hi ha dos tipus d'imatges digitals:

- **Mapa de bits** (*raster*): una quadrícula de punts de color (**píxels**). Les fotografies i les captures de pantalla en són. Si s'amplien massa, es veuen els píxels.
- **Vectorial:** formes geomètriques (línies, corbes, textos) descrites amb fórmules. Es poden ampliar sense perdre qualitat. És el format dels logotips, les icones i els diagrames.

**Paràmetres d'una imatge de mapa de bits:**

| Paràmetre                | Significat                                         | Exemple                                    |
| ------------------------ | -------------------------------------------------- | ------------------------------------------ |
| **Resolució**            | Píxels d'amplada × píxels d'alçada                 | 1920 × 1080 (unes 2 megapíxels)            |
| **Profunditat de color** | Bits per píxel                                     | 24 bits: 8 per a cada color (vermell, verd i blau), uns 16,7 milions de colors |
| **Canal alfa**           | Transparència de cada píxel                        | 32 bits = 24 de color + 8 de transparència |

**Mida sense comprimir:** 1920 × 1080 píxels × 3 bytes = 6.220.800 bytes, uns **6,2 MB** per imatge.

**Models de color:**

- **RGB** (vermell, verd i blau): el de les pantalles.
- **CMYK:** el de la impremta.
- **YUV** (Y'CbCr): el del vídeo. Separa la **lluminositat** (Y) del **color** (U i V). Com que l'ull és més sensible a la llum que al color, el vídeo guarda el color amb menys resolució. Amb el **mostreig 4:2:0**, el color es guarda amb la meitat de resolució horitzontal i vertical, i les dades es redueixen a la meitat sense que es noti.

### Formats d'Imatge

| Format   | Tipus                        | Compressió             | Transparència | Animació | Ús típic                                   |
| -------- | ---------------------------- | ---------------------- | ------------- | -------- | ------------------------------------------ |
| **JPEG** | Mapa de bits                 | Amb pèrdua             | No            | No       | Fotografies                                |
| **PNG**  | Mapa de bits                 | Sense pèrdua           | Sí            | No       | Captures de pantalla, gràfics, logotips    |
| **GIF**  | Mapa de bits, 256 colors     | Sense pèrdua (paleta)  | Sí (sí o no)  | Sí       | Animacions senzilles (format antic)        |
| **WebP** | Mapa de bits                 | Amb o sense pèrdua     | Sí            | Sí       | Web: substitueix JPEG i PNG, amb fitxers més petits |
| **AVIF** | Mapa de bits (basat en AV1)  | Amb o sense pèrdua     | Sí            | Sí       | Web: encara més compressió que WebP         |
| **HEIC** | Mapa de bits (basat en H.265) | Amb pèrdua            | Sí            | Sí       | Fotografies dels mòbils d'Apple            |
| **SVG**  | Vectorial (XML)              | -                      | Sí            | Sí       | Logotips, icones, diagrames                |

**Pràctica: convertir imatges** (al servidor 0375):

```bash
sudo apt install imagemagick webp
mkdir -p ~/video && cd ~/video

# Imatge de prova de 1920×1080, generada amb ffmpeg
ffmpeg -f lavfi -i testsrc2=size=1920x1080 -frames:v 1 prova.png

identify prova.png                               # format, resolució i profunditat de color
convert prova.png -quality 85 prova.jpg          # JPEG de qualitat 85 (de 0 a 100)
cwebp -q 80 prova.png -o prova.webp              # WebP de qualitat 80
convert prova.png -resize 640x prova-640.jpg     # miniatura de 640 píxels d'amplada

ls -lh prova*
```

`convert` i `identify` són les ordres d'**ImageMagick 6**, la versió d'Ubuntu 24.04. A ImageMagick 7, l'ordre és `magick`.

Una pàgina web pot oferir diversos formats de la mateixa imatge. El navegador fa servir el primer que sap mostrar:

```html
<picture>
  <source srcset="foto.avif" type="image/avif">
  <source srcset="foto.webp" type="image/webp">
  <img src="foto.jpg" alt="Descripció de la imatge">
</picture>
```

### El Vídeo Digital

Un vídeo és una seqüència d'imatges (**fotogrames**) que es mostren molt de pressa, normalment acompanyada d'una o més pistes d'àudio.

| Paràmetre                | Valors habituals                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------ |
| **Resolució**            | 640×360 (360p), 1280×720 (**HD**, 720p), 1920×1080 (**Full HD**, 1080p), 3840×2160 (**4K UHD**) |
| **Fotogrames per segon** | 24 (cinema), 25 (televisió europea), 30 (web), 50 o 60 (esports, videojocs)          |
| **Relació d'aspecte**    | 16:9 (actual), 4:3 (antic), 9:16 (vídeo vertical de mòbil)                           |
| **Escaneig**             | Progressiu (`p`): fotogrames sencers. Entrellaçat (`i`): línies parells i senars alternes (televisió antiga) |
| **Mostreig del color**   | 4:2:0 (la majoria de vídeos), 4:2:2 (professional)                                    |

### Per què Cal Comprimir el Vídeo

Un vídeo Full HD a 30 fotogrames per segon, sense comprimir i amb 24 bits per píxel:

```
1920 × 1080 píxels × 24 bits × 30 fotogrames/s = 1.492.992.000 bps ≈ 1,49 Gbps
1 hora = 1,49 Gbps × 3.600 s / 8 ≈ 672 GB
```

Amb el còdec H.264 a 5 Mbps, la mateixa hora ocupa 2,25 GB: unes **300 vegades menys**. Sense compressió, el vídeo per Internet no seria possible.

### Com es Comprimeix el Vídeo

- **Compressió espacial (dins de cada fotograma):** es comprimeix com una fotografia JPEG. S'eliminen els detalls que l'ull gairebé no veu.
- **Compressió temporal (entre fotogrames):** dos fotogrames seguits s'assemblen molt. En lloc de guardar-los sencers, es guarden les diferències i el moviment dels blocs d'imatge (**vectors de moviment**).

Per això hi ha tres tipus de fotogrames:

| Tipus                    | Contingut                                                         | Mida        |
| ------------------------ | ----------------------------------------------------------------- | ----------- |
| **I** (*intra*)          | Imatge completa. Es pot descodificar sola                          | La més gran |
| **P** (predit)           | Només les diferències respecte d'un fotograma anterior            | Mitjana     |
| **B** (bidireccional)    | Diferències respecte de fotogrames anteriors i posteriors         | La més petita |

```
 I  B  B  P  B  B  P  B  B  P  B  B  I  B  B  P ...
 │◄──────────────── GOP ─────────────►│
```

El **GOP** (*Group of Pictures*) és la seqüència des d'un fotograma I fins al següent:

- Un GOP llarg comprimeix més.
- Però només es pot començar a reproduir, o saltar a un punt del vídeo, des d'un fotograma I. A més, si una part es fa malbé, l'error dura fins al fotograma I següent.
- En streaming es fa servir **un fotograma I cada 2 segons**, aproximadament (apartats 06 i 07).

### Taxes de Bits Orientatives (H.264)

Valors semblants als que recomanen les plataformes de vídeo per a 30 fotogrames per segon:

| Resolució             | Taxa de vídeo          |
| --------------------- | ---------------------- |
| 426×240 (240p)        | 300-700 kbps           |
| 640×360 (360p)        | 400-1.000 kbps         |
| 854×480 (480p)        | 500-2.000 kbps         |
| 1280×720 (720p)       | 1.500-4.000 kbps       |
| 1920×1080 (1080p)     | 3.000-6.000 kbps       |
| 3840×2160 (4K)        | 13.000-34.000 kbps     |

A aquesta taxa s'hi ha de sumar la de l'àudio, normalment de 96 a 128 kbps en AAC. Els còdecs més moderns (H.265, VP9, AV1) aconsegueixen la mateixa qualitat amb un 30-50 % menys de taxa (apartat 03).
