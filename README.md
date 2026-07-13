# ZKDP-FL

Bu rehber, ZKDP-FL deneylerinin sanal makine yerine fiziksel host bilgisayarda sıfırdan, tutarlı koşullarla uygun biçimde çalıştırılması için hazırlanmıştır.

## 1. Proje klasörüne geç

Aşağıdaki yolu kendi proje konumuna göre düzenle:

```powershell
cd C:\ZKDP-FL\ZKDP_FL_SCI_Final
```

## 2. Python ortamını oluştur

Python 3.11 kullanılması önerilir.

```powershell
py -3.11 -m venv .venv
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

### GPU durumunu kontrol et

```powershell
python -c "import torch; print('PyTorch:',torch.__version__); print('CUDA:',torch.cuda.is_available()); print('GPU:',torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU')"
```

RTX 5060 doğru tanınırsa aşağıdakine benzer bir çıktı görülmelidir:

```text
CUDA: True
GPU: NVIDIA GeForce RTX 5060
```

`CUDA: False` görülürse deneyler yine çalışır; ancak makine öğrenmesi aşamaları CPU üzerinde daha uzun sürebilir.

---

## 3. E6 parametrelerini güncelle

Mevcut kampanya dosyasında E6 eski parametrelerle tanımlı olabilir. Bu nedenle önce dosyayı yedekle:

```powershell
Copy-Item experiments\campaign.py experiments\campaign_original.py -Force
```

Ardından aşağıdaki komutla E6 parametrelerini güncelle:

```powershell
@'
from pathlib import Path

path = Path("experiments/campaign.py")
text = path.read_text(encoding="utf-8")

old = '''    e6base = ("--dataset", "mnist", "--clients", "1", "--rounds", "10", "--epochs", "5",
              "--batch", "32", "--lr", "0.5", "--subset", "3000", "--alpha", "100",'''

new = '''    e6base = ("--dataset", "mnist", "--clients", "1", "--rounds", "10", "--epochs", "2",
              "--batch", "256", "--lr", "0.05", "--subset", "12000", "--alpha", "100",'''

if old not in text:
    raise SystemExit("E6 ayar bloğu bulunamadı; campaign.py değiştirilmedi.")

path.write_text(text.replace(old, new, 1), encoding="utf-8")
print("E6 ayarları başarıyla güncellendi.")
'@ | python -
```

### Güncellemeyi kontrol et

```powershell
Select-String -Path experiments\campaign.py -Pattern "e6base" -Context 0,4
```

Aşağıdaki değerler görünmelidir:

```text
epochs 2
batch 256
lr 0.05
subset 12000
target-eps 2
```

E6 için kullanılacak temel parametreler:

- 10 federated round
- Her round için 2 yerel epoch
- Toplam 20 yerel epoch
- Batch size: 256
- Learning rate: 0.05
- Eğitim alt kümesi: 12.000 örnek
- Clipping normu: 1.0
- Hedef gizlilik bütçesi: ε = 2
- Delta: 10^-5
- Canary sayısı: 1.000

---

## 4. Kod testlerini çalıştır

İstatistiksel ve tekrar üretilebilirlik testleri:

```powershell
pytest -q tests\test_audit.py tests\test_repro.py
```

Kriptografik testler:

```powershell
python tests\test_zkp.py
```

Python sözdizimi kontrolü:

```powershell
python -m py_compile `
  experiments\campaign.py `
  experiments\analyze.py `
  experiments\make_plots.py `
  src\phase1_fl_baseline.py `
  src\phase2_fl_dp.py `
  src\phase3_fl_dp_zkp.py `
  src\phase4_audit.py
```

---

## 5. Host sistem bilgilerini kaydet

Deney ortamını makalede doğru raporlamak için aşağıdaki çıktıyı sakla:

```powershell
python -c "import platform,torch,os; print('OS:',platform.platform()); print('CPU:',platform.processor()); print('Logical cores:',os.cpu_count()); print('PyTorch:',torch.__version__); print('CUDA:',torch.cuda.is_available()); print('GPU:',torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'None')"
```

Mümkünse bu çıktıyı ayrıca bir dosyaya yönlendir:

```powershell
New-Item -ItemType Directory -Force output\metadata | Out-Null

python -c "import platform,torch,os; print('OS:',platform.platform()); print('CPU:',platform.processor()); print('Logical cores:',os.cpu_count()); print('PyTorch:',torch.__version__); print('CUDA:',torch.cuda.is_available()); print('GPU:',torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'None')" |
Out-File output\metadata\host_system_info.txt -Encoding utf8
```

---

## 6. Kısa smoke testi çalıştır

Smoke testi yalnızca sistemin doğru çalıştığını kontrol eder. Bu sonuçlar makalede kullanılmamalıdır.

```powershell
python run_experiments.py --profile smoke --device cpu --clean
```

Smoke testi tamamlandıktan sonra test çıktısını sil:

```powershell
Remove-Item output -Recurse -Force
New-Item -ItemType Directory output | Out-Null
```

---

# 7. Ana SCI kampanyasını çalıştır

İlk aşamada E1, E2, E4, E5, düzeltilmiş E6, E8 ve E9 çalıştırılır.

```powershell
python experiments\campaign.py `
  --profile paper `
  --device auto `
  --only E1,E2,E4,E5,E6,E8,E9 `
  --timeout-hours 24
```

Bu aşamada beklenen koşu sayısı:

| Deney | Koşu sayısı |
|---|---:|
| E1 | 10 |
| E2 | 30 |
| E4 | 40 |
| E5 | 80 |
| E6 | 60 |
| E8 | 60 |
| E9 | 20 |
| **Toplam** | **300** |

---

# 8. E3 ZKP performans deneylerini host CPU üzerinde çalıştır

E3 performans deneyleri CPU üzerinde ayrı çalıştırılmalıdır. Böylece ZKP prove ve verify süreleri GPU eğitim süresinden ayrılır.

```powershell
python experiments\campaign.py `
  --profile paper `
  --device cpu `
  --only E3 `
  --seeds 0,1,2,3,4,5,6,7,8,9 `
  --timeout-hours 24
```

E3 için beklenen koşu sayısı:

```text
4 ZKP konfigürasyonu × 10 seed = 40 koşu
```

E3 koşullarında kullanılan temel yapı:

- MODP 2048-bit grup
- Slack = 6
- Clipping normu = 1
- ε = 8
- 3 istemci
- 2 federated round
- k = 4, 8 ve 16
- İki farklı model boyutu

Performans ölçümleri sırasında:

- Windows Update geçici olarak duraklatılmalı
- Güç modu “En iyi performans” olmalı
- Ağır uygulamalar kapatılmalı
- Aynı anda başka deney çalıştırılmamalı
- Bilgisayar uyku moduna geçmemeli
- İlk çalışma warm-up olarak değerlendirilebilir

---

## 9. Windows deneylerinden sonra analiz üret

```powershell
python run_experiments.py --analyze-only
```

### Ham koşu sayısını kontrol et

```powershell
(Get-ChildItem output\raw -Filter "seed_*.json" -Recurse).Count
```

E7 hariç beklenen toplam:

```text
340
```

Bu sayı:

```text
300 ana kampanya + 40 E3 = 340
```

şeklinde oluşur.

### Kalite kontrollerini görüntüle

```powershell
Import-Csv output\statistics\data_quality_checks.csv |
Format-Table -AutoSize
```

Tüm kritik satırlar `PASS` olmalıdır.

### Kampanya oturumlarını kontrol et

```powershell
Get-ChildItem output\metadata\sessions\campaign_*.json |
ForEach-Object {
    $m = Get-Content $_.FullName | ConvertFrom-Json
    [PSCustomObject]@{
        File     = $_.Name
        Status   = $m.status
        Planned  = $m.planned_run_count
        Complete = $m.n_complete
        Failed   = $m.n_failed
    }
} | Format-Table -AutoSize
```

Beklenen:

```text
Status = complete
Failed = 0
```

---

# 10. E7 ezkl/Halo2 deneyini WSL2 üzerinde çalıştır

E7 aynı fiziksel bilgisayarda, Windows host üzerindeki WSL2 Ubuntu ortamında çalıştırılmalıdır.

## WSL2 içinde proje klasörüne geç

Örneğin proje Windows’ta `C:\ZKDP-FL\ZKDP_FL_SCI_Final` içindeyse:

```bash
cd /mnt/c/ZKDP-FL/ZKDP_FL_SCI_Final
```

## E7 ortamını oluştur

```bash
python3 -m venv .venv-ezkl
source .venv-ezkl/bin/activate

python -m pip install --upgrade pip setuptools wheel
pip install -r requirements-ezkl.txt
```

## E7 kampanyasını çalıştır

```bash
python experiments/campaign.py \
  --profile paper \
  --only E7 \
  --include-ezkl \
  --seeds 0,1,2,3,4 \
  --device cpu \
  --timeout-hours 24
```

E7 için beklenen koşu sayısı:

```text
2 model boyutu × 5 seed = 10 koşu
```

## E7 sonrasında analizleri güncelle

```bash
python run_experiments.py --analyze-only
```

Windows PowerShell’e döndükten sonra toplam ham koşu sayısını kontrol et:

```powershell
(Get-ChildItem output\raw -Filter "seed_*.json" -Recurse).Count
```

Beklenen toplam:

```text
350
```

---

# 11. Sonuç dosyalarını kontrol et

## Ana tablolar

```powershell
Get-ChildItem output\tables\*.csv
```

Beklenen tablolar:

```text
T1_privacy_utility.csv
T2_zkp_cost.csv
T3_attack_detection.csv
T4_detection_probability.csv
T5_one_run_privacy_audit.csv
T6_ezkl_cost.csv
T7_noniid_robustness.csv
T8_cross_dataset.csv
```

## Grafikler

```powershell
Get-ChildItem output\figures\*.png
Get-ChildItem output\figures\*.pdf
```

## İstatistiksel testler

```powershell
Import-Csv output\statistics\inferential_tests.csv |
Format-Table -AutoSize
```

## E6 sonuçlarını özel olarak kontrol et

```powershell
Import-Csv output\tables\T5_one_run_privacy_audit.csv |
Format-Table -AutoSize
```

E6 için beklenen temel yapı:

- Honest doğruluk, rastgele tahmin düzeyinin belirgin üzerinde olmalı
- Honest koşullarda `claim_rejected` düşük veya sıfır olmalı
- Ampirik epsilon sıralaması aşağıdaki gibi olmalı:

```text
honest < low-noise < no-noise
```

- No-noise saldırısı, low-noise saldırısından daha yüksek yakalama oranına sahip olmalı
- Noise canary, label canary kadar veya daha güçlü olmalı

---


# 12. Tek bakışta çalıştırma sırası

## Windows PowerShell

```powershell
# Proje klasörü
cd C:\ZKDP-FL\ZKDP_FL_SCI_Final

# Sanal ortam
py -3.11 -m venv .venv
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1

# Paketler
python -m pip install --upgrade pip setuptools wheel
pip install -r requirements.txt

# Testler
pytest -q tests\test_audit.py tests\test_repro.py
python tests\test_zkp.py

# Ana kampanya
python experiments\campaign.py --profile paper --device auto --only E1,E2,E4,E5,E6,E8,E9 --timeout-hours 24

# E3 host CPU performansı
python experiments\campaign.py --profile paper --device cpu --only E3 --seeds 0,1,2,3,4,5,6,7,8,9 --timeout-hours 24

# Analiz
python run_experiments.py --analyze-only
```

## WSL2 Ubuntu

```bash
cd /mnt/c/ZKDP-FL/ZKDP_FL_SCI_Final

python3 -m venv .venv-ezkl
source .venv-ezkl/bin/activate

python -m pip install --upgrade pip setuptools wheel
pip install -r requirements-ezkl.txt

python experiments/campaign.py --profile paper --only E7 --include-ezkl --seeds 0,1,2,3,4 --device cpu --timeout-hours 24

python run_experiments.py --analyze-only
```

---


