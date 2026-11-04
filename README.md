# BLDC / PMSM Motor Driver AI (KiCad 10 Project)

Bu depo, modüler BLDC / PMSM ve EESM motor sürücüsü donanım tasarımını (KiCad 10) içermektedir.

## 📁 Proje Yapısı

- `bldc_driver_ai.kicad_pro` / `.kicad_sch` / `.kicad_pcb`: Ana sistem projesi ve üst seviye bağlantılar
- `kontrol_karti.kicad_sch`: dsPIC tabanlı kontrol kartı şematiği
- `guc_karti.kicad_sch`: TO-220 MOSFET tabanlı yüksek akım güç katı
- `gate_driver_u.kicad_sch`, `gate_driver_v.kicad_sch`, `gate_driver_w.kicad_sch`: Faz sürücü modülleri
- `eesm_modulu.kicad_sch`: EESM uyarma kontrol modülü
- `montaj/`: 3D montaj modeli (`.step`), mekanik yerleşim ve render görselleri
- `uretim/`: Üretim Gerber arşivleri (`.zip`) ve SMD dizgi koordinat listeleri (`.csv`)
- `yardımcı_dokuman/`: Revizyon raporları, teknik analizler ve destekleyici dokümanlar
- `bldc_driver_ai_BOM.xlsx`: Malzeme listesi (BOM)
- `SISTEM_REVIZYON_PLANI.md`: Kapsamlı sistem revizyon ve doğrulama planı

## 🛠️ Gereksinimler

- [KiCad 10.0+](https://www.kicad.org/) veya uyumlu sürüm

