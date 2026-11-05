# bldc_driver_ai – Sağlıklı ve Güvenli Sistem İçin Revizyon Planı (Rev 3.0 planı)

Tarih: 3 Kasım 2026.

Bu plan üç kaynağı birleştiriyor:
1. Dış inceleme (22 madde, "D" ile numaralı).
2. Üç ayrı alt sistem denetimi: kontrol kartı, gate sürücü/ölçüm/EESM ve güç katı/koruma ("K" ile numaralı yeni bulgular).
3. Kritik iddiaların dosyalardan doğrulanması. Netlist dökümü `C:\kw\audit\` altında; lug ve vida bakır mesafeleri PCB'den ölçüldü.

## 0. Mevcut durumun özeti

**Kart bu haliyle çalışmaz. Montaj anında kısa devre yapar. Çalıştırılsa güç katı yanar.**

* **Motor sürülemiyor.** Gate sinyalleri dsPIC'in PWM'siz RE pinlerinde (D1).
* **Gate sürücüler açılmıyor.** 12 V buck yaklaşık 6.6 V veriyor (D2). Buck 12 V'a düzeltilse bile akım sınırı nedeniyle hiccup moduna girer (K1).
* **Montajda kısa devre.** J2 (GND) lug pad'inin 0.3 mm dışında VBAT bakırı var. M3 vida başları VBAT, Faz V ve Faz W bakırına basıyor ve hepsi aynı alüminyum levhaya vidalanıyor (D6, K10).
* **Güç katı 10 kW için tasarlanmamış.**
  * PQFN 5×6 MOSFET'ler ve kaplamasız via'larla gerçekçi sürekli akım: çıplak levhada **~30 A**, fanlı soğutucuda **~65 A** (faz başına). Bu yaklaşık 1.5–3.4 kW eder.
  * Komütasyon döngüsü büyük. 100 V MOSFET'lerde 95–130 V tepe gerilimi bekleniyor (D3, D4, D10).
* **Koruma yok.** Sigorta, ters polarite koruması, ön şarj, TVS, donanımsal aşırı akım ve sıcaklık ölçümü eksik (D9, D18).

Doğru olanlar da var ve korunacak: IR2110 pin bağlantıları, USB-C CC dirençleri, UART yönleri (ADuM1201 bakırı doğru), TJA1051 bağlantısı, V_BUS bölücüsü, LDO pin düzenleri, dsPIC besleme ve ICSP pinleri, analog girişlerin gerçek AN pinlerine gitmesi.

---

## 1. Önce verilmesi gereken kararlar

Güç kartı ve gate kartlarının yerleşimi bu kararlara göre baştan değişecek. Kontrol kartı düzeltmeleri (Faz 1) bu kararlardan bağımsız.

| Karar | Seçenekler | Önerim |
|---|---|---|
| **K-A: Güç hedefi** | (a) 10 kW sürekli, (b) 5 kW sürekli / 10 kW 30 s tepe, (c) ~3 kW | **(b)**: 48 V, faz başına 100 A rms sürekli, 190 A rms ≤30 s. 10 kW sürekli, bu boyutta ve havalı soğutmayla gerçekçi değil. |
| **K-B: Katman sayısı ve akım yolları** | **KARAR VERİLDİ (3 Kasım 2026): tüm kartlar 2 katman, yüksek akım yollarına lehim takviyesi.** | Uygulama kuralları şartnamenin 7.5 maddesinde: akım yollarında maske penceresi, gerekirse bakır tel/şerit ile takviye, takviyeli yollar arası ≥ 1 mm, katman geçişleri via dizisi ya da perçin. Not: 2 katmanda B.Cu kesintisiz GND düzlemi olamayacağı için komütasyon döngüsü yarım köprü başına yerel MLCC'lerle küçültülmeli (D4). Lehim takviyesi MOSFET termal yolunu iyileştirmez (D10). |
| **K-C: MOSFET** | **KARAR VERİLDİ VE UYGULANDI (3 Kasım 2026): güç katında IRFB4110PBF (TO-220, 100 V, 4.5 mΩ), anahtar başına 2 paralel.** | Kartın altında yatık monte edilir (TO-220-3_Horizontal_TabUp, B yüzü), tab yalıtkan ped + omuzlu burç + M3 vida ile 15 mm levhaya basar; kart levhadan 4.8 mm yukarıda. EESM'de de IRFB4110PBF kullanılıyor (Q15–Q18, TO-220 dik, soğutucusuz; birkaç amperde kayıp çok düşük). Isı artık PCB via'larından değil doğrudan tabdan levhaya geçer (D10'un büyük kısmı çözülür). |
| **K-D: Akım ölçümü** | (a) Şönt (Kelvin'i düzeltilmiş), (b) İzole Hall sensörü | **(b)**: Faz hattında 3× ACS772-200B ya da LEM HTFS. Şönt, via ve VSS–COM sorunları birlikte kalkar. Şöntte kalınacaksa 4 terminalli şönt ve amplifikatör güç kartında olmalı. |
| **K-E: Gate sürücü** | (a) IR2110 kalsın + harici aşırı akım, (b) Koruma özellikli sürücü (DRV8353 vb.) | **(a)**: Pin bağlantıları doğru. Eksikler (dekuplaj, COM dönüşü, donanım kapatma) tamamlanır. (b) gate kartlarının tamamen yeniden tasarımı demek. |
| **K-F: Sistem koruması** | Sigorta, kontaktör + ön şarj, TVS, NTC, bleeder | **Hepsi eklensin.** Ayrıntılar Faz 6'da. |

---

## 2. Tüm bulgular (doğrulama durumuyla)

Önem dereceleri:
* **K** (kritik): çalışmaz, hasar görür ya da kısa devre olur.
* **B** (büyük): ciddi performans veya güvenilirlik sorunu.
* **k** (küçük).

### 2.1 Kontrol kartı

| No | Önem | Bulgu | Doğrulama | Düzeltme |
|---|---|---|---|---|
| D1 | K | Gate girişleri PWM'siz RE0–RE9 pinlerinde. CAN PWM3'ü (75/76), SD ise PWM4L'yi (74) işgal ediyor. | Netlistte doğrulandı. Pin fonksiyonları Microchip DS50003063B Tablo 2'den. | PWM1 → U (1/3), PWM2 → V (78/80), PWM3 → W (75/76), PWM4 → EESM bacak 1 (73/74), PWM5 → EESM bacak 2 (63/65). SD → pin 72 (RD2). CAN TX/RX → pin 69/68 (RD3/RD4, PPS). Pinlerin boş olduğu doğrulandı. |
| D2 | K | Buck çıkışı 0.6 × (1 + 20k/2k) = 6.6 V. R21 ve R22 ikisi de FREQ → GND. | Netlistte doğrulandı. | R97 = 0 Ω (ya da kaldır), R19 = 1.05 kΩ %1 → 12.03 V. R21'in GND ucu VBAT'a alınır (fSW ≈ 300 kHz). R21 1206 olmalı. |
| K1 | K | MIC2128 akım sınırı: R4 = 1.5k, L2 = 10 µH. 12 V'ta dalgalanma 3 A p-p olduğundan sınır neredeyse sıfır, devre hiccup yapar. Değerler datasheet'in 5 V örneğinden kopyalanmış. | R4 ve L2 değerleri doğrulandı. Hesap datasheet §4.3'e göre. | L2 = 33 µH ekranlı (Isat ≥ 4.5 A), R4 = 2.7 kΩ. K2'deki FET değişiminden sonra yeniden hesaplanacak. |
| K2 | B | Q13/Q14 CSD19538Q2: Vth en fazla 3.8 V, oysa sürücü yalnızca ~5 V veriyor. | Datasheet'ten. | Rds(on) değeri 4.5 V'ta garanti edilen 100 V lojik seviye FET (ör. BSZ150N10LS3 sınıfı). |
| D5 | K | C9 (VBAT) ve C11 (SW düğümü) 50 V'luk. | Doğrulandı. | 100 V X7R kullanılacak (0805/1206). |
| K3 | B | Ripple enjeksiyonu 177 mV, önerilen aralık 20–100 mV. | Hesap (datasheet §5.8). | R5 = 100 kΩ. |
| K4 | B | Faz V akımının karşılaştırıcı girişi yok. IF ölçümü W ile aynı karşılaştırıcıyı paylaşıyor. | Pin tablosundan. | IN181_Vout ayrıca pin 33'e (CMP2A) bağlanacak. CMP1/2/3 → PWM PCI fault. Akım ölçümü Hall sensörüne geçerse yeniden atanacak. |
| K5 | B | LDO ısısı: 12 V'tan 3.3 V'a lineer düşürme, U8 (3V3_IO) ~1.3 W'a kadar ısınabilir. | Hesap. | 12 V → 5 V küçük buck ön regülatör ekle; LDO'lar 5 V'tan beslensin. |
| D15 | B | Y3 kristali OSC pinlerine ~13 mm uzakta. VDD pin 12'nin yakınında kapasitör yok. | Dış inceleme (yerleşim). | Kristal ve yük kapasitörleri OSC pinlerine en fazla 3–5 mm, her VDD pinine 100 nF en fazla 2 mm. |
| D19 | B | CAN'da 120 Ω sonlandırma, ESD koruması ve J8'de GND pini yok. | Doğrulandı. | J8 3 pinli olacak (CANH, CANL, GND). 120 Ω jumper'lı sonlandırma + PESD2CAN eklenecek. |
| K6 | k | MCLR hattında filtre kapasitörü yok. | Doğrulandı. | MCLR_SW–GND arasına 100 nF. |
| K7 | k | Kristal yükü 20 pF, oysa kristalin CL değeri 18 pF. Frekans +50…70 ppm kayar. | P/N'den (doğrulanacak). | C99/C101 ve C43/C44 = 27 pF. |
| K8 | k | Enkoder filtresi 100 Ω/20 pF, yani filtre etmiyor. ESD koruması yok. Enkoder 3.3 V ile besleniyor. | Doğrulandı. | 1 nF kapasitör + ESD dizisi. Enkoder 5 V istiyorsa 5 V besleme ve seviye uyumu. |
| K9 | k | USB VBUS'ta bulk kapasitör ve ESD koruması yok. | Doğrulandı. | 4.7 µF + USBLC6-2 (GND_PC tarafında). |
| K11 | k | 12 V hattındaki 10 µF/0603 kapasitörlerin gerilim değeri belirsiz. NCP1117 çıkış kapasitörü sınırda. | Doğrulandı. | 25 V X7R 1206. LDO çıkışlarına 22 µF. |
| K12 | k | Buck'ta giriş düşük gerilim kilidi (UVLO) yok: EN doğrudan VBAT'a bağlı. PG kullanılmıyor. | Doğrulandı. | EN bölücüsü (~30 V başlatma). PG → MCU. |
| D20 | k | ADuM1201 sembolündeki pin adları ADuM1200'e göre. **Bakır doğru, bağlantıları değiştirmeyin.** | Doğrulandı. | Yalnızca sembol pin adları düzeltilecek. GND_PC–GND boşluğu ≥ 2.5 mm (fonksiyonel izolasyon). |
| K13 | k | TJA1051'in varyantı belirtilmemiş. Pin 5 VIO olarak kullanılıyor, bu yalnızca /3 varyantında var. | Doğrulandı. | BOM'da TJA1051T/3/2Z (BOM'da zaten öyle). |
| K14 | k | Q13, Q14 ve U9'da netsiz pad'ler var (EP / termal alt pad'ler). | Footprint'ten. | Land pattern'e göre D/S/GND atanacak. |

### 2.2 Gate sürücüler

| No | Önem | Bulgu | Doğrulama | Düzeltme |
|---|---|---|---|---|
| D11 + K15 | B | VCC'de yalnızca 2×1 µF var ve bunlar **GND'ye (VSS) gidiyor, COM'a değil**. LO ve bootstrap dönüş akımı şönt → güç kartı GND → riser → kontrol kartı → gate kartı yolunu izliyor: üç kart ve iki konnektör. | Doğrulandı (C41: 12V–GND; USH netinde VCC kapasitörü yok). | Her sürücüde pin 3 (VCC) ile pin 2 (COM) arasına 10 µF/25 V X7R + 100 nF. |
| D12 | B | VDD (3V3_IO) dekuplajı yok: gate kartlarında ve EESM'de. | Doğrulandı. | Her sürücüde VDD–VSS arasına 100 nF + 1 µF. |
| D13 | B | Gate pull-down'ları gate kartında. Kart çıkarsa MOSFET gate'leri yüzer. | Doğrulandı. | Her MOSFET'in dibine, güç kartına 10 kΩ G–S. Gate kartındakiler kalabilir. |
| K16 | B | VDD = 3.3 V iken VSS–COM sınırı −3.3 V. Şönt ESL'i ve zemin sıçraması bu sınırı aşabilir. | Datasheet Not 2. Büyüklüğü belirsiz. | K15 + düşük ESL şönt. Hall sensörüne geçilirse (K-D) sorun kalkar. Prototipte ölçülecek. |
| K17 | B | SD pull-up'ı 10 k ve 5 sürücüyü tutuyor. Giriş akımı 40 µA'e çıkarsa SD seviyesi VIH altına düşebilir. Ayrıca 3V3_MCU'ya bağlı, sürücü VDD'sine değil. | Doğrulandı (R57 → 3V3_MCU). | R57 = 2.2 kΩ, 3V3_IO'ya. |
| D14 | B | Gate kartındaki GND dökümü yüksek taraf bölgesine (VB/HO/faz) giriyor. | Dış inceleme. | Yüksek taraf bölgesinde döküm olmayacak. Faz düğümüyle düşük taraf arasında ≥ 1 mm boşluk. |
| K18 | k | Bootstrap diyotları SS18 (80 V). Diyot VBAT + 12 V ≈ 66 V görüyor. | Doğrulandı. | ES1D/US1D (200 V ultra hızlı) ya da STPS1H100. |
| K19 | k | Bootstrap kapasitörü 10 µF/0603. DC bias altında kapasitesi %20–40'a düşüyor. | Doğrulandı. | 25 V X7R 1206 (2.2–4.7 µF yeterli, Qg'ye göre). |
| K20 | k | Kapatma diyotlarında seri direnç yok. Paralel gate'ler sönümsüz bağlı. | Doğrulandı. | Her diyota seri 2.2–3.3 Ω. |
| K21 | B | Ölçüm çiftleri (U_SNS_P/N) gate kartının üstünden geçiyor. J15 ve J28'in ikisi de dişi soket, mekanik olarak eşleşmiyor. | Doğrulandı. | Ölçüm sinyali gate kartına uğramadan doğrudan kontrol kartına gidecek. Hall seçilirse amplifikatör güç kartında olacak. |

### 2.3 Akım ölçümü

| No | Önem | Bulgu | Düzeltme |
|---|---|---|---|
| D7 | B | Net-tie'lar şönt pad'inden 4.6–6.6 mm uzakta, akım taşıyan bakırın içinde. Kelvin bağlantısı gerçek değil, hata %50–150. | Şöntte kalınırsa 4 terminalli şönt ya da net-tie'lar pad'in iç kenarına. Önerim Hall sensörü (K-D). |
| D8 | B | Şönt 3 W: 0.2 mΩ'da sınır ~122 A rms. EESM şöntü R96 0.2 mΩ, alan akımı için çözünürlüğü yetersiz. | Faz: daha yüksek güçte şönt ya da Hall. EESM: 5–10 mΩ. Ayrıca R96 alan akımını değil köprü akımını ölçüyor (K24). |
| D9 | K | Donanımsal aşırı akım kapatması yok. 10 Ω/1 nF filtre işe yaramıyor. | Karşılaştırıcı → PWM PCI fault + SD; tepki süresi < 2 µs. |
| K22 | B | Şönt endüktansı (L/R ≈ 5 µs) örnekleme anında ~12 A'lik hata yaratabiliyor. | Filtre zaman sabiti ESL/R'ye eşitlenecek (ör. 2×10 Ω + 220 nF). Ortak mod kapasitörleri. ESL üreticiden istenecek. |
| K23 | B | TLV9001, 10 Ω üzerinden 100 nF sürüyor; faz payı ~20°, kararsız olabilir (doğrulanacak). | Riso 47–100 Ω + REF düğümünden geri besleme, ya da REF2033 referans entegresi. |
| K24 | k | INA181 kazanç varyantı belirtilmemiş (A1 olmalı). Offset için kalibrasyon gerekiyor. | BOM'da INA181A1 (zaten öyle). Açılışta offset kalibrasyonu. |

### 2.4 EESM modülü

| No | Önem | Bulgu | Düzeltme |
|---|---|---|---|
| D16 | B | Primerde DC blok kapasitörü yok, trafo doyuma girer. | Primer ile seri film kapasitör (akım ve frekansa göre boyutlandırılacak). |
| D17 | B | Toprak döngüsü: GND hem J45 kablosundan hem J49'dan geliyor. IFout analog sinyali farklı toprağa göre ölçülüyor. | J49 GND tek pin ve yalnızca sinyal referansı olacak. Güç dönüşü yalnız kablodan. IFout diferansiyel olarak ölçülecek. |
| K25 | B | VCC kapasitörleri burada da GND'ye gidiyor; COM = IF_H olduğu için dönüş R96'nın üstünden geçiyor. | VCC–COM dekuplajı (K15 ile aynı). |
| K26 | k | Firmware notu: üst taraf serbest dönüşü uzun tutulursa bootstrap boşalır. | Firmware'de sınırlanacak. |

### 2.5 Güç kartı

| No | Önem | Bulgu | Düzeltme |
|---|---|---|---|
| D3 | K | Faz boyunları 4.4 mm (V, W) ve 6.7 mm (U). H6/H7 delikleri tam bu boyunlarda. Şönt dönüşü yalnızca 4 via. | Yeni güç kartı: bakır bara + kaplamalı via dizileri. Akım yolunda delik olmayacak. |
| D4 | K | Yarım köprülerin yanında seramik kapasitör yok; en yakını 16–18 mm uzakta. Döngü endüktansı 15–30 nH, tepe gerilimi 95–130 V. | Her yarım köprüde 4–6× 4.7 µF/100 V (1210), en fazla 3 mm mesafede. 2 katmanda B.Cu GND'si yarım köprü altında bölünmemeli. İsteğe bağlı RC snubber. |
| D10 | K | Termal: kaplamasız via'lar ve PQFN ile ~30–65 A sınırı. MOSFET başına 2 via var, şartname 6–8 istiyor. | K-C: TOLL kılıf + fabrika kartı + ≤ 0.15–0.2 K/W fanlı soğutucu. |
| K27 | B | Elektrolitlerin dalgalanma akımı yetersiz: 24 adet yaklaşık 31–38 A taşır, 70 A faz akımında ~45 A gerekir. 63 V değeri 54 V bara için sınırda. | ≥ 80–100 V, toplam ≥ 70 A rms dalgalanma kapasitesi: düşük ESR'li elektrolit, hibrit polimer ya da film. |
| D6 + K10 | K | J2 VBAT bakırı pad'den 0.3 mm uzakta. M3 vida başları VBAT ve faz bakırına basıyor; hepsi aynı levhada. | Her bağlantı elemanı çevresinde ≥ 3 mm bakırsız alan. Naylon vida ya da yalıtkan omuzlu pul. Lug altında levhada yalıtımlı cep. Lug yerine pres-fit bara terminali (ör. Würth REDCUBE). |
| K28 | B | VBAT–GND arası 0.4 mm. Kaplamasız (maskesiz) kartta yetersiz. | ≥ 0.6 mm (IPC-2221B B2), maske zorunlu. |
| D22 | B | Ev üretimine uygun değil. | K-B: fabrika üretimi. |

### 2.6 Sistem koruması (hepsi eksik)

Sigorta, ters polarite koruması, ön şarj (11.3 mF), bara TVS'i, bleeder (şu an τ ≈ 20 dk), sıcaklık ölçümü (NTC yok), donanımsal aşırı/düşük gerilim koruması. Ayrıntılar Faz 6'da.

### 2.7 Proje dosyaları

| No | Bulgu | Düzeltme |
|---|---|---|
| D21 | `.kicad_dru`, `sym-lib-table` ve `bldc_proje.kicad_sym` `yardımcı_dokuman/` klasöründe. Güç kartı DRC kuralları uygulanmıyor, dsPIC sembolü bulunamıyor. | Proje köküne geri alınacak. |
| K29 | Şartnamedeki pin tablosu ("RE0/PWM1L") ve referans numaraları yanlış. Bu belge firmware için kaynak olarak kullanılmamalı. | Şartname Rev 3.0 olarak güncellenecek. |

---

## 2.8 Rev 3.0'da yapılanlar (3 Kasım 2026)

* **D21:** `bldc_proje.kicad_sym`, `sym-lib-table` ve `bldc_driver_ai.kicad_dru` proje köküne kopyalandı (asılları `yardımcı_dokuman/` içinde de duruyor).
* **Şematik:** Q1–Q12, `bldc_proje:IRFB4110PBF` sembolüne çevrildi. Pin numaraları TO-220'ye göre: G = 1, D = 2, S = 3. Pin konumları eski sembolle aynı tutulduğu için etiketlerin hepsi bağlı kaldı. ERC 0 hata; netlistte her MOSFET'in G/D/S netleri doğrulandı.
* **PCB (güç kartı yerleşimi):**
  * MOSFET'ler iki sıra halinde kartın altında. Yüksek taraf gövdeleri yukarı, alçak taraf gövdeleri aşağı bakıyor; bacaklar ortada karşı karşıya.
  * Elektrolitler sıkıştırıldı (4×6 ızgara). 2.2 µF seramikler yüksek taraf drain'lerinin hemen yanına alındı.
  * Şöntler, net-tie'lar, faz lug'ları ve H1–H8 delikleri yeni yerlerinde. Net-tie'lar şönt pad'inin üstünde (D7 kısmen).
  * Eski güç kartı bakırı (yol, via, döküm, maske açıklıkları) silindi. **Güç kartı henüz yönlendirilmedi** (151 bağlantısız), bu iş Faz 3'te yapılacak.
* **DRC kuralları:**
  * Döküm ve iz boşluğu 0.6 mm.
  * Vida delikleri çevresinde 2.5 mm bakırsız alan (D6/K10).
  * M5 lug'lar çevresinde farklı nete 3.5 mm.
  * İz ≥ 1.5 mm, via ≥ 1.0/1.8 mm.
* **DRC sonucu:** 0 hata, parite 0. Diğer modüllerde bağlantısızlık yok.
* **EESM:**
  * Q15–Q18 de IRFB4110PBF yapıldı (TO-220-3_Vertical, üst yüzde dik).
  * PQFN pinleri için çizilmiş primer_1/2 ve IF_H bakır alanları kaldırıldı. H-köprüsü netleri (VBAT, primer_1/2, IF_H ve 4 gate) 2.0–2.5 mm izlerle yeniden yönlendirildi; SD bir kez yer değiştirdi.
  * R96 yanında bir via düzeltildi. 45° kontrolü yapıldı.
  * EESM'de bağlantısız yok, DRC 0 hata.
* **3B montaj:** Kartlar levhadan 4.8 mm yukarı alındı; TO-220 tabları levhaya basıyor.
* **BOM:** Güncellendi. Mekanik sayfasına 12× TO-220 yalıtkan ped, burç ve M3 vida, levha delikleri ve lug cepleri eklendi.

**Yerleşimden doğan ve Faz 3'te çözülecek konu:**
* Yüksek taraf MOSFET'lerinde G pini sol uçta, gate konnektörleri (J26/J29/J32) ise sağda. Gate izleri alt katta GND dönüş yolunu kesecek.
* Faz 3'te bu izler, faz başına sol kenarda bırakılan GND kanalı korunacak şekilde yönlendirilecek. Gerekirse gate kartı konnektör konumu da değiştirilecek.

## 2.9 Faz 1 – Kontrol kartı şematiği: yapılanlar (3 Kasım 2026)

`kontrol_karti.kicad_sch` güncellendi. Sayfa A1'den A0'a büyütüldü; yeni parçalar sağdaki boş alanda. Önceki hali: `yedek_rev2.5/kontrol_karti_faz0.kicad_sch`.

**Doğrulama:**
* ERC 0 hata. 4 uyarı var; hepsi TJA1051 kütüphane sembolünden geliyor ve önceden de vardı.
* Netlistte 58 beklenen pin–net ataması tek tek kontrol edildi, uyumsuzluk 0.
* Eski RE pinleri (2, 4, 17, 19, 22, 24, 37, 39, 42, 44) artık boşta ve no-connect işaretli.

**PCB henüz bu şematikle senkron değil.** Yeni parçaların yerleşimi ve kontrol kartının yeniden yönlendirilmesi Faz 6'da yapılacak.

### dsPIC pin ataması (Microchip DS50003063B, 80 pin)

| Sinyal | Eski pin | Yeni pin | Yeni pin fonksiyonu |
|---|---|---|---|
| G_UH_IN / G_UL_IN | 4 / 2 (RE1/RE0, GPIO) | **1 / 3** | PWM1H / PWM1L (RB14 / RB15) |
| G_VH_IN / G_VL_IN | 19 / 17 | **78 / 80** | PWM2H / PWM2L (RB12 / RB13) |
| G_WH_IN / G_WL_IN | 24 / 22 | **75 / 76** | PWM3H / PWM3L (RB10 / RB11) |
| primer_1_H / _L | 39 / 37 | **73 / 74** | PWM4H / PWM4L (RD1 / RD0) |
| primer_2_H / _L | 44 / 42 | **63 / 65** | PWM5H / PWM5L (RC4 / RC5) |
| SD | 74 | **72** | RD2 / RP66 |
| CAN TX / RX | 75 / 76 | **69 / 68** | RD3/RP67, RD4/RP68 (PPS ile CAN) |
| IN181_Vout (V akımı) | 18 | 18 + **33** | ANA1 (ADC çekirdek 1) + CMP2A (karşılaştırıcı) |
| BUCK_PG | – | **66** | RC10 / RP58 (giriş) |

Firmware: `JTAGEN = OFF` zorunlu (TDI/TDO/TMS/TCK pinleri PWM2H, W akımı ve PWM3 olarak kullanılıyor).

### MIC2128 buck hesap tablosu (VIN 48 V nominal, 36–58.8 V)

| Büyüklük | Formül / değer | Sonuç |
|---|---|---|
| Çıkış gerilimi | 0.6 V × (1 + R18 / (R19 + R97)) = 0.6 × (1 + 20k / 1.05k) | **12.03 V** (eskisi 6.6 V) |
| Anahtarlama frekansı | 800 kHz × R22 / (R21 + R22) = 800k × 60k / 160k (R21 artık VIN'de) | **300 kHz** |
| Bobin dalgalanması | (VIN − VO) · D / (fSW · L), L = 33 µH | 0.91 A p-p (48 V), 0.94 A (54 V) |
| Akım sınırı (tipik) | (R4 · 96 µA) / Rds(on) − ΔI/2, R4 = 5.6k, Rds ≈ 0.20 Ω | ≈ 2.2 A (tepe ≈ 2.7 A) |
| Akım sınırı (en kötü) | (5.6k · 80 µA − 15 mV) / 0.33 Ω − ΔI/2 | ≈ 0.84 A (12 V yükü ≈ 0.2–0.4 A) ✓ |
| Ripple enjeksiyonu | VO · (1 − D) / (C12 · R5 · fSW) = 12 · 0.75 / (4.7 nF · 100k · 300 kHz) | **64 mV** (hedef 20–100 mV) ✓ |
| Başlama gerilimi (EN) | V_EN,high(min) · (R98 + R99) / R99 = 1.6 V · 497 / 27 (R99 = 27k, Faz 7) | garanti ≈ 29.5 V (MIC2128 EN yüksek eşiği en az 1.6 V) |
| FET'ler | IRLML0100TRPBF (100 V, SOT-23, Rds(on) 4.5 V'ta tanımlı, Qg ≈ 2.5 nC) | İletim kaybı < 60 mW |
| Bobin | 33 µH ekranlı, Isat ≥ 3 A (Bourns SRR1260-330M) | Isat **doğrulanacak** |
| VBAT / SW kapasitörleri | C9, C11: 100 nF / 100 V X7R 0805 (CL21B104KCFNNNE) | ✓ |
| R21 (VIN'e bağlı) | 100k, 1206 (çalışma gerilimi ≥ 200 V) | ✓ |

### Diğer Faz 1 değişiklikleri

* **K5, 5 V ön regülatör:** U25 TRACO TSR 1-2450 (12 V → 5V_SYS, 1 A, 7805 uyumlu SIP-3) ve C146 22 µF eklendi. U8, U19 ve U20 artık 5V_SYS'ten besleniyor. LDO kaybı ≤ 0.26 W (eskiden 1.3 W'a kadar).
* **K11:** C18/C91/C93 → 10 µF/25 V 1206. LDO çıkışları C19/C92/C94 → 22 µF/10 V 0805.
* **K7:** Kristal yük kapasitörleri C43/C44/C99/C101 → 27 pF.
* **K8:** Enkoder filtresi C132–C134 → 1 nF (τ = 100 ns). ESD dizisi U23 SP0503BAHT eklendi (A/B/Z soket tarafı).
* **K6:** MCLR filtresi C144 100 nF eklendi.
* **K9:** USB'ye U24 USBLC6-2SC6 (D+/D−/VBUS, GND_PC tarafı) ve C145 4.7 µF VBUS eklendi.
* **D19:** J8 3 pinli (CANL, CANH, GND). R101 120 Ω + JP1 lehim köprüsü ile sonlandırma; D37 NUP2105L CAN ESD.
* **K17:** SD pull-up R57 → 2.2 kΩ ve 3V3_IO'ya bağlandı (sürücülerin VDD'si).
* **D20:** ADuM1201 sembol pin adları düzeltildi (2 = VOA, 3 = VIB, 6 = VOB, 7 = VIA). Bakır aynı. Not: kullanıcı kütüphanesindeki ADuM1201 sembolü de düzeltilmezse "sembolü güncelle" işlemi eski adları geri getirir.
* **Açık kalan:** K14 (netsiz EP pad'leri) PCB işi, Faz 6'da yapılacak. K1/K2'deki FET değişimi footprint'i değiştirdiği için yerleşim de Faz 6'da.

## 2.10 Faz 2 – Gate sürücü şematiği: yapılanlar (3 Kasım 2026)

Güncellenen sayfalar:
* `gate_driver_u/v/w.kicad_sch` (A4'ten A3'e büyütüldü)
* `eesm_modulu.kicad_sch` (A2'den A1'e büyütüldü)
* `guc_karti.kicad_sch`

Önceki halleri `yedek_rev2.5/*_faz1.kicad_sch` olarak saklandı.

**Doğrulama:**
* ERC 0 hata (yalnız önceki 4 TJA1051 uyarısı).
* Netlistte her sürücünün VCC–COM kapasitörleri, turn-off dirençleri ve güç kartı pull-down'ları tek tek kontrol edildi.

**PCB henüz senkron değil (Faz 6).**

**Boyutlandırma (IRFB4110, Qg en fazla 210 nC; yüksek tarafta 2 paralel = 420 nC):**

| Parça | Yeni değer | Gerekçe |
|---|---|---|
| Bootstrap kapasitörü (C3, C4, C7, C79, C83) | 2.2 µF / 25 V X7R 1206 (CL31B225KAHNNNE) | 12 V'ta efektif ≈ 1.5 µF; anahtarlamada düşüş ≈ 420 nC / 1.5 µF ≈ 0.28 V |
| Bootstrap diyodu (D1, D2, D13, D32, D18) | US1D (200 V, 1 A, ultra hızlı) | SS18'in 80 V'u VBAT + 12 V ≈ 66 V'a karşı yetersiz paydı; Schottky sızıntısı da yok |
| VCC–COM (her IR2110 için yeni) | 22 µF / 25 V X7R 1206 + 100 nF, **pin 3 (VCC) ile pin 2 (COM) arasında** | LO ve bootstrap şarj akımı artık yerel döngüde kapanıyor; efektif VCC/Cboot ≈ 8× |
| VDD (her IR2110 için yeni) | 100 nF + 1 µF, 3V3_IO–GND | D12 |
| Turn-off diyotları (D10, D23–D31, D33–D36) | Her birine seri 2.2 Ω (0603) | Paralel gate'ler sönümleniyor; kapama direnci 10 Ω ∥ 2.2 Ω ≈ 1.8 Ω |
| Güç kartı gate pull-down'ları (yeni, R118–R129) | 10 kΩ, her MOSFET'in G–S'si | Gate kartı takılı değilken gate'ler yüzmüyor (D13) |

EESM'de U15/U16 için aynı düzeltmeler yapıldı; burada COM = IF_H (K25).

**Açık kalanlar (sonraki fazlar):**
* D14: gate kartında yüksek taraf bölgesinden GND dökümünün çekilmesi (yerleşim, Faz 6).
* K16: VSS–COM ofseti. K15 bunu iyileştirir; değer prototipte ölçülecek.
* K21: ölçüm sinyallerinin gate kartı üzerinden geçmesi. Akım ölçüm kararıyla (K-D) birlikte Faz 3'te çözülecek.

## 2.11 Faz 3 – Güç katı: kararlar, hesaplar, şematik (3 Kasım 2026)

**Kararlar:**
* **K-A:** 5 kW sürekli. 48 V'ta faz başına ≈ 100 A rms; tepe 170–190 A (süre sınırı aşağıda).
* **K-D:** Şönt ile ölçüm, düzeltilmiş.

**Şematik değişiklikleri.** Güncellenen sayfalar: `guc_karti`, `kontrol_karti` ve `eesm_modulu` (önceki halleri `yedek_rev2.5/*_faz2.kicad_sch`). ERC 0 hata; netlist kontrol edildi. **PCB senkron değil (Faz 6).**

| Konu | Değişiklik |
|---|---|
| Şönt (D7, D8) | R1–R3 → 4 terminalli 0.2 mΩ şönt, 4026 kılıf (Isabellenhütte BVR, footprint `R_Shunt_Isabellenhuette_BVR4026`). Ölçüm pad'leri (2, 3) doğrudan U/V/W_SNS_P/N'e gidiyor. Kelvin artık şöntün içinde; **NT1–NT6 kaldırıldı.** Parça numarası ve 5 W değeri datasheet'ten **doğrulanacak** (Bourns CSS4J-4026 muadil). |
| Şönt filtresi (K22) | C23/C26/C33 → 220 nF; (10 Ω + 10 Ω) · 220 nF = 4.4 µs ≈ şöntün L/R'si. ESL ölçülünce ayarlanacak. Her girişe 1 nF ortak mod kapasitörü (C173–C178). |
| REF tamponu (K23) | C21/C25/C32/C84: 100 nF → 1 nF C0G (TLV9001 10 Ω üzerinden 100 nF sürerken faz payı ≈ 20° idi). |
| Yerel seramikler (D4) | C52–C75 → 4.7 µF / 100 V X7R 1210 (GRM32ER72A475KA12L). Yüksek taraf drain'lerinin yanında, yarım köprü başına 8 adet. |
| Snubber (D4) | Her anahtara RC: 4.7 nF/100 V C0G 1206 + 2.2 Ω 2010 (C167–C172, R130–R135). Kaybı ≈ 0.27 W/adet. Önce takılı test edilecek, ringing ölçümüne göre değerler ayarlanacak. |
| Bara kapasitörleri (K27) | C108–C131 → 100 V, 12.5×20 mm, düşük ESR (≥ 1.8 A rms @ 100 kHz/105 °C; örn. Chemi-Con KZN). 25 mm yükseklik kontrol kartına çarptığı için 20 mm'de kalındı. |

### Kayıp ve termal hesap (IRFB4110PBF, anahtar başına 2 paralel, 20 kHz, 54 V)

Varsayımlar:
* Rds(on) en fazla 4.5 mΩ @25 °C; 125 °C'de ≈ 7.7 mΩ.
* Rth(j-c) 0.40 K/W; TO-220 yalıtkan ped Rth(c-s) ≈ 0.7 K/W.
* Anahtarlama kenarı ≈ 60 ns (IR2110, 10 Ω).

| Büyüklük | 100 A rms (sürekli) | 170 A rms | 190 A rms |
|---|---|---|---|
| MOSFET başına iletim kaybı | (100/√2/2)² · 7.7 mΩ ≈ 9.6 W | ≈ 27.7 W | ≈ 34.6 W |
| Yüksek taraf anahtarlama + Qrr | ≈ 3.2 W | ≈ 5 W | ≈ 5.5 W |
| Toplam kayıp (12 MOSFET) | ≈ 140 W | ≈ 380 W | ≈ 460 W |
| Tj − Tsoğutucu (en sıcak MOSFET) | ≈ 14 K | ≈ 36 K | ≈ 44 K |
| Şönt kaybı (rms ≈ I/√2) | 1.0 W (5 W anma) | 2.9 W | 3.6 W |

* **Soğutucu gereksinimi:**
  * Sürekli 5 kW için Tamb 40 °C ve Tj ≤ 125 °C koşulunda Rth(s-a) ≤ 0.45 K/W. Hedef **≤ 0.3 K/W**: Tsoğutucu ≈ 85 °C, Tj ≈ 100 °C.
  * **15 mm düz levha doğal soğumada ≈ 2–3 K/W'tır; yetmez.** Kanatlı + fanlı soğutucu gerekir (ya da levha fanlı bir kanatlı soğutucuya bağlanır).
* **Tepe süre sınırı:**
  * Levhanın ısıl kapasitesi ≈ 440 J/K (100×120×15 mm alüminyum).
  * 170 A için ≤ 30 s: Tj ≈ 135 °C.
  * 190 A için ≤ 10 s.
  * Firmware'de I²t sınırı ile uygulanacak.
* **Bakır:**
  * 100 A rms'i 2 oz bakır tek başına taşıyamaz (1 mm genişlik ≈ 0.07 mm²).
  * VBAT, GND ve faz yollarına **toplam ≥ 6–10 mm² bakır tel/şerit** lehimlenmeli (şartname 7.5).
  * Kart üstü yollar bu takviyenin taşıyıcısı ve bağlantı yeri olarak tasarlanacak.
* **Bara dalgalanma akımı (açık konu):**
  * 5 kW'ta gereken ≈ 0.6 · 100 A ≈ **60 A rms**.
  * 24 × 12.5×20 elektrolitin toplam kapasitesi ≈ 40–45 A (20 kHz'de biraz daha az). Seramikler yüksek frekans kısmını alır, ama sürekli tam güçte elektrolitler sınırın üstünde çalışır.
  * Çözüm seçenekleri:
    1. Batarya girişine kart dışı düşük ESR film/polimer kapasitör bankası (≈ 20–30 A rms ek);
    2. Kartı büyütüp 16 mm çaplı kapasitörlere geçmek;
    3. Sürekli akımı kapasitör sıcaklığına göre ≈ 70–75 A rms ile sınırlamak.
  * **KARAR VE UYGULAMA (4 Kasım 2026):**
    * Kart büyütüldü, mevcut 470 µF/63 V (PKLH-063V471MH200-T/A5.0) kapasitörlerden eklendi.
    * Güç kartı yukarı doğru 42 mm uzatıldı: 100 × 162 mm (y −42 … 120). Yeni alan üstteki kartların izdüşümü dışında.
    * 3 sıra / 18 kapasitör eklendi (C179–C196). Toplam **42 × 470 µF ≈ 19.7 mF**, dalgalanma kapasitesi ≈ 42 × 1.3–1.6 A ≈ **55–65 A** + seramik bank. 5 kW için gereken ≈ 60 A ile uyumlu; KLH'nin kesin ripple değeri datasheet'ten **doğrulanacak**.
    * Faz 3'te yazılan "100 V 180 µF" değişikliği geri alındı (C108–C131 tekrar 470 µF/63 V).
    * 63 V anma, 54.6 V (13S) bara için sınırlı pay demek. Rejeneratif frenleme sınırı, aşırı gerilim kapatması ve bara TVS'i Faz 5'te zorunlu.
    * H1 ve H4 yeni üst köşelere taşındı. 15 mm levha da 100 × 162 mm olmalı. 3B montaj güncellendi.
    * DRC 0 hata. Güç kartı yönlendirmesi Faz 6'da.

## 2.12 Faz 4 (EESM) ve Faz 5 (koruma): yapılanlar (4 Kasım 2026)

Güncellenen sayfalar: `eesm_modulu`, `guc_karti` ve `kontrol_karti` (önceki halleri `yedek_rev2.5/*_faz4oncesi.kicad_sch`).

**Doğrulama:**
* ERC 0 hata.
* Netlist farkı tek tek kontrol edildi: yalnızca aşağıdaki 10 pin değişti, yeni parçalar beklenen netlerde, kopan ya da birleşen net yok.

**PCB senkron değil (Faz 6).**

### Faz 4 – EESM

| Bulgu | Değişiklik |
|---|---|
| D8 | R96: 0.2 mΩ → **5 mΩ 2512 ≥ 3 W** (Bourns CSS2H-2512R-5L00F, doğrulanacak). INA181A1 ile 0.1 V/A, ±16.5 A tam skala. 10 A'de kayıp 0.5 W. |
| D16 | Döner trafo primerine seri DC blok: C197 + C198 = 2 × 4.7 µF/100 V X7R 1210 (primer_1 → J6, yeni net PRIMER_1_TR). Değer, trafonun kaçak endüktansı ve anahtarlama frekansıyla rezonansa göre ayarlanacak. |
| D17 | Köprünün güç toprağı ayrıldı. **PGND_EESM** netinde: J45 kablosu, C140–C142 ve R96 GND ucu (NT8 Kelvin). Lojik GND ile tek noktada birleşiyor: R96 yanında NT9 net-tie. J49 başlığındaki GND pinleri artık yalnız lojik/sinyal referansı. |
| m5 | R96 köprünün DC akımını ölçer, alan akımını değil. Alan akımı firmware'de trafo modeliyle kestirilecek (not). |

### Faz 5 – Koruma

**Kart üstü:**

| Yer | Parça | Gerekçe |
|---|---|---|
| Güç kartı | D38, D39: 5.0SMDJ58CA (5 kW, çift yönlü, SMC), VBAT–GND | Bara darbe koruması. 58 V tutma, 64.4 V kırılma. **Not:** yüksek akımda kıskaç ≈ 94 V; 100 V MOSFET'ler için sınırda, 63 V kapasitörlerin darbe değerinin üstünde. Asıl koruma firmware'deki aşırı gerilim kapatması + rejeneratif frenleme sınırı. |
| Güç kartı | R136–R138: 3 × 10 kΩ 2512 (1 W), VBAT–GND | Bleeder: 3.3 kΩ ∥, τ = 3.3 kΩ · 19.7 mF ≈ 66 s; toplam kayıp 0.9 W (her direnç 0.3 W). |
| Kontrol kartı | Q19/Q20 AO3400A (30 V, SOT-23) + R139–R142 (100 Ω gate, 10 kΩ pull-down) + D40/D41 SS18 flyback; J50 (COIL_12V, CONT_K, PRE_K, GND) | Kontaktör ve ön şarj rölesi sürücüleri. Bobinler **harici 12 V**'tan beslenir (EV200 sınıfı kontaktörün kalkış akımını buck karşılayamaz). MCU pinleri: CONT_EN = U6.2 (RE0), PRE_EN = U6.4 (RE1). |
| Kontrol kartı | J51 + R143 10 kΩ %0.1 (3V3_ANA'ya) + R144 1 kΩ / C199 100 nF → U6.20 (AN9) | Soğutucu sıcaklığı: levhaya vidalı 10 kΩ B3435 NTC (2 tel). 25 °C'de ≈ 1.65 V, 100 °C'de ≈ 0.29 V. |

**Kart dışı (BOM Mekanik sayfasına eklenecek):**

| Parça | Değer | Not |
|---|---|---|
| Ana sigorta | 200–250 A, ≥ 80 V DC, ANL / MEGA | Batarya + ucuna, kontaktörden önce |
| Ana kontaktör | ≥ 200 A sürekli, ≥ 100 V DC, 12 V bobin (TE EV200 sınıfı) | J50.CONT_K ile sürülür |
| Ön şarj rölesi + direnci | Röle ≥ 5 A / 100 V DC, 12 V bobin; direnç 10 Ω / 50 W alüminyum gövdeli | τ = 10 Ω · 19.7 mF ≈ 0.2 s; 5τ ≈ 1 s'de bara dolar. Darbe enerjisi ½CV² ≈ 29 J; ilk an gücü ≈ 300 W. |
| Polarize güç konnektörü | ≥ 200 A | Ters bağlantıyı mekanik olarak önler. Ters bağlanırsa ön şarj direnci akımı sınırlar; kontaktör kapatılmamalı. |
| NTC | 10 kΩ B3435, vidalı (M3) | Soğutucu levhaya |

**Firmware (zorunlu):**
* **Açılış sırası:**
  1. PRE_EN açılır.
  2. V_BUS üstel olarak yükselir.
  3. V_BUS batarya geriliminin %95'ine ulaşınca CONT_EN açılır, PRE_EN kapanır.
  4. Belirlenen sürede bara dolmazsa hata verilir (kısa devre / ters bağlantı).
* **Donanım kapatma:**
  * CMP1/CMP2/CMP3 (U/V/W akım) → PWM PCI fault, tepki süresi < 2 µs.
  * V_BUS > 60 V → ADC dijital karşılaştırıcı ile PWM kapatma + rejeneratif frenleme sınırı.
  * V_BUS < 36 V → kapatma.
* **Sıcaklık:** NTC 80 °C'de akım azaltma, 95 °C'de kapatma. I²t ile tepe akım sınırı (Faz 3 tablosu).

## 2.13 Faz 6 – PCB senkron, yerleşim ve yönlendirme: tamamlandı (4 Kasım 2026)

**PCB şematikle senkron:** parite 0. 99 footprint eklendi, 46 footprint değiştirildi, NT1–NT6 silindi, 388 pad neti güncellendi. Önceki PCB `yedek_rev2.5/bldc_driver_ai_faz6oncesi.kicad_pcb` olarak saklandı.

| Kart | Durum |
|---|---|
| **Güç kartı** | **Tamam.** DRC 0 hata, bağlantısız 0. Üst katta güç dökümleri: VBAT; faz şeridi + lug sütunu; xSH; GND bölgesi. Alt katta tam GND. Seramikler, pull-down'lar, snubber'lar, TVS ve bleeder'lar yerleştirildi; gate ve ölçüm izleri altta. |
| **Gate kartları U/V/W** | **Tamam.** DRC 0 hata, bağlantısız 0. Yeni VCC–COM/VDD kapasitörleri, turn-off dirençleri ve bootstrap kapasitörü alt yüzde. Yüksek taraf bölgesinde GND dökümü yok (D14, L şeklinde keep-out alanı). Kartlar sıfırdan yönlendirildi. |
| **EESM** | **Tamam.** DRC 0 hata, bağlantısız 0. Yeni parçalar yerleştirildi; NT9 alt yüzde, R96 altında. Kart sıfırdan yönlendirildi. |
| **Kontrol kartı** | **Tamam.** DRC 0 hata, bağlantısız 0. Kart 20 mm büyütüldü: 100 × 76.5 mm, yeni şerit bara kapasitörlerinin üstünde, yığında serbest. CAN, ICSP ve enkoder konnektörleri yeni kenarda; TSR, kontaktör ve NTC devreleri yeni şeritte. L2 (33 µH) alt yüzde, Q13/Q14 altında. USB-C (J23) elle açıldı: D+ çifti alt katta birleştirildi, CC1 alttan R41'e, USBLC6 (U24) geçişli. GND_PC alt dökümü U24 altına uzatıldı. U6 GND pinleri 32 ve 70 gövde altı via'larla bağlandı. G_WL_IN için G_WH_IN, primer_1_L/H, SD, BUCK_PG ve CONT_EN söküldü ve yeniden yönlendirildi. |

**Kontrol kartı GND dökümü:** döküm boşluğu 0.5 mm'den 0.25 mm'ye indirildi. 2 katlı yoğun kartta 0.5 mm, GND'yi 12 adacığa bölüyordu. VBAT (60 V) ile dökümler arası 0.5 mm, `.kicad_dru` içindeki yeni `VBAT_dokum_0.5mm` kuralıyla korunuyor. Kalan adacıklar dikiş via'larıyla ve kısa köprülerle ana GND'ye bağlandı. Kartta bağlı olmayan döküm adası yok.

**Son temizlik (tüm kartlar):**
- 372 + 9 kopya iz/via silindi.
- Aynı nette iç içe geçmiş 7 via çifti birleştirildi; DRC'de delik-delik uyarısı kalmadı.
- Boşta kalan uçlar ve via'lar silindi.
- Tüm izler 0°/45°/90°'ye çevrildi: 1176 parça "düz + 45°" kırığa dönüştürüldü. 5°'den fazla sapan iz kalmadı. Dönüşümün yarattığı 4 boşluk hatası düzeltildi; EESM IF_H pad çıkışı 2.0 mm'ye indi, hattın geri kalanı zaten 1.15 mm.
- Serigrafide görünen 92 MPN/açıklama alanı gizlendi.

**Son DRC (proje klasöründeki PCB):**
- 0 hata, 0 kart içi bağlantısız, şematik–PCB paritesi 0.
- 61 kartlar arası "bağlantısız" beklenen durum: kartlar B2B konnektörlerle birleşiyor.
- Uyarılar: 4 serigrafi–kenar mesafesi (görsel) ve J23 kütüphane farkı.

**Çıktılar:**
- `montaj/bldc_driver_ai_3d_montaj.kicad_pcb` ve `montaj/3d/*.step`: yığın montaj (alüminyum levha → güç → gate U/V/W → kontrol → EESM).
- `montaj/render_montaj_iso.png`, `montaj/render_pcb_ust.png`.
- `bldc_driver_ai_BOM.xlsx`: 472 footprint'ten yeniden üretildi; Faz 1–5 parçaları MPN'leriyle eklendi. Bu MPN'ler distribütörde doğrulanmadı; özellikle BVR şönt son eki ve SRR1260-330M doyma akımı kontrol edilmeli.
- Yedekler: `yedek_rev2.5/bldc_driver_ai_faz6ara.kicad_pcb`, `yedek_rev2.5/bldc_driver_ai_faz6oncesi.kicad_dru`.

**Sonraki adım:** Faz 7 (doğrulama): elektriksel gözden geçirme, termal kontrol ve üretici DFM kontrolü.

> **Faz 7 düzeltmesi:** Yukarıdaki "DRC 0 hata" sonucu eksikti. `.kicad_dru` dosyası UTF-8 BOM ile başladığı için KiCad güç kartının özel kurallarının hiçbirini okumamıştı (bkz. 2.14).

## 2.14 Faz 7 – Doğrulama: bulgular ve düzeltmeler (4 Kasım 2026)

### 2.14.1 Kök neden: tasarım kuralları devre dışıydı (KRİTİK, düzeltildi)
- **Ne oldu:** `bldc_driver_ai.kicad_dru` UTF-8 BOM (EF BB BF) ile başlıyordu. KiCad dosyayı hata vermeden yok saydı, kuralların hiçbiri uygulanmadı:
  - iz ≥ 1.5 mm,
  - via 1.0 / 1.8 mm,
  - 0.6 mm boşluk,
  - vida çevresi 2.5 mm,
  - pabuç çevresi 3.5 mm.
- **Sonucu:** dökümler vida deliklerine 0.3 mm'ye kadar yaklaşmıştı; üstte VBAT, altta GND. Metal pul ve ara parça VBAT ile GND'yi vida ve soğutucu üzerinden kısa devre ederdi.
- **Düzeltme:**
  - Dosya BOM'suz yazıldı.
  - Kuralların çalıştığı kasıtlı bir test iziyle doğrulandı.
  - Dökümler `kicad-cli pcb drc --refill-zones --save-board` ile kurallara göre yeniden dolduruldu.
- Kurallar devreye girince güç kartında **165 ihlal** ortaya çıktı (iz genişliği ve via kuralları zaten sağlanıyordu). Hepsi aşağıdaki gibi giderildi.

### 2.14.2 Kural güncellemeleri (kullanıcı kararı: yalıtkan kit + yeniden düzenleme)
| Kural | Önce | Şimdi | Gerekçe |
|---|---|---|---|
| H1–H8 montaj delikleri | 2.5 mm | **2.5 mm** (aynı) | Metal ara parça ve metal pul |
| Q1–Q12 TO-220 vida delikleri | 2.5 mm | **1.0 mm** | **Yalıtkan kit zorunlu:** naylon omuzlu burç + Ø7 naylon pul. Bakır naylon pulun altında kalabilir; vida gövdesi delik duvarına değebilir, bu yüzden 1.0 mm. |
| J1–J5 pabuç, üst yüz | 3.5 mm | **3.5 mm** (F.Cu) | Kablo pabucu dili üstte, dönebilir |
| J1–J5 pabuç, alt yüz | 3.5 mm | **1.0 mm** (B.Cu) | Altta yalnızca M5 somun ve pul var; ikisi de pad içinde kalıyor |
| 0.6 mm boşluk (güç kartı) | 0.6 mm | **0.6 mm** (aynı) | IPC-2221B B2 |

### 2.14.3 Yerleşim düzeltmeleri (güç kartı)
- **Gate hatları:** G_xH1/H2/L1/L2 ve xSH izleri tek bir U fazı şablonundan yeniden çizildi. V ve W fazları aynı düzenin +32 ve +64 mm ötelenmiş kopyası; footprint'ler de tam bu ötelemelerle yerleştirilmiş.
  - TO-220 bacaklarının yanından geçişler en az 0.6 mm.
  - Pull-down dirençlerine girişler komşu pad'den uzak taraftan yapılıyor.
- **Şönt Kelvin uçları:**
  - Ölçüm pad'i içinde 0.15 mm kaydırıldı.
  - P ve N köşegenleri faz pabuçlarından en az 1.0 mm, Q8/Q10/Q12 vida deliklerinden en az 1.0 mm uzakta. P–N arası 0.64 mm.
- **12 seramik bara kondansatörü:** GND çıkış izleri ve via'ları 0.37 mm kaydırıldı. 1.5 mm izin uç kapağı artık VBAT pad'ine 0.6 mm'den yakın değil.
- **J1/J2 (VBAT/GND) pabuç pad'leri:** Ø10.6'dan Ø10.4'e küçültüldü (M5 pulu Ø10). Aradaki boşluk 3.6 mm.
- **TVS D39:** 1 mm yukarı taşındı; GND izi ve via'sı da birlikte.
- **Sonuç (kurallar devrede):** 0 hata, 0 kart içi bağlantısız, parite 0. 5°'den fazla sapan iz yok.

### 2.14.4 Bağımsız inceleme (pin denetimi + elektriksel)
Tüm entegrelerin pin → net eşleşmeleri datasheet'lerle karşılaştırıldı (dsPIC Tablo 8, MIC2128, IR2110S, INA181, TLV9001, NCP1117, TJA1051, CH340G, ADuM1201, USBLC6 ve diğerleri). **Yanlış bağlanmış pin, ters diyot ya da G/D/S hatası bulunmadı.**

| # | Bulgu | Önem | Durum |
|---|---|---|---|
| 1 | Kontrol kartı yalnızca VBAT barasından beslenir. Bara, MCU'nun kapattığı kontaktör ve ön şarj rölesinin arkasında olduğu için sistem açılamaz. | **Kritik** | **Düzeltildi:** D42 (SS34, 3 A/40 V) harici 12 V'u (J50.1) kartın 12 V rayına diyot-VEYA ile bağlar. MCU araç 12 V'undan açılır; bara dolunca buck devreye girer. Bobin/besleme izleri DRC'nin izin verdiği kadar genişletildi (çoğu 0.8–1.0 mm, darboğaz 0.3–0.4 mm). |
| 2 | MIC2128 EN bölücüsü (470k/20k) 36 V'ta 1.47 V verir; garanti eşik 1.6 V. | Büyük | **Düzeltildi:** R99 = 27k. 36 V'ta 1.96 V; garanti açılış ≈ 29.5 V. |
| 3 | NCP1117, ESR ≥ 33 mΩ ister; 22 µF seramikle salınabilir. | Büyük | **Düzeltildi:** U8/U19/U20 → LDL1117S33R, U21 → LDL1117S50R. Seramikle kararlı, aynı SOT-223 bacak düzeni. |
| 4 | LDO değer alanında gerilim yoktu; ayarlı tip takılırsa çıkış 1.25 V olurdu. | Büyük | **Düzeltildi:** değerler sabit gerilimli parça numaraları oldu. |
| 5 | 470 µF/63 V bara kondansatörlerinin 60 V'a payı %5. 5.0SMDJ58CA 64.4 V'tan önce iletmez, kıskaç ≈ 94 V. | Büyük | **Kullanıcı kararı: olduğu gibi.** 60 V sınırı firmware'de zorunlu (V_BUS aşırı gerilim kapatması + rejeneratif fren sınırı). **Açık risk:** bu sınır olmadan rejeneratif frende kondansatörler hasar görebilir. |
| 6 | "Kontaktör bobini beslemesiz" | — | **Yanlış alarm:** Faz 5'te bobinlerin J50.1'den harici 12 V ile beslenmesine bilerek karar verildi. |
| 7 | ADuM1201 sembolünde pin 2/7 adları (VIA/VOA) ters. Bağlantı doğru. | Küçük | Açık: yalnızca sembol etiketleri; kablolama değişmeyecek. |
| 8 | JTAG pinleri PWM/ADC ile ortak. | Küçük | Firmware: `JTAGEN = OFF` (bölüm 4'te zaten var). |
| 9 | Diğer: tek LDO giriş kondansatörü (C146); R66 100k 0603 60 V'ta; snubber direnç gücü (40 kHz'de 0.68 W); kristal footprint pad sayısı doğrulanmadı. | Küçük | Açık; prototipte kontrol edilecek. |

### 2.14.5 Mekanik denetim (ölçüm betiği `C:\kw\audit\mech_audit.py`, `hole_fill.py`)
- **H1–H8:** dökümün delik kenarına uzaklığı 2.55 mm (≥ 2.5). Delik çevresinde iz ya da via yok.
- **Q1–Q12:** kural 1.0 mm sağlanıyor. Pulun altındaki bakır naylon pulla yalıtılıyor (BOM Mekanik sayfasında "ZORUNLU" olarak yazıldı).
- **J1–J5:** üstte ≥ 3.5 mm, altta ≥ 1.0 mm.

### 2.14.6 Hesap özeti
| Konu | Değer |
|---|---|
| Buck çıkışı | 0.6 V × (1 + 20k/1.05k) = **12.03 V**, fSW ≈ 300 kHz |
| Buck açılışı | EN: garanti açılış ≈ 29.5 V (R98 470k / R99 27k); 60 V'ta EN 3.26 V (sınır: VIN + 0.3 V) |
| Buck akım sınırı | ≈ 1.4–2.2 A (ripple öncesi); yük gate sürücüleri + TSR + LDO'lar ≈ 0.5–0.8 A. Bobinler harici 12 V'tan beslendiği için pay yeterli. |
| LDO ısısı | (12 − 3.3) V × ~0.1 A ≈ 0.9 W, SOT-223 ve bakır alanla kabul edilebilir |
| Şönt kaybı | 0.2 mΩ × (100 A)² = 2 W (BVR 4026) |
| Bleeder | 3.3 kΩ ∥, 60 V'ta 1.08 W, τ ≈ 66 s |
| MOSFET kayıp ve Tj | Bölüm 2.11'deki hesap geçerli (yerleşim değişikliği termal yolu etkilemedi) |
| Gerilim değerleri | VBAT'a bağlı seramikler, MOSFET'ler ve IRLML0100 100 V; MIC2128 76 V. **İstisna:** bara elektrolitikleri 63 V (madde 5). |

### 2.14.7 Çıktılar
- `uretim/<modül>_gerber.zip` (6 modül): Gerber, Excellon (PTH/NPTH ayrı), delik haritası PDF; ayrıca `<modül>_pos.csv` dizgi dosyaları.
- `montaj/`: güncel STEP dosyaları ve render.
- `bldc_driver_ai_BOM.xlsx`: LDL1117, 27k, SS34 ve yalıtkan TO-220 kiti eklendi.
- **Yedekler (`yedek_faz7/`, varlığı doğrulandı):**
  - `bldc_driver_ai_faz7_kural_oncesi.kicad_pcb`: kurallar devreye girmeden önceki PCB
  - `bldc_driver_ai_faz7_D42.kicad_pcb`: D42 eklendikten sonraki PCB
  - `*_footprint_denemesi.*`: 2.14.9'daki footprint değişikliği denemesi (PCB + iki şematik sayfası)

> **Düzeltme (yedekler):** Bu belgede `yedek_rev2.5/` altında sayılan yedekler (faz0–faz7) projede yok; klasör hiç oluşmamış ya da sonradan silinmiş. Kopyalama komutlarının sonucunu o sırada doğrulamamıştım. Geçerli yedekler yalnızca `yedek_faz7/` içindekiler ve çalışma klasöründeki ara kartlar (`C:\kw\p6\*.kicad_pcb`).

### 2.14.8 Açık maddelerin kapatılması (4 Kasım 2026)
| Madde | Sonuç |
|---|---|
| ADuM1201 pin 2/7 etiketleri | **Kapandı.** Şematik sembolü zaten doğruydu (pin 2 = VOA, pin 7 = VIA). Ters görünen adlar PCB pad'lerindeki eski "pin function" etiketlerinden geliyordu; 291 pad netlist'ten yenilendi, bağlantı değişmedi. |
| IRFB4110 (dik ve yatay), AO3400A, IRLML0100, NUP2105L, TSR 1-2450, IR2110S, US1D/SS18/SS34 kutupları | **Doğru** (üretici datasheet'leri; IR2110S pin şekli resim olduğu için standart SOIC-16W pin tablosuyla). |
| USBLC6-2SC6 | Doğru (ST alıntıları ikincil kaynaklardan). |
| SP0503BAHT | Pin 1 = ortak anot (GND), pin 2–4 = hatlar. Tasarımla aynı; iki ikincil kaynak, Littelfuse PDF'i erişilemedi. |
| LDL1117S33R/S50R | Bacak düzeni NCP1117 ile aynı (1 GND, 2 OUT, 3 IN, tab OUT), giriş 2.5–18 V. Kararlılık eğrileri (Şekil 20–21) okunamadı; 22 µF / 10 µF seramik tipik uygulamanın üstünde. |
| MIC2128 4 numarasız pad | Yalnızca lehim pastası penceresi; sorun yok. |
| Kristal (Y2, Y3) | Gerçek parça 2 pad'li HC-49/US SMD 12 × 4.8 mm; footprint pad sayısı doğru, ölçüleri doğrulanmadı. BOM açıklaması düzeltildi. |
| **MPN düzeltmeleri (BOM)** | WSLP2512R0050FEA → **WSLP25125L000FEA**; GRM32ER72A475KA35L (yok) → **HMK325C7475KN-TE**; EEE-FK1J221P (yok) → **EEE-FK1J101P** (C140 değeri 100 µF/63 V oldu); CL10A106KA8NRNC → **CL10A106MA8NRNC**; SS34 (onsemi'de SMC) → **B340A-13-F** (SMA); SS18-T/R → **SS18**; PKLH-… → **KLH-063V471MH200-T/A5.0**; MCF03KTB500103 → **CL10B103KB8NNNC**; TJA1051T/3/2Z → **TJA1051T/3/1J**. SRR1260-330M: Isat 2.8 A; akım sınırının üst değeri ≈ 2.2 A olduğu için yeterli (L2 değeri "33uH" yapıldı). |

### 2.14.9 Açık: özel footprint'lerin land pattern'i (sipariş öncesi düzeltilmeli)
- **Bulgu:** INA181, TLV9001 (SOT-23), TJA1051, ADuM1201 (SOIC-8) ve CH340G (SOIC-16) özel footprint'lerinde dış pad kenarı tam nominal bacak ucunda. Lehim dolgusu (toe fillet) için yer yok.
  - SOT-23'te pad sıraları ±0.75 mm'de; TI'nin önerisi ±1.3 mm.
  - SOIC'lerde pad sıraları ±2.23 mm'de; standart değer yaklaşık ±2.5–2.7 mm.
- Pin numaraları ve yönleri doğru. MIC2128'in sinyal pad'leri de kısa: gövde dışına yalnızca 0.06 mm taşıyor.
- **Deneme:** 12 parça KiCad standart paketlerine geçirildi (SOT-23-5/6, SOIC-8/16, VQFN-16 EP1.68). Yeni pad'ler dışa doğru 0.3–0.45 mm uzandığı için yakındaki izlerle ~80 çakışma çıktı. Otomatik yerel sökme ve yeniden yönlendirme yakınsamadı; kopukluklar arttı.
- Deneme geri alındı. Proje eski footprint'lerle tutarlı ve DRC temiz durumda; deneme `yedek_faz7/*_footprint_denemesi.*` olarak saklandı.
- **Gereken:** bu 12 parçanın çevresinde elle, parça parça yeniden yerleşim ve yönlendirme. Ya da kart küçültme çalışmasıyla birlikte ele alınması.

## 2.15 Dış inceleme düzeltmeleri ve kart dışı DC-link film bankı (5 Kasım 2026)

### 2.15.1 Dış incelemenin bulguları (hepsi netlist/PCB üzerinde doğrulandı)
| # | Bulgu | Durum |
|---|---|---|
| K1 | **Şönt ölçüm hattı mekanik olarak kopuk** (K21). Güç kartı J28/J31/J34 dişi soket, gate kartı J15/J16/J25 de düz dişi soketti; gate kartının altında erkek pin yoktu. Sonuç: INA181 girişleri boşta, FOC akım ölçümü ve CMP → PCI aşırı akım kapatması çalışmaz. | **Düzeltildi:** J15/J16/J25 **stacking (uzun kuyruklu) soket** oldu (üstte 8.5 mm soket, altta ≥ 10.5 mm kuyruk; kart arası 11 mm → J28'e ≈ 6 mm girer). Footprint aynı; BOM'da ayrı satır, 3D modelde kuyruk görünür. |
| O2 | **EESM toprak döngüsü:** PGND_EESM hem J45 kablosuyla güç toprağına hem NT9 üzerinden lojik GND'ye bağlıydı; köprü akımının bir kısmı J49'un 2.54 mm pinlerinden dönüyordu. | **Düzeltildi:** NT9 kaldırıldı. Köprü dönüşü yalnızca J45 kablosundan, lojik referans yalnızca J49 GND. IR2110 COM = IF_H ve INA181 diferansiyel olduğu için ölçüm etkilenmez. |
| O3 | **Harici 12 V girişi (J50.1) korumasız.** | **Düzeltildi:** D43 SMAJ15A (VRWM 15 V, kıskaç 24.4 V < IR2110 VCC mutlak sınırı 25 V) J50.1–GND; F1 1812L110/33MR PTC (1.1 A) kart besleme yolunda, D42 öncesi (yeni net 12V_EXT). Bobin/besleme izleri DRC'nin izin verdiği kadar genişletildi (çoğu 0.8–1.0 mm, darboğaz 0.3–0.4 mm). 3V3 LDO'lar zaten TSR 1-2450 çıkışından (5V_SYS) besleniyor; yalnızca U21 (LDL1117S50, 20 V mutlak) 12 V'ta, kısa darbelerde F1 + ray kondansatörleriyle korunur. |
| D4 | **Riser J46/J47 pin boyu:** kontrol kartı altı güç kartının 23.6 mm üstünde; J46 (8.5 mm soket) için plastik altında ≈ 18 mm pin gerekir. | **Düzeltildi (BOM):** J47 = 2.54 mm 2x2 erkek başlık, toplam pin boyu ≥ 25 mm. 3D modelde pin uzatıldı. |
| — | Planın 2.9 tablosunda R99 hâlâ 20k yazıyordu (belge tutarsızlığı). | **Düzeltildi:** tabloda 27k, garanti açılış ≈ 29.5 V. |

### 2.15.2 Bara kondansatörleri: kart dışı DC-link film bankı
- **Araştırma (DigiKey, LCSC, TDK datasheet'i):** 100 V sınıfında DC-link polipropilen film yok denecek kadar az; TDK B3277x serisinin en düşük gerilimi 450 V.
  - B32774D4226K000: 22 µF, 14.5 A, 5.4 mΩ, 22 x 36.5 x 31.5 mm.
  - B32776G4506K000: 50 µF, 20 A, 4.3 mΩ, 28 x 42.5 x 42 mm.
- **Neden kart dışı:** film kondansatörler 36.5–42.5 mm yüksekliğinde. Kontrol kartının altındaki boşluk 23.6 mm, kartın serbest bölgesine de yalnızca 6 x 22 µF (132 µF) sığıyordu. Bu kadarı elektrolitiklerle paralel kalınca 20–40 kHz dalgalanmanın çoğunu yine elektrolitikler taşırdı. (Kullanıcı kararı: kart dışı bank.)
- **Seçim:** 4 x B32776G4506K000 = **200 µF / 80 A** (gereken ≈ 50 A rms; 40 kHz'te ΔV ≈ 1 V rms). J1/J2 M5 saplamalarına lamine bara çiftiyle (≤ 100 mm) bağlanır.
- **Kart:** 42 x 470 µF/63 V kaldırıldı. 26 x 4.7 µF/100 V seramik yüksek frekans dekuplajı olarak kaldı.
- **Sonuçlar:**
  - **63 V gerilim payı riski ortadan kalktı.** Firmware'deki V_BUS sınırı yine de önerilir; TVS kıskacı 93.6 V, MOSFET'ler 100 V.
  - Ön şarj τ ≈ 10 Ω x 0.26 mF ≈ 3 ms. Deşarj direnci τ ≈ 0.9 s; 1.08 W sürekli kayıp için R136–R138 büyütülebilir.
  - **Sönümleme:** düşük ESR'li film + batarya kablosu endüktansı rezonans yapabilir. Sönümleme batarya/kablo direncine kalıyor; Faz 8 adım 4'te bara geriliminde çınlama ölçülmeli. Gerekirse banka bir adet 100 V elektrolitik (ESR'si sönümleme yapar) eklenir.
- **Güç kartı 100 x 162 mm'den 100 x 93 mm'ye indi** (üst kenar y = 27). J1/J2 lug'ları, TVS D38/D39, deşarj dirençleri R136–R138 ve H1/H4/H5 montaj delikleri y 27–55 bandına taşındı. GND pad'leri 1.8/1.0 mm via ile alt dökümle birleştirildi. Yığındaki diğer kartlar değişmedi (kontrol kartı y ≥ 31.45'te).
- **Doğrulama (kurallar devrede):** DRC 0 hata, 0 kart içi bağlantısız, parite 0; izlerin tamamı 0°/45°/90°. Gerber'ler, montaj ve render güncellendi.

## 2.16 Land pattern düzeltmesi tamamlandı (5 Kasım 2026)

**Yeni footprint'ler** (`bldc_proje.pretty`, proje `fp-lib-table`): kullanılan özel footprint'lerin kopyası. Pad numarası, yönü ve orijini aynı; pad'ler yalnızca dışa uzatıldı.

| Footprint | Parçalar | Pad (önce → şimdi) | Dış kenar / bacak ucu (en büyük) |
|---|---|---|---|
| INA181_LP | U7, U12, U14, U18 | 1.10 mm @ ±0.75 → 1.15 mm @ ±1.175 | 1.75 / 1.5 mm |
| TLV9001_LP | U10, U11, U13, U17 | aynı | 1.75 / 1.5 mm |
| TJA1051_LP, ADuM1201_LP | U22, U4 | 1.55 mm @ ±2.23 → 1.8 mm @ ±2.5 | 3.4 / 3.1 mm |
| CH340G_LP | U5 | aynı, aralık tam 1.27 mm (önce 1.26–1.28) | 3.4 / 3.1 mm |
| MIC2128_LP | U9 | sinyal pad'leri 0.40 → 0.69 mm | gövdenin 0.35 mm dışı |

**Courtyard:** KiCad standardındaki gibi gövde + 0.25 mm ile pad sıraları + 0.25 mm'nin birleşimi. Komşu pasiflerle pad aralığı 0.62–1.81 mm.

**Yerleşim:** parçalar yerinde kaldı. Her parça için çevresindeki kutuda kendi netleri ve çakışan netler kesildi, dışarıdaki bakıra dokunulmadı. Ardından kutu içi pad'ler ve kesilen saplama uçları yeniden bağlandı (yeni `box` / `pt` yönlendirme görevleri). Sonuç parça başına DRC ile doğrulandı, kötüleşen parça geri alındı.
- **Otomatik çözülenler:** U7, U10, U12, U13, U18, U5.
- **Elle tamamlananlar:**
  - U4: RXD_DSP gövde altı via'sıyla; TXD via'sı 0.45 mm kaydırıldı.
  - U22: 3V3_IO koridoru 0.3 mm'ye daraltıldı; CANH, CANL ve CONT_EN yeniden yönlendirildi.
  - U9: DL düz iniş; BUCK_EN ve BUCK_FREQ yeniden yönlendirildi.
  - GND / GND_PC döküm adacıkları: dikiş via'sı, C145.2 → U24.2 bağlantısı, RXD'nin alt katman kısmı yeniden çizildi.

**Son durum (proje klasörü, kurallar devrede):**
- DRC 0 hata, 0 kart içi bağlantısız, parite 0; izlerin tamamı 0°/45°/90°.
- ERC temel durumda: 4 bilinen TJA1051 pin_to_pin uyarısı.
- Uyarılar: 5 serigrafi–kenar, 3 kütüphane farkı (J1/J2 lug pad Ø10.4, J23).
- Temizlikte 353 kopya iz ve 4 tek katmanlı (boşta) via silindi.
- **Gerber'ler artık siparişe uygun.**

## 2.17 Modül 5: DC-link film bank kartı (5 Kasım 2026)

**Neden:** 2.15'te bara kondansatörleri yalnızca BOM'un Mekanik sayfasına "kart dışı bank" olarak yazılmıştı; şematikte ve kartta yoktu. Kullanıcı kararıyla ayrı bir kart yapıldı.

- **Şematik (güç sayfası):** C200–C203 = 4 x TDK B32776G4506K000 (50 µF / 450 V DC, 20 A), VBAT–GND. J52 (VBAT) ve J53 (GND) M5 bara lug'ları.
- **Footprint:** `bldc_proje:C_Film_TDK_B32776G_L42mm_W28mm_P37.5x10.2mm_4pin`. 4 bacak (37.5 x 10.2 mm); her uçtaki iki bacak aynı elektrot (pad 1-1 / 2-2). Pad Ø3.2 mm, delik 1.4 mm.
- **Kart:** 116 x 66 mm, 2 katman.
  - Üstte tam VBAT, altta tam GND dökümü; üst üste iki düzlem düşük endüktanslı lamine bara gibi çalışır. Pad bağlantısı termal değil, tam dolgu.
  - Güç kartının evde üretim kuralları burada da geçerli (GUC_KARTI kural alanı).
  - J52/J53 lug kuralına eklendi: üstte 3.5 mm. Lug pad'leri Ø10.4 mm.
  - M3 montaj delikleri H9–H12.
- **Montaj:** bank, güç kartının sağ yanında ve aynı düzlemde. J52/J53, J1/J2 ile aynı hizada ve 14 mm aralıkta; iki kısa (≈ 25 mm) M5 delikli bakır bara ile bağlanır. Aradaki 0.1–0.2 mm yalıtımla üst üste konulursa endüktans düşük kalır.
- **Ekran görüntüsündeki kırmızı çerçeve:** güç kartı kısaltıldığında GUC_KARTI kural alanı ve VBAT/GND döküm sınırları eski ölçüde (y = −42) kalmıştı. Yeni kenara (y = 27) kırpıldı.
- **Doğrulama (kurallar devrede):** DRC 0 hata, 0 kart içi bağlantısız, parite 0. Kartlar arası 63 "bağlantısız" beklenen durum; 2'si bank ile güç kartı arasındaki VBAT/GND bara bağlantısı.
- **Çıktılar:** `uretim/film_bank_gerber.zip`, montaj ve BOM'da 5. modül.

### 2.14.10 Hâlâ açık olanlar
1. ~~Bara kondansatörü gerilim payı~~: 2.15.2 ile kapandı (450 V film bankı). Firmware V_BUS sınırı yine önerilir.
2. ~~Özel footprint land pattern'leri (2.14.9)~~: 2.16 ile kapandı.
4. Film bankı + batarya kablosu rezonansı: Faz 8'de ölçülecek (2.15.2).
3. Primer kaynaktan doğrulanamayanlar: SP0503BAHT pin şekli, MIC2128 ve kristal pad ölçüleri, LDL1117 kararlılık eğrileri, USB4105 pad ölçüleri.

## 3. Uygulama planı

Her fazın sonunda bir kontrol noktası var. Bir faz bitmeden sonrakine geçilmez.

### Faz 0 – Hazırlık
1. `.kicad_dru`, `sym-lib-table` ve `bldc_proje.kicad_sym` proje köküne alınacak.
2. Mevcut durumun yedeği alınacak (`yedek_rev2.5/`).
3. Bölüm 1'deki kararlar netleşecek.

**Kontrol noktası:** KiCad sembol ve kurallarla açılıyor; DRC'de güç kartı kuralları görünüyor.

### Faz 1 – Kontrol kartı şematiği (kararlardan bağımsız)
D1, D2, K1, K2, D5, K3, K4, K5, D19, K6, K7, K8, K9, K11, K12, D20, K14, K17. JTAG pinleri için firmware tarafında `JTAGEN = OFF` zorunlu.

**Kontrol noktası:**
* ERC 0 hata.
* Pin atama tablosu Microchip DS70005349 ile tek tek karşılaştırılacak.
* Buck için çıkış gerilimi, frekans, akım sınırı ve ripple hesabı bir tabloya yazılacak.

### Faz 2 – Gate sürücü şematiği
D11/K15, D12, D13, K17, K18, K19, K20, K21. Seçilen akım ölçüm yöntemine göre ölçüm hattı yeniden çizilecek.

**Kontrol noktası:** ERC. Her sürücüde VCC–COM, VDD–VSS ve VB–VS kapasitörleri pin başında.

### Faz 3 – Güç katı yeniden tasarımı (K-A … K-D kararlarına göre)
1. MOSFET seçimi ve kayıp/termal hesabı (tablo halinde, hedef Tj < 125 °C).
2. DC bara: elektrolit ve film seçimi, dalgalanma akımı hesabı, yarım köprü başına yerel MLCC'ler.
3. Akım ölçümü: Hall sensörü ya da 4 terminalli şönt; amplifikatör güç kartında.
4. Bakır bara / terminal tasarımı; bağlantı elemanı yalıtımı.
5. Gate pull-down'ları ve isteğe bağlı RC snubber.
6. 2 katman yığın yapısı (karar K-B). Yüksek akım yollarında maske penceresi ve lehim/bakır tel takviyesi (şartname 7.5). B.Cu'da GND, yarım köprülerin altında mümkün olduğunca bölünmeden. Kaplamalı via dizileri (drain başına 20–30 adet 0.3–0.5 mm) ya da ev üretiminde bakır tel perçin.

**Kontrol noktası:** Termal ve akım hesap tablosu. Döngü endüktansı tahmini 5 nH'nin altında.

### Faz 4 – EESM
D16, D17, K25, D8 (EESM şöntü). Alan akımı ölçüm yöntemi netleşecek.

### Faz 5 – Sistem koruması (yeni bir koruma modülü ya da güç kartına)
* **Sigorta:** batarya tarafında 200–250 A MEGA/ANL.
* **Kontaktör ve ön şarj:** ana kontaktör (EV200 sınıfı) + ön şarj (≈10 Ω/25 W, MCU kontrollü). Kontaktör kapanmadan bara şarjını kontrol ettiği için ters polariteye karşı da koruma sağlar. Ek olarak polarize güç konnektörü.
* **TVS:** bara TVS'i (60–64 V sınıfı, 5 kW).
* **Rejeneratif koruma:** fren chopper'ı ya da firmware'de regen sınırı + donanımsal aşırı gerilim karşılaştırıcısı.
* **Bleeder:** 4.7 kΩ/1 W, τ ≈ 53 s.
* **Sıcaklık:** her yarım köprüde ya da soğutucuda NTC, ADC'ye.
* **Donanımsal kapatma:** aşırı akım, aşırı gerilim ve düşük gerilim → PWM PCI fault + SD.

### Faz 6 – Yerleşim ve yönlendirme
* Kontrol kartında kristal ve dekuplaj mesafeleri.
* Gate kartında yüksek taraf bölgesinde döküm olmayacak.
* Güç kartında komütasyon döngüsü minimum.
* Ölçüm çiftleri diferansiyel ve kısa.
* Bağlantı elemanlarının çevresi bakırsız.
* Boşluk ≥ 0.6 mm, maske zorunlu.

### Faz 7 – Doğrulama (bu sefer elektriksel de)
Önceki revizyonda yalnızca DRC ve bağlantı kontrolü yapılmıştı. Bu yetersizdi. Her revizyonda şunlar yapılacak:

1. **ERC ve DRC** 0 hata, şematik–PCB paritesi 0.
2. **Pin denetimi.** Her entegrenin her pini datasheet pin tablosuyla otomatik karşılaştırılacak. Netlist dökümü + kontrol listesi, `C:\kw\audit\` altındaki dökümle aynı yöntem.
3. **Hesap tablosu.** Buck (Vout, fSW, ILIM, ripple), LDO ısısı, bootstrap boyutu, MOSFET kaybı ve Tj, şönt veya sensör gücü, kapasitör dalgalanma akımı, bakır kesit akımı.
4. **Gerilim değeri denetimi.** VBAT'a ve SW düğümüne bağlı her parça ≥ 100 V (ya da ≥ 1.5 × Vmax).
5. **Mekanik denetim.** Her vida, lug ve bağlantı elemanı çevresinde farklı potansiyelli bakır mesafesi ölçülecek (bu revizyonda kullandığım ölçüm betiğiyle).
6. **Bağımsız inceleme.** Tasarımcı dışında bir göz.

### Faz 8 – Prototip devreye alma prosedürü
1. Kart enerjisiz. Bara–GND, faz–GND ve faz–bara direnci ölçülür. Levha ile bütün netler arasında yalıtım ölçülür (megger, 500 V).
2. Kontrol kartı tek başına, 12 V'luk ayarlı beslemeden: buck çıkışı, LDO'lar, MCU programlama.
3. Gate kartları güç kartısız: PWM ve dead-time ile HO/LO dalga biçimleri osiloskopta. SD ve PCI fault testi.
4. Güç katı, akım sınırlı laboratuvar beslemesiyle (24 V, 1–2 A): boşta anahtarlama, faz düğümünde tepe gerilim ölçümü.
5. Düşük akımla motor (≤ 10 A), akım ölçüm kalibrasyonu.
6. Kademeli akım artışı, her kademede termal kamera. Donanımsal aşırı akım testi.
7. Batarya ile: önce ön şarj ve kontaktör sırası, sonra yük.

---

## 4. Firmware için zorunlu notlar
* `JTAGEN = OFF`. TDO (RB7, W akımı), TMS/TCK (RB10/RB11) ve TDI (RB12) pinleri başka işlevlerde kullanılıyor.
* ADC kanalları:
  * U = AN0 (dedike çekirdek 0);
  * V = ANA1 (dedike çekirdek 1, alternatif giriş seçilecek);
  * W = AN2 (paylaşımlı çekirdek);
  * V_BUS = AN3, IF = AN4 (paylaşımlı çekirdek, farklı tetikte).
  * Şartnamedeki AN9 ve AN1 adları yanlış.
* Osilatör: 12 MHz, `POSCMD = HS`.
* Karşılaştırıcı → PWM PCI fault, açılışta akım ölçüm offset kalibrasyonu, EESM'de üst taraf serbest dönüş süresinin sınırlanması.

## 5. Doğrulanması gereken varsayımlar
* **DI101N10PQ:** Rds(on), Qg ve Rth(j-c). Resmi PDF okunamadı; dağıtıcı değerleri birbirini tutmuyor (8–9 mΩ).
* **HoLRS2512 0.2 mΩ:** ESL değeri.
* **KLH 470 µF/63 V:** dalgalanma akımı değeri.
* **Raltron AS-12.000-18:** CL değeri.
* **TLV9001:** 100 nF yükte kararlılık.
* **IR2110:** VDD = 3.3 V'ta SD giriş akımı (datasheet Şekil 20B).
