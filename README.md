# BLDC / PMSM Motor Driver AI (KiCad 10 Project)

Bu depo, modüler BLDC / PMSM ve EESM motor sürücüsü donanım tasarımını (KiCad 10) içermektedir.

## 📁 Proje Yapısı

- `bldc_driver_ai.kicad_pro` / `.kicad_sch` / `.kicad_pcb`: Ana sistem projesi ve üst seviye bağlantılar
- `kontrol_karti.kicad_sch`: dsPIC tabanlı kontrol kartı şematiği
- `guc_karti.kicad_sch`: TO-220 MOSFET tabanlı yüksek akım güç katı ve 24× KOSHIN KLH (24 mF, 60 A rms) DC-link bara kondansatör grubu
- `gate_driver_u.kicad_sch`, `gate_driver_v.kicad_sch`, `gate_driver_w.kicad_sch`: Faz sürücü modülleri
- `eesm_modulu.kicad_sch`: EESM uyarma kontrol modülü
- `bldc_proje.pretty/` & `fp-lib-table`: Projeye özel footprint kütüphanesi (KOSHIN KLH radial elektrolitik, özel land pattern'ler)
- `montaj/`: 3D montaj modelleri (`.step`), mekanik yerleşim ve render görselleri
- `uretim/`: Modüllere ait üretim Gerber arşivleri (`.zip`), klasörleri ve SMD dizgi koordinat listeleri (`.csv`)
- `yardımcı_dokuman/`: Revizyon raporları, teknik analizler ve destekleyici dokümanlar
- `bldc_driver_ai_BOM.xlsx`: Güncel malzeme listesi (BOM)
- `SISTEM_REVIZYON_PLANI.md`: Kapsamlı sistem revizyon ve doğrulama planı

## 🛠️ Gereksinimler

- [KiCad 10.0+](https://www.kicad.org/) veya uyumlu sürüm

## 🔍 Tasarım İncelemesi 3: Bulgular ve Yapılan Değişiklikler

Bu bölüm, 3. tasarım incelemesinde açık kalan maddeleri ve her biri için verilen kararı özetler. Ayrıntılar `SISTEM_REVIZYON_PLANI.md` §2.21'dedir. Değişiklik öncesi yedek `yedek_faz7/pre_inceleme3/` klasöründedir.

**Doğrulama:** DRC 0 hata, kart içi bağlanmamış nokta 0, şematik–PCB eşleşmesi tam; ERC'de yalnızca bilinen 4 TJA1051 uyarısı var.

### 1. Harici 12 V girişi

**Bulgu:** Önceki incelemede söylenen üç madde şematikte düzeltilmemişti:
- D43 (TVS) F1'in önündeydi.
- U21 (5 V LDO) aşırı gerilime karşı korumasızdı.
- F1 (PTC) sıcakta kendiliğinden atabilir.

**Yapılan:**
- **D43 F1'in arkasına alındı.** D43'ün katodu artık 12V_EXT hattında. Uzun süren aşırı gerilimde TVS iletime geçiyor ve F1 atıyor; TVS yanmıyor.
- Kartta D43, D42'nin yanına (322.6, 17.6) taşındı ve 12V_EXT'e alt katmandan kısa bir köprüyle bağlandı. Eski izler silindi.

**Bilerek değiştirilmeyenler (açık madde):**
- **U21:** hâlâ 12 V hattından besleniyor. Önerilen çözüm, U21'i kaldırıp CAN alıcısını mevcut 5V_SYS hattından beslemek.
- **F1 (PTC):** sıcakta değer kaybettiği için kutu içi sıcaklıkta erken atabilir. Prototipte kontrol edilmeli; gerekirse daha yüksek akımlı bir PTC veya cam sigorta kullanılabilir.

### 2. VBAT ve GND pabuçları arası 15 mm

**Bulgu:** 100 A için kullanılan 25–35 mm² pabuçlar 12–15 mm genişliğinde. Pabuçlar dönerse birbirine değebilir ve bataryayı kısa devre eder.

**Yapılan:** Mesafe korundu, izolasyon zorunlu hale getirildi:
- BOM'un mekanik sayfasına zorunlu bir satır eklendi: J1 (VBAT) ve J2 (GND) pabuçlarının ikisine de kalın makaron (ısıyla daralan) veya izolasyon başlığı geçirilecek.
- Güç kartında J1'in altına ipek baskı uyarısı eklendi: `J1/J2 PABUC: MAKARON VEYA BASLIK ZORUNLU`.

### 3. Kondansatörlerin MOSFET'lerin üstünde olması

**Bulgu:**
- Kondansatör bacak ucu ile MOSFET gövdesi arasında yalnızca yaklaşık 0.7 mm boşluk vardı (2.2 mm aralık, eksi 1.5 mm kesilmiş bacak).
- MOSFET ısısı kondansatörleri ısıtır ve ömürlerini kısaltır.
- Arızalı bir MOSFET'i değiştirmek için 24 kondansatörlü kartın tamamı sökülmeli.

**Yapılan:**
- **Kart, soğutucunun 8.5 mm üstüne çıkarıldı** (önceden 7 mm). Plastik gövde ile kart arası artık 3.7 mm. Bacaklar 1.5 mm kesildiğinde yaklaşık 2.2 mm pay kalıyor. MOSFET bacak boyu bu yüksekliğe yetiyor.
- Montaj modeli (`montaj/`) ve BOM'daki ara parçalar (M3 vida + 8.5 mm ara parça) güncellendi.

**Açık kalanlar:**
- **Sıcaklık:** 5 kW'ta kondansatör gövde sıcaklığı prototipte ölçülmeli.
- **Servis:** MOSFET değişimi için kartın sökülmesi gerekiyor. Bu bir tasarım hatası değil, yerleşimin kabul edilmiş bir sonucu.

### 4. Motor tarafı

**Bulgu:** Hall sensörü için 5 V besleme ve motor sıcaklık sensörü için giriş yoktu.

**Karar:** Hall sensörü yerine enkoder kullanılacak.

**Enkoder:**
- Besleme 3.3 V olarak kaldı.
- Seri dirençler R71, R73 ve R75 100 Ω'dan 1 kΩ'a çıkarıldı. 5 V çıkışlı bir enkoder takılsa bile MCU pinine giren akım yaklaşık 1.4 mA'de kalır; sınır 5 mA.

**Motor sıcaklık girişi (yeni):**
- **Konnektör:** J52 "Motor_Sicaklik" (1×02 pin header). Pin 1 NTC_MOT, pin 2 GND. Kontrol kartının üst kenarında, enkoder soketinin sağında.
- **Devre:**
  - R145: 2.2 kΩ %0.1 pull-up (3V3_ANA hattına). Hem 10k NTC hem de KTY84-130 sensörüyle çalışır.
  - R146: 22 kΩ seri koruma direnci. Sargı yalıtımı bozulup hatta 60 V gelse bile pine giren akımı yaklaşık 2.6 mA'de tutar.
  - C224: 100 nF filtre.
- **MCU bağlantısı:** U6 pin 40 (**AN16**/RC7). Firmware'de ADC kanalı AN16 olarak tanımlanmalı.
  - Önce pin 19 (AN23) planlandı, ancak dar pin aralığı yüzünden iz çıkarılamadı.
  - Yeni iz U6'nın altındaki GND adasını ana GND'den ayırdığı için üst katmana bir GND köprüsü eklendi.

### Teyit bekleyen maddeler

Motor datasheet'i (`QS 48V 5000W 690RPM V3.pdf`) okunamadı. Aşağıdakiler motor kablosuna ve datasheet'e göre doğrulanmalı:
- Motor sıcaklık sensörünün tipi (NTC veya KTY84) ve J52 pin dizilimi.
- Enkoderin 3.3 V beslemeyle çalışıp çalışmadığı.
