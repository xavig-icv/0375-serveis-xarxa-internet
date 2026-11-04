## 03. Modes de Connexió: Actiu i Passiu

La connexió de **control** sempre l'obre el client cap al port 21 del servidor. En canvi, la connexió de **dades** es pot obrir de dues maneres diferents, i aquesta és la diferència entre el **mode actiu** i el **mode passiu**. Entendre-ho és imprescindible per configurar correctament el servidor i el tallafoc.

**Pregunta clau:** Qui obre la connexió de dades?

- **Mode actiu**: la connexió de dades l'obre el **servidor** cap al client.
- **Mode passiu**: la connexió de dades l'obre el **client** cap al servidor.

### Mode Actiu (PORT)

És el mode original de l'FTP. El client obre un port local i li diu al servidor: "connecta't tu a mi en aquest port".

```
CLIENT (192.168.1.50)                                 SERVIDOR (192.168.1.2)
port aleatori 50000                                   port 21 (control)
      │                                                       │
      │ 1. Connexió de control: 192.168.1.50:50000 → :21      │
      │──────────────────────────────────────────────────────>│
      │                                                       │
      │ 2. El client obre el port 50001 en escolta i envia:   │
      │    PORT 192,168,1,50,195,81                           │
      │──────────────────────────────────────────────────────>│
      │<─────────── 200 PORT command successful ──────────────│
      │                                                       │
      │ 3. LIST / RETR / STOR                                 │
      │──────────────────────────────────────────────────────>│
      │                                                       │
      │ 4. El SERVIDOR obre la connexió de dades:             │
      │    192.168.1.2:20  →  192.168.1.50:50001              │
      │<══════════════════════════════════════════════════════│
      │                                                       │
```

**Com es calcula el port?** L'ordre `PORT` envia 6 números: els 4 primers són la IP i els 2 últims el port (p1, p2):

```
PORT 192,168,1,50,195,81

IP   = 192.168.1.50
Port = p1 × 256 + p2 = 195 × 256 + 81 = 49920 + 81 = 50001
```

**Problema del mode actiu:** és el **servidor qui inicia una connexió entrant cap al client**. Això avui dia gairebé mai funciona perquè:

- El **tallafoc del client** (Windows Defender, `ufw`) bloqueja les connexions entrants.
- Si el client és darrere d'un **router amb NAT**, envia la seva IP privada a l'ordre `PORT` i el servidor no hi pot arribar des d'Internet.

**Avantatge:** al servidor només cal obrir els ports **21** (entrant) i **20** (sortint).

### Mode Passiu (PASV)

Per solucionar els problemes del mode actiu es va crear el mode passiu. Ara el client demana al servidor: "obre tu un port i jo m'hi connecto".

```
CLIENT (192.168.1.50)                                 SERVIDOR (192.168.1.2)
port aleatori 50000                                   port 21 (control)
      │                                                       │
      │ 1. Connexió de control: 192.168.1.50:50000 → :21      │
      │──────────────────────────────────────────────────────>│
      │                                                       │
      │ 2. PASV                                               │
      │──────────────────────────────────────────────────────>│
      │                                                       │ El servidor obre un
      │<─── 227 Entering Passive Mode (192,168,1,2,156,64) ───│ port del rang passiu
      │                                                       │ (40000) en escolta
      │ 3. El CLIENT obre la connexió de dades:               │
      │    192.168.1.50:50001  →  192.168.1.2:40000           │
      │══════════════════════════════════════════════════════>│
      │                                                       │
      │ 4. LIST / RETR / STOR                                 │
      │──────────────────────────────────────────────────────>│
      │<═══════════════ dades ════════════════════════════════│
```

**Càlcul del port passiu:**

```
227 Entering Passive Mode (192,168,1,2,156,64)

IP   = 192.168.1.2
Port = 156 × 256 + 64 = 39936 + 64 = 40000
```

**Avantatges del mode passiu:**

- Totes les connexions les inicia el client: **funciona darrere de NAT i tallafocs** al costat del client.
- És el **mode per defecte** a tots els clients moderns (FileZilla, lftp, curl, navegadors antics).

**Inconvenients:**

- Al servidor cal obrir el port 21 **i un rang de ports** per a les dades (per exemple 40000-40100).
- Si el servidor és darrere d'un NAT, ha d'anunciar la seva **IP pública** a la resposta `227` (opció `pasv_address` de vsftpd).

### Modes Estesos: EPRT i EPSV (IPv6)

Les ordres `PORT` i `PASV` només admeten adreces **IPv4**. L'RFC 2428 va afegir dues ordres noves que també funcionen amb **IPv6**:

| Ordre clàssica | Ordre estesa | Exemple de resposta                                  |
| -------------- | ------------ | ---------------------------------------------------- |
| `PORT`         | `EPRT`       | `EPRT \|2\|2001:db8::10\|50001\|`                    |
| `PASV`         | `EPSV`       | `229 Entering Extended Passive Mode (\|\|\|40000\|)` |

`EPSV` **no envia la IP**, només el port: el client es connecta a la mateixa IP de la connexió de control. Per això també funciona millor amb NAT. Molts clients moderns proven primer `EPSV` i, si falla, `PASV`.

### Comparativa: Actiu vs Passiu

| Característica                   | Mode Actiu (PORT)                  | Mode Passiu (PASV)                     |
| -------------------------------- | ---------------------------------- | -------------------------------------- |
| Qui obre la connexió de dades    | El servidor                        | El client                              |
| Port de dades al servidor        | 20 (origen)                        | Port aleatori del rang configurat      |
| Port de dades al client          | Aleatori (> 1023), en escolta      | Aleatori (> 1023), d'origen            |
| Ports a obrir al tallafoc del servidor | 21 entrant, 20 sortint       | 21 i rang passiu entrants              |
| Funciona amb client darrere NAT  | No (normalment)                    | Sí                                     |
| Funciona amb servidor darrere NAT| Sí                                 | Sí, amb `pasv_address`                 |
| Ús actual                        | Xarxes internes, clients antics    | **Opció per defecte i recomanada**     |

### FTP, Tallafocs i NAT

Els tallafocs amb estat (*stateful*) només deixen passar les connexions que coneixen. Com que el port de dades es negocia **dins de la connexió de control**, el tallafoc no el pot preveure. Hi ha dues solucions:

**1. Obrir un rang fix de ports passius (la més habitual)**

- Es limita el rang a vsftpd (`pasv_min_port` i `pasv_max_port`).
- S'obre aquest rang al tallafoc del servidor.
- **És l'única opció quan l'FTP va xifrat (FTPS)**.

**2. Mòduls d'ajuda de connexions (*conntrack helpers*)**

- Alguns tallafocs (mòdul `nf_conntrack_ftp` de Linux, routers domèstics) "llegeixen" la connexió de control, detecten les respostes `PORT`/`227` i obren el port automàticament.
- **No funciona amb FTPS**, perquè la connexió de control va xifrada i el tallafoc no la pot llegir.
- Als nuclis Linux actuals, aquests mòduls s'han d'activar explícitament per motius de seguretat.

```
                          ┌──────────────────────────┐
CLIENT ── NAT ── Internet │ TALLAFOC DEL SERVIDOR    │── SERVIDOR FTP
                          │  21/tcp        → OBERT   │
                          │  40000-40100   → OBERT   │   pasv_min_port=40000
                          │  resta de ports→ TANCAT  │   pasv_max_port=40100
                          └──────────────────────────┘
```

**Per què SFTP no té aquest problema?** SFTP funciona sobre una **única connexió SSH** (port 22): ordres i dades viatgen per la mateixa connexió. Només cal obrir un port, i és un dels motius pels quals ha substituït l'FTP en molts entorns.

`Nota:` Ho comprovarem a la pràctica al punt 5, capturant el trànsit i veient les respostes `227` amb el càlcul del port, i al punt 9 provant els dos modes amb el client `ftp`.
