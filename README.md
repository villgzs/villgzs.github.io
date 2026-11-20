# villgzs.github.io / home-assistant-arm32 wheels

[musllinux](https://villgzs.github.io/musllinux/)  
[musllinux-index](https://villgzs.github.io/musllinux-index/)  
[wheels32bit](https://github.com/villgzs/wheels32bit)

[**MarkDownGuide**](https://www.markdownguide.org/basic-syntax/)

```
BASIC CODE
10 PRINT "HELLO"
20 GOTO 10
```
[ASCIIflow](https://asciiflow.com/)

A **Owon PDS6062T** (USB VID:PID `5345:1234`) Linux alatt jól kezelhető nyílt forráskódú eszközökkel, mivel a gyártói szoftver Windows-ra van, és Wine-ban a USB-adatátvitel általában nem működik megbízhatóan.

### Ajánlott programok

| Program | Leírás | Link | Megjegyzés |
|---------|--------|------|------------|
| **owondump** | Speciálisan Owon PDS sorozatra (PDS5022, PDS6062 stb.) készült userspace USB-drájver. Letölti a hullámformát (vektorgram + BMP), majd szöveges formátumba alakítja (pl. gnuplot-hoz). | [github.com/u1f35c/owondump](https://github.com/u1f35c/owondump) | A legközvetlenebb és legjobban dokumentált megoldás a PDS6062-hez. |
| **LibOwonPds** | C könyvtár + parancssori segédprogram Owon PDS oszcilloszkópokhoz. CSV-be vagy PNG-be ment, van Python wrapper is. | [github.com/EarToEarOak/LibOwonPds](https://github.com/EarToEarOak/LibOwonPds) | PDS5022S-en tesztelték, de a PDS6062-vel is kompatibilisnek kell lennie. |

### Gyors indítás (owondump példán)

```bash
# Függőségek
sudo apt install libusb-dev   # vagy libusb-1.0-0-dev attól függően

# Klónozás és fordítás
git clone https://github.com/u1f35c/owondump.git
cd owondump
gcc -o owondump owondump.c -lusb
gcc -o owonfileread owonfileread.c -lusb

# Futtatás (root-ként vagy udev-szabály után)
./owondump
```

### Fontos: USB-hozzáférés beállítása

Hozz létre egy udev-szabályt, hogy ne kelljen root-ként futtatni:

```bash
sudo tee /etc/udev/rules.d/99-owon.rules << EOF
SUBSYSTEM=="usb", ATTR{idVendor}=="5345", ATTR{idProduct}=="1234", MODE="0660", GROUP="plugdev"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

(A felhasználód legyen a `plugdev` csoportban: `sudo usermod -aG plugdev $USER`, majd jelentkezz ki-be.)

### Egyéb megjegyzések

- Az újabb Linux kernelek (kb. 6.14+) a `usb_serial_simple` drivert is betöltik erre a VID:PID-re (főleg az újabb HDS sorozathoz SCPI támogatás miatt). Ha konfliktus van a libusb-alapú programokkal, ideiglenesen unloadolhatod:
  ```bash
  sudo modprobe -r usb_serial_simple
  ```
- A hivatalos Windows szoftver Wine alatt részben elindul, de a valós idejű adatátvitel általában nem működik.
- Ha csak egyszeri hullámforma-mentés kell, az `owondump` a legegyszerűbb. Ha programozni is szeretnél (pl. Pythonból), a LibOwonPds praktikusabb.

Ha elakadsz a fordításnál vagy a futtatásnál, írd meg a pontos hibaüzenetet, és segítek.


---

A hiba oka szinte biztosan az, hogy a **kernel** `usb_serial_simple` drivere automatikusan lefoglalja az eszközt (5345:1234).  
Ezt a drivert 2025-ben adták hozzá a kernelhez (főleg az újabb HDS sorozathoz), és sajnos ugyanezt a VID:PID-t használja a te PDS6062T-d is.

### Gyors megoldás (ideiglenes)

Próbáld ki ezt:

```bash
sudo modprobe -r usb_serial_simple
```

Utána azonnal futtasd újra:

```bash
./owondump
```

Ha sikerül, akkor tényleg ez volt a gond.

### Tartósabb megoldás – udev szabály

Hozz létre egy szabályt, ami automatikusan leválasztja a kernel drivert, amikor az oszcilloszkóp csatlakozik:

```bash
sudo tee /etc/udev/rules.d/99-owon-unbind.rules << 'EOF'
# Owon PDS / HDS / VDS közös VID:PID
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="5345", ATTR{idProduct}=="1234", MODE="0660", GROUP="plugdev"
ACTION=="bind", DRIVER=="usb_serial_simple", ATTRS{idVendor}=="5345", ATTRS{idProduct}=="1234", RUN+="/bin/sh -c 'echo %k > /sys/bus/usb/drivers/usb_serial_simple/unbind'"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

Ezután húzd ki és dugd vissza az oszcilloszkópot, majd próbáld újra az `./owondump`-ot.

### Ellenőrzés

Ha kíváncsi vagy, hogy tényleg a `usb_serial_simple` foglalja-e le:

```bash
lsusb -t
# vagy
dmesg | tail -30
```

Látnod kellene valami ilyesmit: `usb_serial_simple ... owon converter now attached to ttyUSB0`

Mondd el, mi történt a `modprobe -r` után!
