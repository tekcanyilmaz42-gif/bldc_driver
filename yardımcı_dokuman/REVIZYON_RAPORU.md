# bldc_driver_ai – Revizyon Raporu (Rev 2.5 – 2 katman, evde üretilebilir güç kartı, 15 mm soğutucu)

Bu revizyon, `Yeni Metin Belgesi.txt` (Rev 2.3 şartnamesi) ve mevcut proje dosyaları incelenerek hazırlandı.
Orijinal dosyalar değiştirilmeden `yedek_orijinal/` klasörüne kopyalandı.

## 1. Proje yapısı

| Dosya | İçerik |
|---|---|
| `bldc_driver_ai.kicad_pro / .kicad_sch / .kicad_pcb` | Ana proje. Şematik hiyerarşik: kök + 6 modül sayfası. PCB'de modüller senin kurduğun düzende, ayrı kart konturları olarak yan yana duruyor. |
| `guc_karti.kicad_sch` | Katman 1 – güç kartı: 24×470 µF, 24×2.2 µF, 12×DI101N10PQ, 3 şönt, M5 bara pabuçları, kartlar arası soketler |
| `gate_driver_u/v/w.kicad_sch` | Katman 2 – faz başına IR2110S gate driver modülü (MOSFET'lerin tam üstünde) |
| `kontrol_karti.kicad_sch` | Katman 3 – dsPIC33CK, MIC2128 buck, 4×NCP1117, izole USB-UART, CAN, enkoder, 3 faz akım ölçümü |
| `eesm_modulu.kicad_sch` | Katman 4 – EESM uyartım H-köprüsü + primer akım ölçümü |
| `bldc_driver_ai_3d_montaj.kicad_pcb` | **3B montaj**: modüller üst üste (KiCad 3B görüntüleyicide açın) |
| `bldc_driver_ai.kicad_dru` | Güç kartına özel evde üretim DRC kuralları (iz ≥ 1.5 mm, via ≥ 1.0/1.8 mm) |
| `3d/*.step` | Montajın kullandığı modül STEP dosyaları |
| `bldc_proje.kicad_sym`, `sym-lib-table` | Proje sembol kütüphanesi: 80 pinli dsPIC33CK256MP508 (global kütüphanedeki sembol 17 pinli ve yanlış) |

Yığın yükseklikleri: güç kartının üstü 1.6 mm. Her kat arası mesafe 11 mm: 8.5 mm dişi soket + 2.5 mm erkek başlık gövdesi.
Buna göre kat tabanları şöyle: gate driver 12.6 mm, kontrol 25.2 mm, EESM 37.8 mm.
Hizalama ofsetleri (ana PCB koordinatları): gate = güç + (118, 0); kontrol = güç + (118, −41.45); kontrol = EESM + (0, −105.15).
Kartlar arası 20 konnektör çiftinin pad'leri script ile doğrulandı: hata 0.00 mm.

## 2. Bulunan hatalar ve yapılan revizyonlar

| No | Hata | Revizyon |
|---|---|---|
| R1 | **Şematik bozuktu.** ~150 parçanın değeri ve footprint'i toplu olarak "Screw_Terminal_M3" yapılmıştı. Bir güç sembolü de bu ada dönüşmüş ve **GND, VBAT ve GND_PC netlerini birbirine kısa devre etmişti.** Dosyanın sonunda, kök ifade kapandıktan sonra J29–J43 eklenmişti. | Şematik, PCB'deki temiz netlistten KiCad MCP ile hiyerarşik olarak yeniden üretildi. ERC'de 0 hata var; PCB ile parite birebir. |
| R2 | ADuM1201, ADuM1200 pin düzenine göre bağlanmıştı: CH340 TXD → izolatörün VOA **çıkışına**, dsPIC TX → VOB **çıkışına** gidiyordu. Çıkış çıkışa bağlı olduğu için UART çalışmazdı. (Şartnamedeki pin tablosu da ADuM1200'e göre yazılmış.) | CH340 TXD→VIB(3), VOB(6)→RXD_DSP(46), TXD_DSP(47)→VIA(7), VOA(2)→CH340 RXD |
| R3 | TJA1051'in VCC'si izole USB 5 V'tan besleniyordu, GND'si ise sistem GND'sindeydi. Bu izolasyonu bozuyordu ve CAN yalnızca USB takılıyken çalışabilirdi. | Kontrol kartına sistem tarafında U21 NCP1117‑5.0 eklendi (5V_CAN). Ayrıca C135 10 µF, C136 ve C143 100 nF eklendi. |
| R4 | AVDD ferrit boncuğundan sonra kapasitör yoktu, yani LC filtre eksikti. | C106 (100 nF) ve C107 (1 µF) AVDD tarafına taşındı. |
| R5 | ADuM1201 VDD2'de yerel dekuplaj yoktu. | C137 100 nF eklendi. |
| R6 | MIC2128 VIN'de yalnızca 100 nF vardı. 12 V çıkışındaki 10 µF/0603 kapasitörler de 12 V DC bias altında kapasitelerinin çoğunu kaybediyordu. | C138 ve C139 (2.2 µF/100 V, 1206) eklendi; C39 ve C40 10 µF/25 V 1206'ya çevrildi. |
| R7 | V_BUS bölücüsü (100k/1k) ADC aralığının yalnızca ~%16'sını kullanıyordu. | R67 4.7k yapıldı: 54 V → 2.42 V, ölçülebilen üst sınır ~73 V. |
| R8 | **Kontrol kartına VBAT gelmiyordu**; buck VIN/EN, Q14 ve V_BUS ölçümü beslenemiyordu. | Güç kartından kontrol kartına riser eklendi: J46 (2×2 soket) / J47 (2×2 başlık). |
| R9 | **Faz düğümleri (IR2110 VS) güç kartıyla gate driver arasında hiçbir konnektörden geçmiyordu**; yüksek taraf sürücüsü havadaydı. | Gate konnektörleri 3 pinli yapıldı: G_xH1 – Phase (VS, Kelvin) – G_xH2 ve G_xL1 – xSH (COM) – G_xL2. |
| R10 | IR2110 COM dönüş akımı, akım ölçüm hattını paylaşıyordu. SD sinyali de güç kartında yükü olmayan bir pine gidiyordu. | SD, kontrol konnektörüne taşındı (HIN–SD–LIN, IR2110'un 12/13/14 pin sırasıyla aynı). COM ayrı pinden geliyor. |
| R11 | Kelvin ölçüm yoktu: INA181 IN− ucu şönt pad'ine değil genel GND'ye gidiyordu. | Her şöntte iki net-tie (NT1–NT8) var. x_SNS_P/N diferansiyel çiftleri ayrı pinlerle taşınıyor; gate driver kartından stacking soketle geçiyor. |
| R12 | Alçak taraf gate pull-down dirençleri lojik GND'ye bağlıydı. | Pull-down'lar MOSFET source'una (xSH) bağlandı. |
| R13 | **EESM modülünün sistemle hiçbir bağlantısı yoktu.** | J48/J49 (2×6) eklendi: 12V, 3V3_IO, 3V3_ANA, IN181_IFout, primer_x_H/L, SD ve GND. Güç için J44/J45 2.5 mm² kablo pad'leri, yerel bulk için C140 220 µF/63 V ve C141/C142 2.2 µF/100 V eklendi. |
| R14 | ~200 A için M3 kutulu terminal blokları yetersizdi. | J1–J5, via halkalı M5 pabuç pad'lerine çevrildi. |
| R15 | USB-C konnektörün shield pini boştaydı. | GND_PC'ye bağlandı. |
| – | Orijinal PCB'de W fazındaki gate driver ile kontrol konnektörleri 1 mm kaymıştı (J22 ve J41). | Tüm konnektör çiftleri yeniden hizalandı ve doğrulandı. |

## 3. PCB (Rev 2.5: tüm kartlar 2 katman)

* **Bütün kartlar 2 katman (F.Cu / B.Cu), 2 oz (70 µm) bakır, 1.6 mm FR4.** 4 katmanlı yapı maliyet nedeniyle kaldırıldı ve tüm kartlar sıfırdan yeniden yönlendirildi. Her kartta B.Cu, mümkün olan her yerde GND dökümü.
* **İz genişlikleri** (net sınıfları senin projendeki adlarla): EESM_Power 2.5 mm, HV_Power_10kW 1.0 mm, Gate_Drive 0.8 mm, LV_Supply_12V_3V3 1.0 mm, Kelvin_Sense 0.3 mm, Default 0.25 mm.
  * İnce adımlı pad'lere (QFN/SOIC/0603) girerken iz pad genişliğine kadar daraltılıyor.
  * Kontrol kartında akımlar 0.5 A'nın altında ve dsPIC çevresi sıkışık. Bu yüzden 12V 0.5 mm, VBAT 0.4 mm, 3V3 0.4 mm ve BUCK_SW 0.6 mm kullanıldı.
  * Kontrol kartındaki en ince iz 0.15 mm: yalnızca U9 (MIC2128) pin 14'ten R21 GND pad'ine giden 2 mm'lik bağlantı. Bu yüzden kontrol tarafında kart minimumu 0.15 mm yapıldı; 2 katman standart üretimde (0.127 mm'ye kadar) ek maliyet çıkarmaz.
* **Yönlendirme:** Freerouting kullanılmadı. Kendi yazdığım 8 yönlü (0/45/90°) A* yönlendiricisi kullanıldı; izler karta KiCad MCP sunucusu üzerinden (`route_trace`, `add_via`, `add_copper_pour`) eklendi. Bükümler 45° tutuldu; 90° dönüşlere yalnızca kaçınılmazsa ve yüksek cezayla izin verildi.

### 3.1 Güç kartı – evde üretim kuralları (yalnızca Katman 1)

| Kural | Değer | Durum |
|---|---|---|
| En dar iz | **1.5 mm** | 213 segmentin en incesi 1.5 mm |
| En küçük via | **1.0 mm delik / 1.8 mm pad** | 90 via'nın tamamı 1.8/1.0 |
| Bakır–bakır açıklık | genelde ≥ 0.25 mm; **en az 0.2 mm** | Bkz. aşağıdaki not |

* Bu kurallar `bldc_driver_ai.kicad_dru` dosyasında, güç kartını kaplayan `GUC_KARTI` kural alanına bağlı. DRC bu kuralları yalnızca bu kartta uygular; diğer kartlar normal fabrika kurallarıyla kalır.
* **Açıklık notu:** DI101N10PQ footprint'inde gate pini ile dren pad'i arasındaki boşluk çok dar. Bu pine 1.5 mm iz girince dren bakırına 0.225–0.24 mm kalıyor. Footprint değişmeden bu mesafe büyütülemiyor. Evde aşındırmada bu noktalara dikkat et (toner ya da film kaplamasını iyi yap, gerekirse maket bıçağıyla kontrol et).
* Güç bakırı F.Cu'da VBAT, faz ve xSH alanlarıyla taşınıyor. B.Cu'da tam GND dökümü ile her fazın altında VBAT ve faz adaları var.

### 3.2 Soğutucu (15 mm düz alüminyum levha, güç kartının altında)

* **Termal via:** Her MOSFET'in dren pad'inde 1.0 mm delikli via'lar var. Isı bu via'larla B.Cu'ya iner. Aynı netli B.Cu adasına ek via'lar konuldu; bunlar başka netlerle açıklığı kontrol edilerek yerleştirildi.
* **Açık bakır (maske açıklığı):** Her MOSFET'in tam altında B.Mask'te ≈5.2×3.8 mm açıklık var. Plaka bu bakıra ısıl pedle doğrudan basar. Altından başka bir netin izi ya da via'sı geçen yerlerde açıklık otomatik olarak açılmadı.
* **Elektriksel yalıtım şart.** Alt adalar dren potansiyelinde: VBAT (yüksek taraf) ve faz (alçak taraf). Levha ile kart arasına **yalıtkan ısıl ped** (silikon/kapton, ≥ 1.5 kV) konmalı. Alüminyum levha doğrudan değmemeli.
* **Altta çıkıntı yapanlar:** Kapasitörler, bara pabuçları ve konnektörler THT. Bunların altta kalan lehim uçları kısa kesilmeli, ya da levhada bu noktalara cep/delik açılmalı. Aksi halde ped düz oturmaz ve kısa devre riski olur.
* **Sabitleme:** H1–H8, M3 delikleri (3.2 mm). Kartı levhaya eşit basınçla bastırmak için kullanılır; vidalar plastik pul ya da yalıtkan burçla takılmalı.
* **3B montaj:** `bldc_driver_ai_3d_montaj.kicad_pcb`. Levha 15 mm kalınlığında gövde olarak gösteriliyor, kartlar üstünde üst üste duruyor. Kat tabanları: güç kartı 0, gate driver 12.6 mm, kontrol 25.2 mm, EESM 37.8 mm (güç kartı üst yüzeyine göre). Görseller: `montaj_3d.png`, `master_top.png`, `master_bottom.png`.

## 4. Doğrulama (Rev 2.5)

| Kontrol | Sonuç |
|---|---|
| ERC (kicad-cli, tüm hiyerarşi) | **0 hata.** 4 uyarı var; hepsi kullanıcı kütüphanesindeki TJA1051 sembolünde tüm pinlerin "bidirectional" tanımlı olmasından geliyor, zararsız. |
| PCB ↔ şematik paritesi | **0** |
| Kartlar arası konnektör hizası | 20/20 çift, hata 0.00 mm |
| DRC – tüm kartlar (güç kartının özel kuralları dahil) | **0 hata, modül içi bağlantısızlık 0.** Uyarılar: 3 lib_footprint_mismatch (kullanıcı kütüphanesi), 3 silk_edge_clearance |
| 45° kuralı | 3289 iz segmentinin tamamı 0°/45°/90° doğrultusunda; 0.5 mm'den uzun dik olmayan segment yok. 90° kırılmaların 79'u pad bakırının içinde, 1'i via'da; açık bakırda yalnızca 2 adet zorunlu 90° kırılma kaldı. |
| "Bağlantısız" 61 kalem | Kartlar arası netler (örneğin güç kartı J26 ↔ gate kartı J11). Bu netler fiziksel olarak kart üstü konnektörlerle bağlanıyor; tek PCB'deki çok kartlı düzenin doğal sonucu, hata değil. |

2 katmana geçişte kontrol kartında elle ya da hedefli olarak tamamlanan bağlantılar:
* **MIC2128 (U9):** FREQ ve 12V via'ları yeniden yerleştirildi. VIN pin 10, C9'a bağlandı. GND pin 14, R21 GND pad'ine 45° kırılmalı ince bir izle bağlandı; böylece pad altı GND adası ana GND'ye katıldı.
* **dsPIC GND pin 26:** Paket altındaki GND adası çevredeki sinyallerle kapalı kalıyordu. AVDD, G_WH_IN, G_WL_IN, IN181_IFout, IN181_Vout, OSCI_DSP ve V_BUS_SENSE sökülüp yeniden yönlendirildi, pin 26 ana GND'ye bağlandı.
* **J48 pin 3 (3V3_IO → EESM):** Yönlendirildi.
* **U10 GND pini, Q13, U6.50, U6.26:** Kısa saplama ve via ile B.Cu GND'ye bağlandı.
* **USB-C:** D+, D−, CC ve VBUS bağlantıları 2 katmana uyarlandı.
* **GND dikişi:** Kontrol kartında ve EESM'de 2.5 mm ızgarada dikiş via'ları var; parça altlarına via konmadı. Ana GND'ye bağlanamayan adalar için ayrı bağlantı izleri çizildi, ölü via'lar silindi.

## 5. Not edilen ve bilerek değiştirilmeyen sapmalar

* Bootstrap kapasitörü 10 µF + 100 nF (şartnamede 1 µF + 100 nF). Daha büyük değer zararsız ve senin PCB tercihin, korundu.
* Bootstrap ve deşarj diyotları SS18 (80 V Schottky); şartnamede ES1J / BAT54 yazıyor. 54 V bara için yeterli; sıcakta kaçak akım artar.
* Referans numaraları şartnamedekinden farklı; PCB'deki numaralar korundu: INA181 = U7, U12, U14, U18; TLV9001 = U10, U11, U13, U17; LDO'lar = U8, U19, U20, U21; CAN = U22.
* 0.2 mΩ 2512 şönt 150 A RMS'de yaklaşık 4.5 W harcar; parçanın güç sınıfı kontrol edilmeli.
* 200 A sürekli akım için 2 oz bakır tek başına yetmez. VBAT, faz ve GND yollarına bakır bara veya lehim takviyesi önerilir.
* J46/J47 riser'ı güç kartından kontrol kartına ~23.6 mm boşluğu geçiyor. Uzun pinli (stacking) başlık ya da ara soket gerekir; 3B modelde boşluk görünür.
* Bazı kullanıcı kütüphanesi footprint'leri güncel kütüphane kopyasından farklı (lib_footprint_mismatch uyarısı). Ayrıca KLH kapasitörün 3B modeli bulunamıyor; 3B görünümde beyaz/yeşil silindir olarak çıkıyor.
