# Klasik Görüntü İşleme ile Mikroskobik Görüntülerde Hücre Sayımı ve Zamansal Analiz

Görüntü İşleme dersi — yüksek lisans ara sınav projesi (bireysel).
Bu projede, ardışık floresan mikroskopi görüntülerinde hücre çekirdeklerini yalnızca **klasik** görüntü işleme yöntemleriyle bölütleyip sayıyorum. Pipeline'ı kontrollü deneylerle kuruyor, doğrulama kümesinde seçip kilitliyor, ardından bağımsız test bloklarında değerlendiriyorum. Derin öğrenme ya da önceden eğitilmiş modeller kullanılmamıştır.

## Veri seti
* **Cell Tracking Challenge — Fluo-N2DH-GOWT1**, eğitim arşivi (sekans 01 ve 02, `01_GT/TRA`, `02_GT/TRA`).
* Resmî indirme bağlantısı: https://data.celltrackingchallenge.net/training-datasets/Fluo-N2DH-GOWT1.zip
* Veri sayfası: https://celltrackingchallenge.net/2d-datasets/
* Veri seti bu depoda **paylaşılmaz**. Notebook, veri `data/Fluo-N2DH-GOWT1/` altında yoksa onu resmî adresten kendisi indirir.
  Başka bir konum için `GOWT1_PARENT` ortam değişkeni kullanılabilir (bu klasörün içinde `Fluo-N2DH-GOWT1/` olmalıdır).

## Tekrarlanabilirlik
* **Random seed:** öğrenci numarasının son 5 hanesi + 2026 = 29059 + 2026 = **31085** (`np.random.default_rng(31085)`).
* Seçim kuralları: Seq 01'den önce 5 doğrulama, sonra 10 geliştirme karesi seçilir ve iki grup arasında en az 5 ara kare bırakılır. Seq 02'den 3 blok × 5 ardışık test karesi seçilir ve bloklar arasında da en az 5 ara kare bulunur. Koşul sağlanmazsa aynı RNG akışından deterministik olarak yeniden çekilir.
* Seçilen kareler `selected_frames.csv` dosyasındadır. Seçim, algoritma geliştirilmeden önce yapılmıştır.
* Referans sayı, `TRA/man_trackTTT.tif` içindeki farklı pozitif etiket sayısıdır.
* Çalıştırılan ortamın kütüphane sürümleri `environment.json` dosyasına yazılır.

## Yazılım gereksinimleri
Python ≥ 3.10; `requirements.txt` (OpenCV, NumPy, SciPy, scikit-image, Matplotlib, pandas, tifffile). Google Colab'da ek kurulum gerekmez.

## Çalıştırma
### Google Colab
1. Colab'da **File → Open notebook → GitHub** yolunu izleyip bu depodaki `main.ipynb` dosyasını açın.
2. **BÖLÜM A** için `# BÖLÜM B` başlığına tıklayın ve **Runtime → Run before** komutunu çalıştırın (~5–10 dk). Bu bölüm veriyi indirir, kareleri seçer, deneyleri ve doğrulamayı yapar, nihai pipeline'ı kilitler ve test tahminlerini **referansları okumadan** kaydeder.
3. Sol paneldeki **Files** bölümünden `final_pipeline.json`, `test_predictions_locked.csv` ve `test_lock.sha256` dosyalarını indirin ve GitHub'a **ayrı bir commit** ile yükleyin (*"Test tahminleri kilitlendi"*).
4. **BÖLÜM B** için **Runtime → Run after** komutunu çalıştırın. Bu bölüm kilidin SHA-256 özetini doğrular, ardından test referanslarını okur, metrikleri, zamansal analizi, hata analizini ve şekilleri üretir. Sonunda `results_bundle.zip` iner.
5. Kalan CSV dosyalarını ve `plots/` klasörünü ikinci bir commit ile yükleyin (*"Test değerlendirmesi"*). Notebook'u yüklemeden önce **Edit → Clear all outputs** ile çıktılarını temizleyin.

### Yerel Jupyter
```bash
pip install -r requirements.txt
jupyter notebook main.ipynb      # veya: jupyter nbconvert --to notebook --execute main.ipynb
```

### Sızıntı korumaları
* `read_tra("02", …)`, `test_lock.sha256` oluşmadan çağrılırsa hata verir; bu sayede test referansları kilitten önce okunamaz.
* Kilit hücresi, kilit dosyası zaten varsa çalışmaz; test tahminleri yeniden üretilemez.
* BÖLÜM B, `test_predictions_locked.csv` ve `final_pipeline.json` dosyalarının SHA-256 özetini kontrol eder.
* Oturum yeniden başlarsa 0–6. bölümleri çalıştırıp doğrudan BÖLÜM B'ye geçilebilir; gerekli durum dosyalardan yüklenir.

## Yöntem özeti
Pipeline'ın aşamaları sırayla şunlardır: **yoğunluk dönüşümü → filtreleme → segmentasyon → morfoloji → küçük nesne eleme → (watershed ile ayırma) → bağlı bileşen sayımı.**

* **Baseline:** gri seviye → Otsu → 8-komşuluk bağlı bileşen etiketleme → (geliştirme kümesinde seçilen sabit min_area) → sayım.
* **Kontrollü deneyler** yalnızca geliştirme kümesinde yapılır. Her aşamada yalnızca o aşamadaki yöntem değiştirilir, aşamanın en iyisi bir sonraki aşamada sabit tutulur:
  * yoğunluk dönüşümü: orijinal, contrast stretching, histogram eşitleme, CLAHE, histogram eşleme;
  * filtre: yok, Gaussian 3/5/9, median 5, Laplasyen keskinleştirme, Gaussian+keskinleştirme, unsharp masking;
  * segmentasyon: sabit eşik, Otsu, adaptif Gauss, K-means k=2/3;
  * morfoloji: erosion, dilation, opening (3/7/11), closing (3/7) ve farklı işlem sıraları;
  * ayırma: distance transform + watershed (`min_distance` ve σ taraması).
* **Aday pipeline'lar:**
  * A: ayırma yok;
  * B: A + watershed;
  * C: farklı segmentasyon ailesi + watershed.
* **Seçim kuralı:** doğrulama MAE'si en düşük olan seçilir; eşitlikte sırasıyla RMSE, süre ve karmaşıklığa bakılır.
* Nihai pipeline'ın tam parametreleri `final_pipeline.json` dosyasındadır.

## Üretilen dosyalar
| Dosya | İçerik |
|---|---|
| `selected_frames.csv` | Seçilen kareler, veri grupları, test blokları, seed ve deneme sayıları |
| `data_overview.csv`, `data_overview_summary.csv` | Her karenin boyutu, veri tipi ve yoğunluk istatistikleri |
| `references_dev_val.csv` | Geliştirme ve doğrulama referans sayıları |
| `experiments.csv` | Kontrollü deneyler: yöntem, parametre, dev MAE ± std, RMSE, Bias, süre, konfigürasyon |
| `parameter_sweeps.csv` | Eşik, adaptif blok/C, CLAHE, watershed ve min_area taramaları |
| `validation_results.csv` | Aday pipeline'ların doğrulama sonuçları ve seçim |
| `error_analysis_validation.csv` / `error_analysis_test.csv` | TRA işaretçilerine göre kaçırılan, birleşen, bölünen ve yanlış pozitif sayıları |
| `final_pipeline.json` | Kilitlenmiş nihai pipeline ve baseline konfigürasyonu |
| `test_predictions_locked.csv`, `test_lock.sha256` | Referanslar okunmadan kaydedilen test tahminleri ve kilit özeti |
| `test_results.csv`, `test_summary.csv` | Kare bazında test sonuçları ile MAE, RMSE, Bias, tam doğru sayım oranı ve süre |
| `validation_vs_test.csv`, `temporal_analysis.csv` | Doğrulama–test karşılaştırması ve bloklardaki ardışık sayım değişimleri |
| `environment.json` | Yazılım sürümleri |
| `plots/` | Veri görüntüsü içermeyen grafikler: histogramlar, deney ve tarama grafikleri, Şekil 1 ve Şekil 5. Notebook çalıştırılınca üretilir; şekiller teknik raporda yer alır |
| `figures_private/` | Veri görüntüsü içeren şekiller (Şekil 2, 3, 4, 6). **GitHub'a yüklenmez**, yalnızca raporda kullanılır |
