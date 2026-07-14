# Ekonometrik / Ekonomik Modelleme Notebook'u

Konfigürasyon odaklı, yeniden kullanılabilir bir ekonometrik modelleme çatısı.
Kodun büyük kısmına dokunmadan, yalnızca en üstteki **`CONFIG`** bloğunu
değiştirerek farklı veri setleri ve değişkenler üzerinde çok sayıda ekonometrik
modeli kurabilir, test edebilir, karşılaştırabilir, en iyi modeli seçebilir,
forecast üretebilir ve tüm çıktıları düzenli bir Excel dosyasına yazabilirsiniz.

Değişken isimleri **hard-code edilmemiştir**; tamamı `CONFIG` üzerinden yönetilir.
Aynı notebook'u kredi faizi, taşıt fiyatı, kur, politika faizi, portföy büyüklüğü,
fon getirisi veya başka herhangi bir ekonomik zaman serisi için kullanabilirsiniz.

## Desteklenen modeller

OLS · ADL · Distributed Lag · ARDL · ECM (Engle-Granger) · VAR · VECM (Johansen) ·
SARIMAX · Ridge · Lasso · ElasticNet

## Kurulum

```bash
pip install -r requirements.txt
```

Opsiyonel paketler (`arch`, `pmdarima`, `linearmodels`) yoksa notebook çökmez;
ilgili gelişmiş özellikler devre dışı kalır ve açık bir uyarı verilir.

## Kullanım

1. `ekonometrik_modelleme.ipynb` dosyasını açın.
2. En üstteki **`CONFIG`** bloğunu kendi verinize göre düzenleyin:
   - `input_file`, `sheet_name`, `date_col`, `target_col`
   - `frequency` (`D`/`W`/`M`/`Q`/`A` ya da doğrudan `MS`/`ME`/`QS`/`QE`/`YS`/`YE`),
     `sample_start/end`, `forecast_start/end`
   - `target_transform` ve `exog_transforms` (level / log / diff / logdiff / pct_change)
   - `models_to_run` ve her model için `model_specs`
   - `forecast_scenarios` (constant / growth / path / linear / shock)
   - `test_size`, `selection_metric`, `expected_signs`, `cov_type`
   - `rolling` (opsiyonel kayan pencere OLS — zamanla değişen katsayılar)
3. Notebook'u **baştan sona** çalıştırın (Kernel → Restart & Run All).

> `input_file` bulunamazsa notebook otomatik olarak sentetik bir örnek veri seti
> (`y`, `usdtry`, `policy_rate`) üretir; böylece "tak-çalıştır" mantığında hemen
> çalışır. Kendi verinizi kullanmak için `CONFIG["input_file"]` alanını
> gerçek Excel dosyanızla değiştirmeniz yeterlidir.

## Çıktılar

- **`model_outputs.xlsx`** — düzenli Excel raporu:
  `00_README`, `01_RAW_DATA`, `02_TRANSFORMED_DATA`, `03_MODEL_COMPARISON`,
  `04_BEST_MODEL_SUMMARY`, `05_COEFFICIENTS_ALL`, `06_DIAGNOSTICS_ALL`,
  `07_VIF_ALL`, `08_FORECASTS`, `09_SCENARIOS`, `10_ERRORS_WARNINGS` ve
  kayan pencere açıksa `11_ROLLING_OLS`.
- **`plots/`** — hedef seri, gerçek vs fitted, residual, histogram, actual vs
  forecast, model karşılaştırma ve (açıksa) kayan pencere katsayı grafikleri (PNG).

### Kayan pencere (rolling) OLS

`CONFIG["rolling"]` ile açılan **opsiyonel** analiz katmanıdır; ana model
seçim/forecast akışını etkilemez (`enabled: False` iken hiç çalışmaz). Seçilen
regresyon spesifikasyonu sabit genişlikte bir pencereyle (`window`) kaydırılarak
her dönem yeniden tahmin edilir ve **katsayıların zaman içindeki değişimi**
`11_ROLLING_OLS` sayfasına + ±2 standart hata bantlı grafiğe yazılır.

```python
CONFIG["rolling"] = {
    "enabled": True,
    "model": "OLS",          # OLS / ADL / DISTRIBUTED_LAG spec'lerinden biri
    "window": 12,            # kayan pencere genişliği (gözlem)
    "min_nobs": 12,
    "step": 1,
    "mode": "coefficients",
}
```
- Notebook içinde **otomatik Türkçe özet yorum** ve tanı testi yorumları.

## Sağlamlık

- Hata veren model tüm süreci durdurmaz; hata `10_ERRORS_WARNINGS` sayfasına yazılır.
- Eksik değer, eksik/tekrar eden tarih, çok küçük örneklem, sıfır/negatif log
  değeri ve MAPE'de sıfır gerçek değer gibi durumlar hata değil **uyarı** üretir.
- **Frekans/tarih hizalaması anchor-farkındalıdır.** Frekans önce veriden
  (`pd.infer_freq`) çıkarılır; çıkarılamazsa `M`/`Q`/`A` kısayolu verinin ilk
  gününe göre dönem-başı (`MS`) veya dönem-sonu (`ME`) olarak hizalanır. Böylece
  **ay-başı** (ör. `2014-01-01`) veya **ay-sonu** (ör. `2014-01-31`) tarihli veriler
  sorunsuz çalışır — daha önce ay-başı veri `ME` takvimine sabitlenip tüm değerleri
  NaN'a düşürdüğü için `prepared_df` boş kalabiliyordu; bu giderildi. Ek güvenlik:
  üretilen frekans takvimi veriyle yeterince örtüşmezse yeniden örnekleme yapılmaz,
  orijinal tarih index'i korunur (veri asla NaN'a düşmez).
- Bilgi kriterleri (AIC/BIC) tüm model ailelerinde aynı ölçeğe getirilir; böylece
  BIC/AIC ile model seçimi anlamlı olur.

---

# BÖLÜM 2 — Heteroskedastisite Teşhis ve İleri Ekonometrik Analiz

Notebook'un ikinci bölümü, **heteroskedastisite tespit edildiğinde yalnızca HC3
uygulamak yerine sorunun kaynağını teşhis eden ve uygun alternatifleri
karşılaştırmalı sınayan** opsiyonel bir katmandır. `CONFIG["advanced"]["enabled"]`
ile açılır/kapanır; **ana akışı (Bölüm 1) etkilemez** ve ayrı bir Excel dosyası
(`econometric_diagnosis_output.xlsx`, 20 sayfa) ile ayrı grafikler (`plots_advanced/`)
üretir. Kapalıyken hiç çalışmaz.

### Ne yapar
- **Durağanlık:** Her değişken için ADF + KPSS (düzey ve fark); I(0)/I(1)/I(2)
  otomatik yorumu. **ARDL'ye I(2) değişken girmemelidir** — I(2) şüphesi uyarı verir.
- **Teşhis (karar mekanizması):** BP/White/Goldfeld–Quandt + **ARCH-LM** + BG birlikte
  değerlendirilir ve sorunun kaynağı ayrılır:
  - *Durum 1/4 (heteroskedastisite, otokorelasyon yok):* HC3 raporlanır **ve** kaynak
    araştırılır (log/fark dönüşümü, yapısal kırılma kuklaları, WLS, FGLS).
  - *Durum 2 (otokorelasyon + heteroskedastisite):* **HAC / Newey-West** standart hataları.
  - *Durum 3 (ARCH-LM anlamlı):* Ortalama denklem korunur, **artıklar üzerinde
    ARCH/GARCH** (ARCH(1)/GARCH(1,1)/… AIC-BIC ile) kurulur.
- **Remediation modelleri:** ARDL_Level / LogY / LogLog / Difference / BreakAdjusted,
  WLS (floor+winsorization'lı birkaç ağırlık şeması), iteratif FGLS.
- **ARDL bounds testi** (UECM), **kısa/uzun dönem etkiler** (delta yöntemiyle SE/CI),
  **yapısal kırılma** (manuel + artık sıçraması + Chow + CUSUM), **aykırı/etkili gözlem**
  (Cook's D, leverage, DFBETAs), **ECM** hata düzeltme katsayısı.
- **Şeffaf temiz-skor (0–100):** Her bileşen (işaret, I(2), otokorelasyon, stabilite,
  ECM, bounds, forecast, AIC/BIC, hetero yönetimi, VIF) ayrı kolonda gösterilir.

### CONFIG["advanced"] alanları
```python
CONFIG["advanced"] = {
    "enabled": True,
    "base_model": "OLS",           # bağımsız değişken + lag kaynağı (OLS/ADL/DISTRIBUTED_LAG)
    "independent_variables": [],   # boşsa base_model spec'inin exog'u kullanılır
    "robust_covariance": "HC3",    # nonrobust/HC0/HC1/HC3/HAC
    "significance_level": 0.05,
    "max_lag_y": 6, "max_lag_x": 6, "lag_selection_criterion": "BIC",
    "test_structural_breaks": True, "manual_break_dates": [],  # ör. ["2018-08-01"]
    "test_wls": True, "test_fgls": True, "test_arch_garch": True,
    "goldfeld_quandt": True, "make_log_variants": True,
    "bias_correction": True,       # log->seviye forecast: exp(f + 0.5*sigma^2)
    "fgls_max_iter": 5,
    "output_excel_path": "econometric_diagnosis_output.xlsx",
    "make_plots": True, "plot_dir": "plots_advanced",
}
```

### HC3 / HAC / WLS / FGLS / GARCH sonuçları nasıl yorumlanır
- **HC3:** Yalnızca standart hataları heteroskedastisiteye karşı dayanıklı yapar;
  **artıkların varyans yapısını değiştirmez.** Bu nedenle HC3 sonrası BP/White'ın
  anlamlı kalması "düzeltme başarısız" demek **değildir**. Otokorelasyon yokken,
  varyansı modellemek istemiyorsanız çıkarım için uygundur.
- **HAC (Newey-West):** Hem otokorelasyon hem heteroskedastisite varken standart
  hataları ikisine karşı dayanıklı yapar (nokta tahminleri değişmez).
- **WLS:** Varyans yapısı makul tahmin edilebiliyorsa etkinlik kazandırır; **yanlış
  ağırlık seçimi sonuçları bozabilir** — bu yüzden birkaç ağırlık şeması denenip
  BP-sonrası en iyisi seçilir ve rapor bu riski açıkça belirtir.
- **FGLS:** Yardımcı varyans regresyonuyla ağırlık tahmin edip iteratif çalışır;
  `Heteroskedasticity` sayfasındaki `BP_p_sonrasi` ile heteroskedastisitenin fiilen
  azalıp azalmadığı görülür.
- **GARCH:** Yalnızca **ARCH-LM anlamlıysa** çalışır. Ortalama denklem korunur;
  artıklardaki koşullu varyans modellenir. Bu durumda **forecast aralıkları sabit
  değil dönemsel değişen varyansa** göre yorumlanmalıdır.

### Excel sayfaları (econometric_diagnosis_output.xlsx)
`Config`, `Data_Quality`, `Descriptive_Stats`, `Stationarity_Tests`, `Lag_Search`,
`Model_Comparison`, `Coefficients`, `Robust_Coefficients`, `Short_Run_Effects`,
`Long_Run_Effects`, `Bounds_Test`, `Diagnostics`, `Heteroskedasticity`, `ARCH_GARCH`,
`Structural_Breaks`, `Outliers_Influence`, `Forecast_Scenarios`, `Forecast_Output`,
`Automatic_Interpretation`, `Errors_Warnings`.

> **Ek paket:** GARCH için `arch` gerekir (`requirements.txt`'e eklendi). Kurulu
> değilse ARCH/GARCH adımı atlanır, diğer tüm ileri analizler çalışır.

### Otomatik yorum dili
Yorumlar istatistiksel olarak doğru dille üretilir; örneğin p>0,05 için
*"otokorelasyon reddedilememektedir"* gibi hatalı ifade yerine **"otokorelasyon
bulunduğuna ilişkin yeterli kanıt yoktur"** kullanılır.

---

# BÖLÜM 3 — ARDL Heteroskedastisite Denetimi (gerçek statsmodels ARDL)

Bölüm 2'deki "advanced" katman ARDL'yi elle kurulan bir OLS tasarımıyla
yaklaşıklıyordu ve `ardl_order`/`ar_lags`/`dl_lags`/`causal` gibi yapıları
göstermiyordu. **Bölüm 3**, `CONFIG["ardl_audit"]` ile açılan ve **gerçek
`statsmodels.tsa.ardl.ARDL`** nesnesi kuran teknik bir denetim katmanıdır. Ana
akışı etkilemez; `ardl_hetero_audit.xlsx` (11 sayfa) + `plots_ardl_audit/` üretir.

### Neyi denetler
- **Gerçek ARDL yapısı:** `ardl_order`, `ar_lags`, `dl_lags` (değişken bazlı),
  `causal`, `trend` açıkça yazdırılır.
- **Sabit (ortak) etkin örneklemli lag seçimi:** Tüm `(p,q)` adayları **aynı**
  örneklem üzerinde BIC ile karşılaştırılır (farklı `p` farklı başlangıç gözlemi
  düşürdüğü için bu şarttır) ve `ardl_select_order` seçimiyle çapraz doğrulanır.
- **HC3 vs klasik:** Katsayı/fitted/artık **aynı**, yalnızca standart hata ve
  p-değerleri farklı — bu koşullar PASS/FAIL kontrol tablosunda doğrulanır.
- **Breusch–Pagan tam ARDL tasarım matrisiyle**, hem **klasik** hem **Koenker**
  (`robust=True`) biçiminde; LM ve F istatistikleri ayrı raporlanır ve ayrı
  yorumlanır (küçük/orta örneklemde LM aşırı reddedebilir → önce Koenker-F).
- **Çoklu-lag ARCH-LM** (1, 3, 6, 12; yetersiz gözlemli lag otomatik atlanır).
- **Güvenli White:** serbestlik derecesi yetersizse `SKIPPED` olarak raporlanır.
- **Alternatif spesifikasyonlar** (LogY, Difference, kırılma/pulse dummy'li ARDL,
  WLS, FGLS) — her biri sonrası tüm testler yeniden çalıştırılır.
- **Yapısal kırılma / etkili gözlem** (CUSUM, Cook's D, leverage, DFBETAs).
- **PASS/FAIL/WARNING/SKIPPED kontrol tablosu** ve **net kullanılabilirlik kararı**.

### CONFIG["ardl_audit"] alanları
```python
CONFIG["ardl_audit"] = {
    "enabled": True,
    "dependent": None,          # None -> target_col
    "independent": None,        # None -> advanced.independent_variables / base_model exog
    "max_p": 8, "max_q": 4, "ic": "bic",
    "trend": "c",               # n/c/ct
    "causal": False,            # cari dönem x dahil mi
    "alpha": 0.05,
    "arch_lags": [1, 3, 6, 12],
    "manual_break_dates": [],
    "output_excel_path": "ardl_hetero_audit.xlsx",
    "make_plots": True, "plot_dir": "plots_ardl_audit",
}
```

### Kritik ilke
HC3 heteroskedastisiteyi **ortadan kaldırmaz**; model artıklarını ve hata
varyansını değiştirmez, yalnızca katsayı standart hatalarını dayanıklı yapar.
Bu yüzden HC3 sonrası BP anlamlı kalabilir — bu **başarısızlık değildir**.
Heteroskedastisiteyi *fiilen* gidermek için dönüşüm, WLS/FGLS, yapısal kırılma
düzeltmesi veya (ARCH varsa) ARCH/GARCH gerekir.

> **Ek not — private attribute:** ARDL tam tasarım matrisi öncelikle `model._x`
> (private) üzerinden alınır; sürüm farkına karşı `ar_lags`/`dl_lags`/`trend`'den
> **manuel yeniden kurulum** fallback'i vardır ve kaynak `ARDL_Structure`
> sayfasında/çıktıda açıkça belirtilir. `resid`↔`X` satır ve kolon boyutları
> `assert` ile doğrulanır.
