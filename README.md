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
