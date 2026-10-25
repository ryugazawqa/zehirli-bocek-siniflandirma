# 🐝 Zehirli / Zehirsiz Böcek Sınıflandırma

Görüntü işleme dersi projesi. Bir böcek fotoğrafından, böceğin insan için **zehirli** (sokarak/ısırarak zehir veren ya da zehirli kıl/salgıyla zarar veren) olup olmadığını tahmin eden derin öğrenme modelleri.

## Veri seti

Fotoğraflar **iNaturalist API**'den çekildi:

- Yalnızca **Research Grade** (tanımı topluluk tarafından doğrulanmış) gözlemler
- Yalnızca açık lisanslı fotoğraflar: CC0, CC-BY, CC-BY-NC
- **38 grup** (aile/cins seviyesinde): 15 zehirli, 23 zehirsiz, ~1.800 farklı takson
- Tırtıl gruplarında (ör. Thaumetopoea, Megalopygidae, Limacodidae) yalnızca **larva** fotoğrafları
- Her gruptan 190 fotoğraf, toplam **~7.220 fotoğraf**

Modelin "tırtılsa zehirlidir" gibi kestirmeler öğrenmemesi için her zehirli grubun yanına ona **benzeyen zehirsiz** bir grup eklendi:

| Zehirli | Benzer zehirsiz |
|---|---|
| Vespidae (eşek arısı) | Syrphidae (arı taklitçisi sinek) |
| Apis mellifera (bal arısı) | Meliponini (iğnesiz arı) |
| Solenopsis, Pogonomyrmex (iğneli karınca) | Formicinae (iğnesiz karınca) |
| Thaumetopoea (keseböceği tırtılı) | Notodontidae tırtılları |
| Megalopygidae (kedi tırtılı) | Sphingidae tırtılları |
| Paederus | Staphylinidae (Paederus hariç) |
| Reduviidae (suikastçı böcek) | Pentatomidae (kalkan böceği) |
| Meloidae (kabarcık böceği) | Chrysomelidae (yaprak böceği) |

## Yöntem

1. **Ön işleme:** 224×224 merkez kırpma, ImageNet normalizasyonu
2. **Veri artırma:** Yatay çevirme, rastgele kırp-yakınlaştır, parlaklık/kontrast ±%20. Uyarı renkleri ayırt edici olduğu için renk tonu değiştirilmedi.
3. **Bölme:** %80 eğitim / %10 doğrulama / %10 test. Ayrıca **görülmemiş türler** testi: bazı türler eğitime hiç sokulmadı ve modelin genelleme yeteneğini ölçmek için kullanıldı.
4. **Dengeli örnekleme:** Her batch %50 zehirli / %50 zehirsiz, sınıf içinde gruplar eşit ağırlıkta.
5. **Transfer öğrenme:** ImageNet üzerinde önceden eğitilmiş modeller ince ayar ile eğitildi (AdamW, OneCycleLR, label smoothing 0.1, AMP, 10 epoch).
6. **İki yaklaşım:**
   - **Model A:** Doğrudan ikili sınıflandırma (zehirli / zehirsiz)
   - **Model B (hiyerarşik):** Önce 38 gruptan hangisi olduğu tahmin edilir, P(zehirli) = zehirli grupların olasılıklarının toplamı
7. **TTA:** Normal ve aynalanmış görüntü tahminlerinin ortalaması
8. **Eşik analizi:** F1-en iyi eşik ve zehirlilerin ≥%90'ını yakalayan "güvenli" eşik
9. **Klasik ML karşılaştırması:** CNN özellik vektörleri üzerinde Random Forest, SVM, Lojistik Regresyon
10. **Grad-CAM:** Modelin fotoğrafın neresine baktığının görselleştirilmesi
11. **Klasik görüntü işleme (OpenCV):** Piksel matrisi, RGB kanalları, histogram ve histogram eşitleme / CLAHE, gürültü filtreleri (ortalama, Gauss, medyan, bilateral) ve PSNR, evrişim çekirdekleri ve kenar bulma (Sobel, Laplace, Canny), HSV uyarı rengi maskesi, morfolojik işlemler, Otsu ve GrabCut segmentasyonu, kontur

## Sonuçlar

### Omurga karşılaştırması (Model B, TTA)

| Omurga | Test Doğruluk | Test F1 | Recall (zehirli) | ROC-AUC | Görülmemiş tür Doğruluk | Görülmemiş tür F1 |
|---|---|---|---|---|---|---|
| EfficientNet-B0 | 0.883 | 0.850 | 0.828 | 0.953 | 0.808 | 0.723 |
| EfficientNet-B3 | 0.829 | 0.787 | 0.785 | 0.916 | 0.790 | 0.705 |
| **ConvNeXt-Tiny** | **0.923** | **0.905** | **0.908** | **0.977** | **0.869** | **0.813** |

### Yaklaşım karşılaştırması (EfficientNet-B0 özellikleri, test seti)

| Model | Doğruluk | F1 | Precision | Recall |
|---|---|---|---|---|
| Model B (hiyerarşik) | 0.874 | 0.833 | 0.884 | 0.788 |
| A+B birleşimi | 0.844 | 0.789 | 0.863 | 0.727 |
| Random Forest | 0.840 | 0.774 | 0.890 | 0.685 |
| Lojistik Regresyon | 0.834 | 0.789 | 0.802 | 0.777 |
| Model A (doğrudan) | 0.806 | 0.735 | 0.810 | 0.673 |
| Linear SVM | 0.752 | 0.584 | 0.890 | 0.435 |

**Bulgular:**
- Hiyerarşik yaklaşım (önce grup, sonra zehir durumu) doğrudan ikili sınıflandırmadan belirgin şekilde daha iyi.
- ConvNeXt-Tiny ek veri kullanılmadan en iyi sonucu verdi.
- En zor çift **Reduviidae ↔ Pentatomidae** oldu.

### Klasik görüntü işleme: renk tek başına yeter mi?

4 benzer çiftte (200 fotoğraf) böcek GrabCut ile arka plandan ayrıldı ve yalnızca böceğin üzerindeki uyarı rengi (sarı + kırmızı, HSV maskesi) oranı ölçüldü:

| Çift | Zehirlide uyarı rengi | Zehirsizde uyarı rengi |
|---|---|---|
| Vespidae / Syrphidae | %23 | %48 |
| Apis mellifera / Meliponini | %48 | %39 |
| Megalopygidae / Sphingidae | %15 | %20 |
| Reduviidae / Pentatomidae | %19 | %21 |

"Uyarı rengi oranı > eşik ise zehirli" kuralı en iyi eşikte bile **~%50** doğrulukta kaldı (yazı-tura düzeyi). Ayrı `goruntu_isleme.ipynb` notebook'unda 5 farklı çiftin (Vespidae/Syrphidae, Meloidae/Chrysomelidae, Megalopygidae/Saturniinae, Limacodidae/Sphingidae, Reduviidae/Pentatomidae) en çok beğenilen 250 fotoğrafıyla tekrarlandığında da sonuç **~%55** oldu. Arı taklitçisi Syrphidae, eşek arısından daha sarı çıktı. Bu nedenle şekil, doku ve rengi birlikte öğrenen CNN'ler kullanıldı (ConvNeXt-Tiny %92,3).

## Dosyalar

| Notebook | İçerik |
|---|---|
| `veri_toplama.ipynb` | İlk 24 grubun iNaturalist'ten toplanması |
| `veri_ek.ipynb` | Benzer görünümlü 14 ek grubun toplanması |
| `egitim_v3.ipynb` | Veri çekme, Model A/B eğitimi, TTA, eşik analizi, RF/SVM/LR, Grad-CAM |
| `egitim_v4_omurga.ipynb` | EfficientNet-B0 / B3 / ConvNeXt-Tiny karşılaştırması + klasik görüntü işleme bölümü (çıktılı) |
| `goruntu_isleme.ipynb` | Yalnızca görüntü işleme (çıktılı, GPU gerekmez): piksel matrisi, RGB, histogram, CLAHE, filtreler ve PSNR, evrişim çekirdekleri, Sobel/Canny, HSV maskesi, morfoloji, Otsu/GrabCut, renk analizi |

## Çalıştırma

Notebook'lar Kaggle'da (GPU T4) hazırlandı. `egitim_v3.ipynb` ve `egitim_v4_omurga.ipynb` girdi veri seti gerektirmez, fotoğrafları doğrudan iNaturalist'ten çeker. Kaggle'da **Internet** ve **GPU** açık olmalıdır.

Kullanılan kütüphaneler: `torch`, `torchvision`, `opencv-python`, `scikit-learn`, `pandas`, `numpy`, `matplotlib`, `Pillow`, `requests`

## Lisans ve atıf

Fotoğraflar iNaturalist kullanıcılarına aittir ve ilgili Creative Commons lisanslarıyla kullanılmıştır. Her fotoğrafın atıf bilgisi notebook çıktısındaki `metadata.csv` dosyasında yer alır. Fotoğraflar bu repoda bulunmaz.
