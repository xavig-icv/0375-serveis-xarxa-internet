## 03. Còdecs, Contenidors i Reproductors

Com a l'àudio (Tema 7), cal distingir el **còdec**, l'algorisme que comprimeix el vídeo, del **contenidor**, el format del fitxer que porta el vídeo, l'àudio, els subtítols i les metadades.

### Còdecs de Vídeo

| Còdec              | Any  | Qui el defineix                    | Llicència                    | Compressió (aprox.)              | Ús típic                                         |
| ------------------ | ---- | ---------------------------------- | ---------------------------- | -------------------------------- | ------------------------------------------------ |
| **MPEG-2**         | 1995 | MPEG (ISO/IEC i ITU-T)             | Patents ja caducades         | Referència                       | DVD, televisió digital en definició estàndard    |
| **H.264 / AVC**    | 2003 | ITU-T i MPEG                       | Amb patents                  | El doble que MPEG-2              | **El més compatible:** web, mòbils, videoconferència |
| **H.265 / HEVC**   | 2013 | ITU-T i MPEG                       | Amb patents                  | Un 50 % menys de taxa que H.264  | 4K, televisió UHD, mòbils                        |
| **VP8**            | 2010 | Google                             | Lliure                       | Semblant a H.264                 | WebRTC                                           |
| **VP9**            | 2013 | Google                             | Lliure                       | Semblant a H.265                 | YouTube, WebRTC                                  |
| **AV1**            | 2018 | Alliance for Open Media            | Lliure i sense drets de patent | Un 30 % menys que VP9 i H.265  | Plataformes de vídeo, videoconferència moderna   |
| **H.266 / VVC**    | 2020 | ITU-T i MPEG                       | Amb patents                  | Un 50 % menys que H.265          | Encara gairebé no es fa servir                   |

- **Patents:** qui fabrica un producte comercial que codifica o descodifica H.264 o H.265 ha de pagar llicències. Per això Google, Netflix, Amazon i altres empreses van crear **AV1**, que es pot fer servir lliurement.
- **WebRTC** (la videoconferència dels navegadors) obliga els navegadors a suportar VP8 i H.264.
- **Codificació per programari o per maquinari.** A Linux, ffmpeg codifica per programari amb `libx264` (H.264), `libx265` (H.265), `libvpx` (VP8 i VP9) i `libsvtav1` o `libaom` (AV1). Les targetes gràfiques i molts processadors també tenen codificadors de maquinari (NVENC, Quick Sync, VA-API). Són molt més ràpids, però donen una mica menys de qualitat amb la mateixa taxa.

**Àudio dels vídeos:** normalment AAC (en MP4) o Opus (en WebM). Vegeu el Tema 7.

### Contenidors

| Contenidor          | Extensió        | Còdecs habituals                        | Ús típic                                              |
| ------------------- | --------------- | --------------------------------------- | ----------------------------------------------------- |
| **MP4**             | `.mp4`, `.m4v`  | H.264, H.265, AV1 + AAC                 | El més compatible: web, mòbils, televisors            |
| **WebM**            | `.webm`         | VP8, VP9, AV1 + Opus o Vorbis           | Web, amb formats lliures                              |
| **Matroska**        | `.mkv`          | Qualsevol                               | Arxiu; diverses pistes d'àudio i de subtítols          |
| **MPEG-TS**         | `.ts`           | H.264, H.265 + AAC                      | Televisió digital; fragments de l'HLS (apartat 06)    |
| **QuickTime**       | `.mov`          | Diversos                                | Edició de vídeo (Apple)                               |
| **FLV**             | `.flv`          | H.264 + AAC                             | Gravacions de directes RTMP (apartat 07)              |
| **AVI**             | `.avi`          | Còdecs antics                           | Obsolet                                               |

L'extensió no garanteix el còdec: un `.mp4` pot portar H.264 o H.265, i potser el reproductor no el sap descodificar. Per saber què porta un fitxer, cal fer servir `ffprobe` (apartat 04).

### Compatibilitat amb els Navegadors

| Combinació                    | Firefox                  | Chrome i Edge            | Safari                                 |
| ----------------------------- | ------------------------ | ------------------------ | -------------------------------------- |
| **MP4: H.264 + AAC**          | Sí                       | Sí                       | Sí                                     |
| **WebM: VP9 + Opus**          | Sí                       | Sí                       | Parcialment, en versions recents       |
| **AV1** (MP4 o WebM)          | Sí                       | Sí                       | Només amb maquinari que el descodifiqui |
| **MP4: H.265**                | Depèn del sistema i del maquinari | Depèn del sistema i del maquinari | Sí                    |

**H.264 + AAC en MP4 és l'única combinació que funciona a tot arreu.** Per això, una web de vídeo l'ofereix sempre, i pot afegir-hi un format més eficient (VP9 o AV1) per als navegadors que el suportin (apartat 05).

### Reproductors

| Reproductor           | Sistemes                  | Característiques                                                        |
| --------------------- | ------------------------- | ----------------------------------------------------------------------- |
| **VLC**               | Tots                      | Reprodueix gairebé qualsevol format i protocol (fitxers, HLS, RTMP, RTSP). També converteix i emet |
| **mpv**               | Linux, Windows, macOS     | Lleuger, per línia d'ordres; molt bo per a proves                       |
| **ffplay**            | Amb ffmpeg                | Proves ràpides                                                          |
| **Navegadors**        | Tots                      | Element `<video>` de les pàgines web; HLS amb la biblioteca hls.js (Safari el reprodueix sense biblioteca) |
| **Kodi**              | Tots                      | Centre multimèdia per al televisor                                      |

```bash
# Al client
sudo apt install vlc mpv
```

### Servidors i Plataformes de Vídeo

| Programari                  | Tipus                                                                   |
| --------------------------- | ----------------------------------------------------------------------- |
| **Apache, Nginx**           | Servidors web: vídeo sota demanda i HLS (apartats 05 i 06)              |
| **Nginx RTMP, MediaMTX, SRS** | Servidors de directes: reben l'emissió (RTMP, SRT) i la reparteixen (HLS, WebRTC). Farem servir Nginx RTMP (apartat 07) |
| **PeerTube**                | Plataforma de vídeo lliure i federada, com un YouTube autoallotjat      |
| **Jellyfin**                | Servidor multimèdia per a la xarxa local                                |
| **Owncast**                 | Plataforma de directes autoallotjada, amb xat                           |
| **YouTube, Vimeo, Twitch**  | Plataformes comercials                                                  |
| **Jitsi Meet, BigBlueButton** | Videoconferència (apartats 09 i 10). BigBlueButton s'integra amb Moodle |
