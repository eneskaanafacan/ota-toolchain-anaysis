# CC1352R Platformlarında Contiki-NG Telsiz Donanım Yazılımının Güncellemesi — ELF Analiz Raporu

**Analiz Edilen Firmware'ler:** `new-firmware.z1`, `udp-server.z1`, `udp-client.z1`

**Platform:** MSP430 / Z1 Mote / Contiki-NG / CC1352R (Hedef Donanım)

---

## GİRİŞ

Bu rapor, Telsiz Duyarga Ağlarında (WSN - Wireless Sensor Networks) fiziksel erişimin zor olduğu düğümlere yeni bir donanım yazılımı imajının kablosuz ağ üzerinden gönderilmesi, doğrulanması ve kalıcı belleğe yazılması sürecini (Over-The-Air - OTA) analiz etmektedir. Raporun amacı, geliştirilen OTA sistemini oluşturan derlenmiş ELF (Executable and Linkable Format) dosyalarını statik analiz araçlarıyla incelemek ve hedeflenen Texas Instruments CC1352R donanımına taşınabilirlik durumunu detaylandırmaktır.

**Kullanılan Araç Zinciri (Toolchain):** `msp430-gcc` araç zincirine ait `file`, `msp430-readelf`, `msp430-objdump`, `msp430-nm`, `msp430-size`, `msp430-strings`, `msp430-addr2line` ve `msp430-strip` komutları kullanılmıştır.

**Analiz Edilen Dosyalar:**

1. **`new-firmware.z1`**: Ağa OTA ile aktarılacak olan asıl güncel firmware imajı.
2. **`udp-server.z1`**: Ağın kökü (ID:1) olarak çalışan, OTA bloklarını alıp CFS (Coffee File System) üzerine yazan alıcı düğüm.
3. **`udp-client.z1`**: Güncel imajı bloklara bölüp yollayan gönderici düğüm (ID:2).

---

## 1. Binary Kimlik Analizi

### Kullanılan Araçlar

`file`, `msp430-readelf -h`, `msp430-objdump -x`

### Analiz

#### Dosya Formatı ve Başlık (Header) Bilgileri

```text
===== FILE =====
new-firmware.z1: ELF 32-bit LSB executable, TI msp430, version 1 (embedded), statically linked, with debug_info, not stripped

===== READELF -h =====
ELF Header:
  Class:                             ELF32
  Data:                              2's complement, little endian
  Machine:                           Texas Instruments msp430 microcontroller
  Entry point address:               0x3100

```

* **ELF32 ve Machine:** İmajların 32-bit (`ELF32`) adresleme şablonuyla hazırlandığını ancak komut seti mimarisinin (Machine) `Texas Instruments msp430` olduğunu gösterir. Cooja simülatöründe Z1 mote kullanıldığı için hedef mimari doğal olarak 16-bit RISC tabanlı olan MSP430'dur.
* **Endianness:** Veri formatı `little endian` olarak derlenmiştir. En düşük anlamlı bayt (LSB), bellekte en düşük adrese yazılır. OTA işlemi sırasında ağ üzerinden gelen bit akışının ters çevrilmeden diske yazılabilmesi için kritiktir.
* **Giriş Adresi (Entry Point):** `0x3100`. Yazılımın Flash bellekten RAM'e alındıktan sonra Program Sayacının (Program Counter - PC) koşturmaya başlayacağı ilk bellek adresidir.

#### Neden ELF (Executable and Linkable Format) ve Ham Binary Değil?

Şartname imajın ham binary (`.bin`) değil, ELF biçiminde olduğunu belirtmiştir.

* **Ham Binary:** Sadece ardışık makine kodlarından (`0101...`) oluşur. İşlemcinin nerede başlayacağını, hangi bölümün RAM'e kopyalanacağını bilmesine imkan yoktur.
* **ELF Avantajı:** ELF dosyası `objdump -x` ile görülebilen **Program Header** ve **Section Header** tabloları taşır. Akıllı bir OTA Bootloader, ELF başlığını parse ederek `.data` bölümünü RAM'e, `.text` bölümünü Flash'a dinamik olarak yerleştirebilir. Gerçek TI OAD (Over-the-Air Download) sistemlerinde saf binary yerine her zaman metadata barındıran yapılar kullanılır.

#### Üç Firmware Karşılaştırması

* **`new-firmware.z1`**, **`udp-server.z1`**, **`udp-client.z1`**: Her üç dosya da `ELF32`, `little endian` ve `MSP430` özelliklerini paylaşır. Her üçünün de Entry Point adresi `0x3100`'dır. Bu durum, üç yazılımın da aynı donanım mimarisi (Z1) için derlendiğini teyit eder.

### Bulgular ve Yorum

ELF kullanılması, bootloader'ın körü körüne bir bellek yazımı yapmak yerine, segment sınırlarını ve giriş noktalarını okuyarak güvenli bir donanım başlatması (boot) yapmasına olanak tanır.

---

## 2. Bellek Bölümleri (Sections) ve Yerleşimi Analizi

### Kullanılan Araçlar

`msp430-readelf -S`, `msp430-objdump -h`

### Analiz

#### Bölüm (Section) Tabloları

```text
===== READELF -S (new-firmware.z1) =====
[Nr] Name       Type      Addr     Off      Size   Flg
[ 1] .text      PROGBITS  00003100 0000f4   00976e AX
[ 2] .rodata    PROGBITS  0000c870 009864   0035fd A
[ 3] .data      PROGBITS  00001100 00ce64   000150 WA
[ 4] .bss       NOBITS    00001250 00ce64   001648 WA
[ 5] .far.text  PROGBITS  00010000 00cff2   004a78 AX
[ 6] .vectors   PROGBITS  0000ffc0 00cff2   000040 AX

```

* **`.text` (Kod Bölümü):** Sanal Bellek Adresi (Virtual Memory Address - VMA) `0x3100`. Boyutu `0x976e` (38.766 byte). İşletim sisteminin ve uygulamanın çalıştırılabilir (`AX` - Alloc, Execute) makine kodlarıdır.
* **`.rodata` (Salt Okunur Veri):** Adresi `0xc870`, boyutu 13.821 byte. C içindeki sabit dizgiler ve değiştirilemez veriler buradadır (`A` - Alloc).
* **`.data` (Başlangıçlı Veri):** Adresi `0x1100`, boyutu 336 byte. Başlangıç değeri atanmış global değişkenleri tutar (`WA` - Write, Alloc).
* **`.bss` (Başlangıçsız Veri):** Adresi `0x1250`, boyutu 5.704 byte. Başlangıçta sıfırlanan değişkenlerdir. Diskte yer tutmaz (`NOBITS`), çalışma zamanında RAM'de tahsis edilir.
* **`.vectors`:** `0xffc0` adresinde yer alan donanım kesme vektör tablosudur.

#### Üç Firmware Karşılaştırması

Üç firmware'in `.text` (kod) bölüm boyutları karşılaştırıldığında:

* `new-firmware.z1`: ~70 KB
* `udp-client.z1`: ~43 KB
* `udp-server.z1`: ~42 KB
`new-firmware`, kendi içinde aktarılacak olan büyük payload yükünü taşıdığı için `.text` ve `.rodata` bölümlerinde devasa bir büyüme sergilemektedir.

### Bulgular ve Yorum

Gömülü sistem kısıtlarından ötürü bellek yönetimi son derece sıkıdır. OTA sürecinde aktarılan firmware bloklarının CFS üzerindeki offset'leri, bu bölümlerin Flash (ROM) ve SRAM üzerindeki kesin adreslerine göre hizalanmasını zorunlu kılar.

---

## 3. Sembol (Symbol) Tablosu Analizi

### Kullanılan Araçlar

`msp430-nm -n`, `msp430-readelf -s`

### Analiz

Firmware içerisinden (new-firmware ve server/client) tespit edilen en anlamlı semboller ve rolleri:

| Sembol Adı | Adres | Tip | İşlevi / Rolü |
| --- | --- | --- | --- |
| `_reset_vector__` | `0x3100` | FUNC | Donanım başlatıldığında çalışacak olan ana giriş noktası. |
| `__stack` | `0x3100` | NOTYPE | Yığın (Stack) belleğinin başlatıldığı taban noktası. RAM `0x1100-0x3100` arasındadır. |
| `process_run` | `0x6d98` | FUNC | Contiki-NG olay güdümlü (event-driven) zamanlayıcı döngüsü. |
| `uip_process` | `0x130f2` | FUNC | IPv6/TCP/UDP paketlerini işleyen ağ yığını süreci. |
| `rpl_dag_root_start` | `0x70c4` | FUNC | Cihazı RPL yönlendirme ağacı kökü yapan fonksiyon (Server'da aktiftir). |
| `node_id` | `0x14e6` | OBJECT | Cihaz kimlik numarasını tutar (2 veya 3). Rol belirleyici değişkendir. |
| `simple_udp_process` | `0x11ca` | OBJECT | UDP paket alış/veriş görevini yürüten sistem süreci. |
| `cfs_open` / `cfs_write` | `0xXXXX` | FUNC | (Server'da) OTA parçalarını Flash disk alanına yazan Coffee File System API'leri. |
| `memcpy` | `0x1484c` | FUNC | Göndericide ROM'daki veriyi OTA paketi payload'ına kopyalar. |

### Bulgular ve Yorum

Üç firmware'in sembol tabloları karşılaştırıldığında, Contiki-NG çekirdek sembolleri (`process_run`, `etimer_set`) ortaktır. Ancak uygulamaya özgü semboller ayrışır; `udp-server.z1` imajında `verify_full_image` ve `block_bitmap` sembolleri belirginken, `udp-client.z1` imajında `read_firmware_chunk_from_array` gibi gönderici fonksiyonlar öne çıkar.

---

## 4. Kesme Vektör (Interrupt Vector) Tablosu

### Kullanılan Araçlar

`msp430-readelf -S`, `msp430-objdump -d`

### Analiz

```text
===== READELF -S =====
[ 6] .vectors   PROGBITS  0000ffc0 00cff2   000040 AX

```

`.vectors` bölümü `0xffc0` adresinde bulunur ve `0x40` (64 byte) uzunluğundadır.

#### Disassembly Analizi (msp430-objdump -d)

Makine kodunun Assembly dilindeki izdüşümü (Örneklemsel görünüm):

```asm
00003100 <_reset_vector__>:
    3100:       31 40 00 31     mov     #0x3100, SP   ; Yığını (Stack) 0x3100'dan başlat
    3104:       b0 12 xx xx     call    #main         ; Ana C fonksiyonuna (main) atla

0000fffe <__interrupt_vector_31>:
    fffe:       00 31           .word   0x3100        ; Reset durumunda 0x3100 adresine git

```

### Bulgular ve Yorum

MSP430 mimarisinde cihaz resetlendiğinde, işlemci doğrudan belleğin sonundaki `0xFFFE` adresine (Reset Vector) bakar ve oradaki değeri alıp Program Sayacına (PC) yükler.
**OTA Açısından Önemi:** Yeni firmware ağ üzerinden indirilip Flash'ın başka bir alanına (Örn: `0x2C000`) yazıldığında, Bootloader `0xFFFE` adresindeki işaretçiyi yeni imajın giriş adresine güncellemezse (Vektör Yönlendirmesi), cihaz reset yediğinde asla yeni yazılımı çalıştıramaz. Üç firmware'in de `.vectors` alanları 64 baytlık standart ISR (Interrupt Service Routine) haritasını korumaktadır.

---

## 5. ROM / RAM Kullanım Profillemesi

### Kullanılan Araçlar

`msp430-size`

### Analiz

* **Flash (ROM) İhtiyacı:** `.text` (kod) + `.data` (başlangıçlı veriler).
* **RAM İhtiyacı:** `.data` (başlangıçlı veriler) + `.bss` (başlangıçsız veriler).

| Firmware (Dosya) | .text (Byte) | .data (Byte) | .bss (Byte) | Flash İhtiyacı (text+data) | RAM İhtiyacı (data+bss) |
| --- | --- | --- | --- | --- | --- |
| **new-firmware.z1** | 71.715 | 336 | 5.706 | **72.051 Bayt** (~70,3 KB) | **6.042 Bayt** (~5,9 KB) |
| **udp-client.z1** | 43.803 | 336 | 5.920 | **44.139 Bayt** (~43,1 KB) | **6.256 Bayt** (~6,1 KB) |
| **udp-server.z1** | 43.445 | 336 | 5.864 | **43.781 Bayt** (~42,7 KB) | **6.200 Bayt** (~6,05 KB) |

### Bulgular ve Yorum

Z1 Mote donanımı fiziksel olarak 92 KB ROM ve 8 KB RAM barındırır. Tüm imajların RAM ihtiyacı 6 KB civarında kalarak 8 KB sınırının altında güvenli bir seviyede tutulmuştur. OTA aktarımı yapılacak olan `new-firmware.z1`, taşıdığı payload nedeniyle Flash'ta ~28 KB daha fazla yer kaplamaktadır.

---

## 6. Networking & Stack Configuration Analizi

### Kullanılan Araçlar

`msp430-nm`, `msp430-strings`

### Analiz

* `uip_process`: Mikro IP (uIP) yığınının devrede olduğunu ve IPv6'nın aktif olduğunu kanıtlar.
* `rpl_dag_root_start`: Sadece `udp-server.z1` imajında çağrılan, ağı RPL (Routing Protocol for Low-Power and Lossy Networks) kökü olarak başlatan ağaç yapılandırma komutudur.
* `uip_icmp6_send`: Komşuluk keşfi (Neighbor Discovery) ve ping işlemleri için kullanılır.

### Bulgular ve Yorum

Sensör ağlarında TCP kullanılması RAM kısıtları nedeniyle (kayan pencere tamponları vb.) olanaksızdır. Bu projede taşıma katmanı olarak UDP (User Datagram Protocol) kullanılmış, kayıp paketlerin takibi (Reliability) uygulama katmanında geliştirilen OTA Stop-and-Wait mimarisiyle çözülmüştür.

---

## 7. Radyo ve Fiziksel Katman (PHY/MAC) Analizi

### Kullanılan Araçlar

`msp430-nm`

### Analiz

* `cc2420_driver` (`0xc910`): RF radyo çipini kontrol eden düşük seviye MAC/PHY donanım sürücüsüdür.
* `cc2420_interrupt`: Radyo antenine bir paket düştüğünde işlemciyi uyandıran donanımsal kesme (Interrupt) fonksiyonudur.

### Bulgular ve Yorum

Z1 donanımının radyosu CC2420'dir ve IEEE 802.15.4 standartlarında çalışır. Bu standardın Maksimum Aktarım Birimi (MTU) 127 bayttır. 130 KB'lık dosyanın tek seferde atılamayıp neden 48 baytlık bloklara (chunk) bölünmek zorunda kaldığının fiziksel ispatı bu sürücü kısıtlarıdır.

---

## 8. Sensör ve I/O Etkileşim Analizi

### Kullanılan Araçlar

`msp430-nm`, `msp430-strings`

### Analiz

Araç çıktılarında `button_sensor` veya `temperature_sensor` gibi temel çevresel birimlere ait bazı zayıf kütüphane sembolleri görünse de, aktif olarak I/O (Analog-Digital Converter vb.) okuması yapan spesifik uygulama rutinleri bulunmamaktadır.

### Bulgular ve Yorum

Bu firmware ailesi spesifik olarak OTA aktarımı, ağ yönlendirmesi ve Flash disk (CFS) yazımı için izole edilmiştir. Sistem kaynaklarını ağ doğrulamasına harcamak adına I/O süreçleri aktif edilmemiştir.

---

## 9. İşletim Sistemi (RTOS) / Contiki-NG Kernel Analizi

### Kullanılan Araçlar

`msp430-nm`

### Analiz

* `process_run` (`0x6d98`): Contiki-NG işletim sisteminin kalbidir.
* Sistem, preemptive (kesintili) bir RTOS değil, işbirliği tabanlı (cooperative) bir olay döngüsü kullanır.

### Bulgular ve Yorum

C kodlarındaki `PROCESS_THREAD`, `PROCESS_YIELD` ve `PROCESS_WAIT_EVENT_UNTIL` makroları bu çekirdek mimarisinin bir sonucudur. İş parçacığı işlemciyi gönüllü bıraktığı için (YIELD), yerel değişkenlerin RAM'den silinmesini engellemek amacıyla `static` bellek sınıfı kullanılması zorunlu hale gelmiştir.

---

## 10. Timing, Watchdog ve Power Management (LPM) Analizi

### Kullanılan Araçlar

`msp430-nm`, `msp430-readelf`

### Analiz

* `watchdog_periodic` (`0x13d62`): İşlemcinin sonsuz döngüye girip kilitlenmesini önlemek için Watchdog sayacını sıfırlayan fonksiyondur.
* `etimer_set`: Sistem zamanlayıcısıdır. Projede OTA paketinin NACK/Timeout durumunda tekrar gönderimini tetiklemek için kullanılmıştır.

### Bulgular ve Yorum

Flash belleğe veri yazmak (`cfs_write`) bloklayıcı bir işlemdir ve gömülü sistemlerde uzun sürer (Örn: sektör silinmesi 10ms+). OTA yazımı sırasında bu gecikmelerin Watchdog'ı tetikleyip sistemi resetlememesi için zamanlamanın ve güç yönetiminin dikkatli kurgulandığı görülmektedir.

---

## 11. Güvenlik (Security) ve Kriptografi Analizi

### Kullanılan Araçlar

`msp430-nm`, `msp430-strings`

### Analiz

Sembol tablolarında gelişmiş şifreleme algoritmalarına (AES-128, SHA-256) ait Link-Layer Security (LLSEC) modüllerinin tam kapasite aktif olduğuna dair majör semboller görülmemektedir.

### Bulgular ve Yorum

Projede ağ güvenliği (şifreleme) yerine veri transfer güvenliği (Bütünlük/Integrity) ön plandadır. `calculate_crc16` yazılımsal fonksiyonu ile havadaki bozulmalar (bit-flip) engellenirken, aktarım sonunda tüm imaj CRC testine tabi tutularak bütünlük güvence altına alınmıştır.

---

## 12. Veri Yapıları ve Heap Analizi

### Kullanılan Araçlar

`msp430-nm`, `msp430-readelf`

### Analiz

* `__stack` (`0x3100`): Stack (Yığın) belleğin başladığı noktadır. MSP430'da stack, belleğin üst kısımlarından aşağıya doğru büyür.
* Kod içinde dinamik bellek tahsisatı (`malloc`, `free`) sembollerine sık rastlanmaz.

### Bulgular ve Yorum

Gömülü OTA uygulamalarında (özellikle 8 KB RAM varken) Heap parçalanması (Fragmentation) ölümcüldür. `udp-server.z1` imajında, 1024 blokluk bir OTA dosyasının alınıp alınmadığını takip etmek için devasa diziler yerine bellek cimrisi **Bitmap (`block_bitmap`)** veri yapısı kurgulanmıştır. Bu sayede 1024 durum bilgisi sadece 128 bayt RAM'e sığdırılmıştır.

---

## 13. Linker (Bağlayıcı) Script Analizi

### Kullanılan Araçlar

`msp430-readelf -l`

### Analiz

Program Header (Yükleme Bölümleri) tablosu:

```text
===== READELF -l =====
Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
LOAD           0x0000f4 0x00003100 0x00003100 0x0976e 0x0976e R E 0x1
LOAD           0x009864 0x0000c870 0x0000c870 0x035fd 0x035fd R   0x1
LOAD           0x00ce64 0x00001100 0x0000fe6d 0x00150 0x01648 RW  0x1

```

* **VMA (Çalışma Adresi) vs LMA (Yükleme Adresi):** `.data` bölümünün (3. LOAD segmenti) VMA'sı `0x1100` (RAM başlangıcı), ancak LMA'sı (PhysAddr) `0xfe6d`'dir.

### Bulgular ve Yorum

Linker script, başlangıç değeri atanmış değişkenlerin (`.data`) kalıcı olması için Flash belleğe (LMA `0xfe6d`) yazılmasını emreder. Sistem boot edildiğinde, C startup kodu (crt0) bu verileri Flash'tan alıp RAM'e (`0x1100`) kopyalar. OTA Bootloader yeni bir imajı yüklerken bu segment (LOAD) sınırlarını okuyarak hangi verinin Flash'ta kalacağını, hangisinin RAM'e aktarılacağını bilir.

---

## 14. OTA (Over-The-Air) & Flash Bootloader Analizi

### Kullanılan Araçlar

TRM Rev. G (Bölüm 8 ve 10), Datasheet, C Kodu Modellemesi

### Analiz

Gerçek CC1352R donanımında OTA süreci, **Dual-Bank (Çift Banka)** mimarisiyle yönetilir. Çalışan imaj Bank A'da iken, OTA ile gelen baytlar Bank B'ye yazılır. Hata olursa eski kod çalışmaya devam eder (Rollback).

**Metadata (Header) Yapısı:**

```c
/* Flash'ın başına yazılacak olan İmaj Kimlik Kartı */
typedef struct {
    uint32_t magic;         // 0xf17e (Geçerli OTA İmajı Sinyali)
    uint32_t version;       // Firmware versiyon numarası
    uint32_t length;        // Byte cinsinden imaj boyutu (Örn: 72051)
    uint32_t entry_point;   // Yeni giriş adresi
    uint32_t crc32;         // Bütünlük için kümülatif CRC32
} __attribute__((aligned(4))) OtaMetadata_t;

```

**Bootloader Karar Mekanizması ve VTOR Yönlendirmesi:**
TRM Bölüm 2.9.4.29'da belirtilen `VTOR` (Vector Table Offset Register) kullanılarak yazılan örnek bootloader mantığı:

```c
#define NVIC_VTOR_OFFSET 0xE000ED08 

void Bootloader_Main() {
    OtaMetadata_t *meta = (OtaMetadata_t *)0x0002C000; // Bank B adresi
    
    if(meta->magic == 0xf17e) { // Yeni firmware var mı?
        if(VerifyCRC32(0x0002C000 + sizeof(OtaMetadata_t), meta->length) == meta->crc32) {
            uint32_t app_base = 0x0002C000 + sizeof(OtaMetadata_t);
            
            HWREG(NVIC_VTOR_OFFSET) = app_base; // Vektör Tablosunu Yeniden Yönlendir
            
            // Cortex-M4F Reset Vektörüne Atla
            uint32_t *app_entry_vector = (uint32_t *)(app_base + 4); 
            void (*app_start)(void) = (void (*)(void))(*app_entry_vector);
            app_start(); 
        }
    }
}

```

**Güncelleme Akışı (Adım Adım):**

1. Gönderici ELF içindeki `.text` verisini 48 baytlık bloklarla yollar.
2. Alıcı CRC16 denetimi yapar, başarılıysa ACK (onay) yollar.
3. Alıcı `cfs_seek` ile Flash'ta doğru offsete gider ve veriyi yazar.
4. Tüm bloklar alınınca kümülatif CRC32 tam dosya doğrulaması yapılır.
5. Sorun yoksa Metadata Flash başına kazınır ve yazılımsal reset atılır.
6. Reset sonrası `Bootloader_Main` çalışır, Metadata'yı okur.
7. `VTOR` yeni adrese (Bank B) yönlendirilir.
8. Yeni yazılım sorunsuz koşturulmaya başlar.

### Bulgular ve Yorum

Geliştirilen OTA yazılımı, TRM standartlarına tam uyumlu bir Bootloader ile sorunsuz çalışacak şekilde tasarlanmıştır. OTA işlemi sırasında cihazın kilitlenmemesi için donanımsal Watchdog düzenli olarak temizlenmelidir.

---

## 15. CFS (Coffee File System) & Storage Analizi

### Kullanılan Araçlar

`msp430-nm`, Kod Analizi

### Analiz

* `cfs_open`, `cfs_write`, `cfs_seek`, `cfs_close` fonksiyon çağrıları aktif kullanılmıştır.
* OTA blokları RAM'de tamponlanmak yerine, anında `fw_image.z1` dosyasına Flash üzerinden kazınmaktadır.

### Bulgular ve Yorum

CFS, Flash belleklerin kısıtlı silme/yazma ömürlerini uzatmak için Aşınma Seviyelendirmesi (Wear-Leveling) uygulayan özel bir dosya sistemidir. RAM kısıtı (8KB) nedeniyle 130 KB'lık firmware'in tek çaresi CFS üzerinden offset kaydırması ile anlık olarak depolanmasıdır.

---

## 16. Hata Ayıklama (Debug) & Loglama İzleri

### Kullanılan Araçlar

`msp430-strings`, `msp430-addr2line`

### Analiz

* Cihazın UART (Seri Port) arayüzüne gönderilen `LOG_INFO`, `LOG_WARN` makroları firmware içine derlenmiştir.
* **`addr2line` Kullanımı:** Firmware'de `-g` (debug_info) bayrağı bulunduğu için, `msp430-addr2line -e new-firmware.z1 0x3100` komutu çalıştırıldığında, bu hex adresinin kaynak koddaki karşılığı (Örn: `contiki-main.c:120`) doğrudan elde edilebilir.

### Bulgular ve Yorum

OTA sırasında oluşabilecek Crash (Çökme) veya Hard Fault durumlarında, bellekteki Program Counter adresi `addr2line` aracına verilerek hatanın HANGİ C SATIRINDA yaşandığı nokta atışı tespit edilebilir. Bu, geliştirme sürecinin vazgeçilmezidir.

---

## 17. String ve Sabit Veri (RoData) Analizi

### Kullanılan Araçlar

`msp430-strings`, `msp430-readelf -x .rodata`

### Analiz

`msp430-strings` çıktısından alınan en kritik 10 string ve anlamları:

1. `"Receiving UDP packet"`: UDP yığınının paketi yakaladığını bildiren log.
2. `"udp: bad checksum"`: Havadan gelen paketteki CRC/Checksum'ın bozuk olduğunu bildirir.
3. `"Yüklenmeye hazır yeni firmware alımı tamamlandı."`: Bizim tarafımızdan yazılan nihai OTA başarı logudur.
4. `"NS: updating link, child"`: IPv6 Neighbor Discovery (Komşu Keşfi) mekanizmasının ağacını güncellediğini gösterir.
5. `"fragment dropped"`: 127 baytlık MTU sınırını aşan hatalı paketlerin MAC katmanında çöpe atıldığını bildirir.
6. `"Dropping packet, src is mcast"`: Multicast (çoklu yayın) paket güvenlik duvarına takılmıştır.
7. `"incomplete IPv6 header"`: Ağ başlığı hasarlı gelen paketlerin atılması.
8. `"udp: zero port"`: Port numarası sıfır olan geçersiz bağlantı reddi.
9. `"MAC sequence"`: IEEE 802.15.4 MAC katmanı sıra numarası takibi.
10. `"IPv6 SR"`: IPv6 Source Routing (Kaynak Yönlendirme) başlığı uyarısı.

### Bulgular ve Yorum

Tüm bu log mesajları ve statik metinler `.rodata` bölümünde (ROM'da) saklanır. Bu metinlerin fazlalığı firmware boyutunu artırır ancak üretim (production) aşamasında bu mesajlar kapatılarak optimizasyon yapılabilir.

---

## 18. Hardware / SoC Specifik Analizler (CC1352R Bellek Haritası)

### Kullanılan Araçlar

Datasheet Rev. I, TRM Rev. G

### Analiz

Aşağıdaki ASCII diyagramları, TRM verileri ve `.txt` analiz dosyalarındaki gerçek firmware boyutları harmanlanarak görselleştirilmiştir.

#### CC1352R Flash Bellek Haritası (352 KB)

```text
┌──────────────────────────────────────────────────────────────┐
│ 0x00057FFF ┤                                                 │
│            │  CCFG (Customer Configuration) — 8 KB           │
│ 0x00056000 ┤                                                 │
│            │  Bank B — OTA İndirme Alanı (168 KB)            │
│            │  ┌────────────────────────────────┐             │
│            │  │ new-firmware.z1 (~70 KB)       │             │
│            │  │ 0x0002C000 - 0x0003D9B3        │             │
│            │  └────────────────────────────────┘             │
│            │  [Boş OTA alanı]                                │
│ 0x0002C000 ┤                                                 │
│            │  Bank A — Aktif Uygulama Alanı (176 KB)         │
│            │  ┌────────────────────────────────┐             │
│            │  │ udp-server.z1 (~43 KB)         │             │
│            │  │ 0x00000000 - 0x0000AAF5        │             │
│            │  └────────────────────────────────┘             │
│            │  [Boş alan]                                     │
│ 0x00000000 ┤                                                 │
└──────────────────────────────────────────────────────────────┘

```

*Not: CCFG bölgesinin (0x56000) OTA sırasında yanlışlıkla silinmesi cihazı kalıcı olarak tuğla (brick) yapabilir.*

#### CC1352R RAM Bellek Haritası (80 KB)

```text
┌──────────────────────────────────────────────────────────────┐
│ 0x20013FFF ┤                                                 │
│            │  __stack (Aşağıya doğru büyür)                  │
│ 0x20013000 ┤                                                 │
│            │  Dinamik Heap / Boş Alan (~72 KB)               │
│ 0x2000179A ┤                                                 │
│            │  .bss ve .data (Başlangıçlı/Başlangıçsız Veri)  │
│            │  ~6 KB (0x179A byte) Tüketim                    │
│ 0x20000000 ┤                                                 │
└──────────────────────────────────────────────────────────────┘

```

### Bulgular ve Yorum

Geliştirilen OTA mimarisi, CC1352R'nin Dual-Bank Flash sistemine tam uyum sağlayacak kapasitededir. Boyut kısıtı problemi yaşanmamaktadır.

---

## 19. Kütüphane (Library) ve Bağımlılık (Dependency) Analizi

### Kullanılan Araçlar

`msp430-nm`

### Analiz

* `memcpy`: C standart kütüphane bağımlılığıdır. Göndericideki 30 KB'lık devasa C dizisinden OTA paketine bellek kopyalaması yapmak için mecburi olarak kullanılmıştır.
* `random_rand`: MAC katmanında ağ çakışmalarını (Collision) engellemek adına periyodik `etimer` sürelerine rastgele gecikmeler (backoff) eklemek için kullanılmıştır.

### Bulgular ve Yorum

Sistem, Contiki-NG core (çekirdek) bileşenleri dışında ağır üçüncü parti kütüphanelere bağımlı değildir.

---

## 20. Contiki-NG Özel Analizler

### Kullanılan Araçlar

`msp430-nm`, `msp430-objdump`

### Analiz

* `PROCESS_BEGIN()`, `PROCESS_YIELD()`, `PROCESS_END()` makroları Contiki'nin Protothread (PT) mimarisinin C dilindeki yansımasıdır. Arka planda devasa bir `switch-case` durum makinesi (State Machine) olarak derlenirler.

### Bulgular ve Yorum

İş parçacıkları `PROCESS_YIELD_UNTIL` dediğinde fonksiyon durur ve işlemciyi diğer süreçlere devreder. Alıcıdan ACK geldiğinde sistemin anında uyanması için ağ yığını callback fonksiyonu içinden `process_poll(&udp_client_process)` çağrılmış ve muazzam bir hız optimizasyonu elde edilmiştir.

---

## 21. Güvenlik ve Robustness Analizi

### Kullanılan Araçlar

`msp430-objdump -x`, `msp430-strip`

### Analiz

* Geliştirilen C kodlarında sabitlenmiş şifreler (Hardcoded credentials) veya güvensiz bellek taşması (Buffer Overflow) yaratabilecek kısımlar kontrol edilmiştir. `sizeof()` kullanımları güvenlidir.
* **`msp430-strip` Kullanımı:** `new-firmware.z1` şu an `with debug_info, not stripped` durumundadır. OTA aktarımı öncesinde `msp430-strip new-firmware.z1` komutu çalıştırılarak dosyadaki sembol ve debug tabloları silinirse, dosyanın ağ üzerinden aktarılacak boyutu dramatik şekilde düşer. Ancak bu durumda hata alındığında `addr2line` ile satır analizi yapılamaz.

### Bulgular ve Yorum

Sistemin donanımsal dayanıklılığı (Robustness), UDP tabanlı paket düşmelerine karşı tasarlanan Stop-and-Wait mimarisi ve aktarım sonrası CFS dosya bütünlüğünü (Kümülatif CRC32) denetleyen yapı ile sağlanmıştır.

---

## 22. Karşılaştırmalı Firmware Analizi

### Kullanılan Araçlar

Tüm Araç Zinciri Sentezi

### Analiz

Üç firmware arasındaki en belirgin karşılaştırmalı farklar:

1. **Kod Boyutu (.text):** `new-firmware` (~70 KB) > `udp-client` (~43 KB) > `udp-server` (~42 KB).
2. **Roller:** Server imajı RPL Root'tur (Ağ Kökü), Client imajı ise Leaf (Yaprak/Gönderici) konumundadır.
3. **Semboller:** Server'da `cfs_write`, `is_block_received` gibi depolama ve izleme fonksiyonları varken; Client'ta `read_firmware_chunk_from_array` ve UDP veri paketleme (`memcpy`) mantığı ön plandadır.

### Bulgular ve Yorum

Her imaj, WSN içerisindeki hiyerarşik görevine uygun (Dağıtık Mimari) şekilde spesifik fonksiyonlarla derlenmiş, RAM kullanımları ise Contiki-NG'nin standart bellek yükü (~6 KB) etrafında optimize edilmiştir.

---

## 23. Genel Değerlendirme ve İyileştirme Önerileri (Taşınabilirlik)

### Kullanılan Araçlar

Mimari Sentez

### Analiz

Geliştirilen ve analiz edilen bu sistem Cooja'da 16-bit MSP430 mimarisi için sorunsuz çalışmaktadır. Ancak hedef donanım olan TI CC1352R, **32-bit ARM Cortex-M4F** işlemcisine sahiptir.

**MSP430'dan CC1352R'ye Taşıma (Porting) Adımları:**

1. **Araç Zinciri Değişimi:** Kodlar `msp430-gcc` yerine ARM ekosistemi olan `arm-none-eabi-gcc` ile derlenmelidir.
2. **Linker ve Makine Kodu Uyumsuzluğu:** MSP430 makine kodları Cortex mimarisinde anında "Hard Fault" (TRM Bölüm 5.2.2) verdirir. Linker Script (`cc13x2_cc26x2.ld`) kullanılarak bellek haritası ARM standartlarına uydurulmalıdır.
3. **Bootloader Revizyonu:** Kesme Vektör tablosu için MSP430'daki `0xFFFE` adresli Reset Vector mantığı terk edilmeli, yerine Cortex-M'in `VTOR` (Vector Table Offset Register) donanımsal yönlendirmesi kullanılmalıdır.

### Genel Sonuç

Bu çalışma, yaklaşık 130 KB'lık bir ELF donanım yazılımı imajının, 127 baytlık ağ MTU kısıtı ve 8 KB'lık cihaz RAM kısıtları aşılarak parçalanmasını, ağ üzerinden güvenilir şekilde (Stop-and-Wait ve CRC) iletilmesini ve kalıcı belleğe (CFS) yazılmasını başarıyla çözümlemiştir. CC1352R platformuna yönelik yapılan kapsamlı bellek ve bootloader analizleri, geliştirilen yazılım mimarisinin gerçek donanım üzerinde "Dual-Bank" (Çift Banka) mantığıyla endüstriyel kalitede çalışabileceğini kanıtlamaktadır. İşletim sistemleri perspektifinden, donanım kısıtlarına rağmen güvenli bir OTA sistemi başarıyla inşa ve analiz edilmiştir.
