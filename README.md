# Phishing (Oltalama) Tespiti: Makine Öğrenmesi Yaklaşımı

Bu proje, oltalama (phishing) saldırılarını tespit etmek amacıyla makine öğrenmesi algoritmalarını kullanır. Projenin bu aşaması; güvenilir kaynaklardan ham URL verilerinin toplanmasını, bu verilerin temizlenip dengelenmesini ve URL'ler üzerinden makine öğrenmesi modellerinin anlayabileceği matematiksel/istatistiksel özelliklerin (Feature Engineering) çıkarılmasını kapsamaktadır.

## 1. Veri Toplama (Data Collection)
Modelin eğitimi için hem zararlı (phishing) hem de meşru (legitimate) URL'ler çeşitli açık kaynaklardan ve API'lerden dinamik olarak toplanmıştır.

**Zararlı URL Kaynakları:**
* **PhishTank:** Güncel ve doğrulanmış oltalama adresleri.
* **OpenPhish:** Gerçek zamanlı phishing veri akışı.
* **URLhaus:** Kötü amaçlı yazılım yayan zararlı URL veritabanı.
* **Certstream & Phishing Army:** Çeşitli GitHub arşivleri ve blocklist kaynakları.

**Meşru URL Kaynakları:**
* **Majestic Million & Tranco List:** Dünyanın en çok ziyaret edilen, güvenilir web siteleri.
* **Cisco Umbrella & Common Crawl:** Alternatif üst düzey alan adı listeleri.
* **Manuel Kürasyon:** Popüler global siteler (Google, YouTube vb.) ve yerel/kurumsal Türk web siteleri (e-devlet, bankalar, haber siteleri).

## 2. Veri Temizleme ve Dengeleme (Data Preprocessing)
Toplanan ham veriler doğrudan kullanıma uygun olmadığı için çeşitli temizleme aşamalarından geçirilmiştir:
* **Duplikasyon Kontrolü:** Tekrar eden URL'ler veri setinden çıkarıldı.
* **Geçersiz Veri Filtrelemesi:** Erişilemeyen, yapısal olarak bozuk veya çok kısa (<12 karakter) URL'ler temizlendi.
* **Domain Çeşitliliği (Undersampling):** Modelin belirli domain'leri ezberlemesini önlemek amacıyla, aynı domain'den gelen URL sayısı sınırlandırıldı.
* **Sınıf Dengelemesi (Class Balancing):** Sınıf dengesizliği (class imbalance) problemini çözmek için azınlık sınıfı baz alınarak veri seti 10.000 Phishing ve 10.000 Meşru URL olacak şekilde eşitlendi (Toplam 20.000 URL).

## 3. Özellik Mühendisliği (Feature Engineering)
Ham URL metinleri üzerinden makine öğrenmesi modellerinin kullanabileceği 30'dan fazla yapısal, istatistiksel ve sözcüksel özellik (feature) çıkarılmıştır. Bu özellikler 5 ana kategoride toplanmıştır:

1. **Temel URL Özellikleri:** URL uzunluğu, domain uzunluğu, path (yol) uzunluğu, HTTPS kullanımı, IP adresi kullanımı, port/fragment varlığı.
2. **Karakter Analizi:** URL içerisindeki özel karakterlerin (nokta, tire, slash, @, = vb.) sayıları, rakam/harf oranları ve genel özel karakter yoğunluğu.
3. **Domain Analizi:** Subdomain sayısı, TLD (Top Level Domain) uzunluğu, domain içerisinde tire veya rakam kullanımı.
4. **Sözcüksel Analiz (Lexical Features):** Phishing saldırılarında sıkça kullanılan şüpheli kelimelerin (`login`, `verify`, `account`, `secure` vb.) URL içerisindeki frekansı.
5. **Entropi Hesaplaması:** URL'nin karmaşıklığını ve rastgeleliğini (randomness) ölçen Shannon Entropi değeri. DGA (Domain Generation Algorithm) ile üretilen sitelerin tespitinde kritik rol oynar.
6. ## 4. Veri Sızıntısını (Data Leakage) Önleme
Modelin ezberlemesini (overfitting) engellemek ve gerçek dünya performansını artırmak için kritik adımlar atılmıştır:
* **Özellik Yumuşatma:** `is_https` veya `is_trusted_domain` gibi modele doğrudan "kopya" verebilecek, kesinlik içermeyen taraflı özellikler çıkarılmıştır.
* **Domain Tabanlı Veri Bölme (Domain-Based Split):** Eğitim (Train) ve Test setleri ayrılırken, aynı alan adına (domain) ait farklı alt sayfaların her iki sette de bulunması engellenmiştir. Böylece modelin URL yapısını ezberlemesi değil, genel oltalama örüntülerini öğrenmesi sağlanmıştır.

## 5. Model Eğitimi ve Hiperparametre Optimizasyonu
Proje kapsamında çeşitli makine öğrenmesi algoritmaları (Random Forest, Gradient Boosting, XGBoost, LightGBM, Logistic Regression, SVM, KNN, Naive Bayes) test edilmiş ve kıyaslanmıştır.
* **Seçilen Modeller:** Eğitim hızı ve ağaç tabanlı yapılarının URL özelliklerini kavramadaki başarısı sebebiyle ağırlıklı olarak **LightGBM** ve **XGBoost** üzerinde durulmuştur.
* **Optimizasyon:** `RandomizedSearchCV` kullanılarak modelin hiperparametreleri (ağaç derinliği, öğrenme hızı, yaprak sayısı vb.) en iyi F1-Skorunu verecek şekilde optimize edilmiştir. Model aşırı öğrenmeye karşı L1/L2 Regularization (Düzenlileştirme) teknikleriyle desteklenmiştir.
* **Ensemble (Topluluk) Öğrenmesi:** Performansı maksimize etmek için LightGBM, XGBoost ve Gradient Boosting modellerinin tahmin olasılıkları (soft-voting) ile birleştirilerek güçlü bir Ensemble model oluşturulmuştur.

## 6. Sınıf Ağırlıklandırma ve Eşik (Threshold) Ayarı
Siber güvenlikte tehlikeli bir bağlantıyı "güvenli" olarak sınıflandırmak (False Negative), güvenli bir bağlantıya "tehlikeli" demekten (False Positive) çok daha risklidir. Bu nedenle projenin odak noktası **Recall (Duyarlılık)** metriğini maksimize etmek olmuştur:
* **Class Weight (Sınıf Ağırlığı):** Phishing sınıfına daha yüksek ceza katsayısı (Örn: 1.5x - 3x) atanarak modelin zararlı bağlantılara daha duyarlı olması sağlanmıştır.
* **Threshold Optimizasyonu:** Varsayılan 0.50 sınıflandırma eşiği, precision ve recall dengesi gözetilerek (örneğin 0.30 - 0.45 seviyelerine) çekilmiş, böylece modelin gözden kaçırdığı oltalama saldırıları (False Negative) minimize edilmiştir.

## 7. Canlı Test ve Çıkarım (Inference)
Eğitilen ve optimize edilen nihai model (`best_model.pkl`), dışarıdan girilen herhangi bir URL'yi anlık olarak analiz edebilecek bir yapıya (`PhishingDetector`) dönüştürülmüştür. 
* Sistem, verilen yeni bir URL'nin özelliklerini milisaniyeler içinde çıkararak modelden geçirir.
* Kullanıcıya sonucun "Phishing" mi yoksa "Meşru" mu olduğunu, modelin güven skoru (confidence) ve URL'nin risk taşıyan özellikleri (şüpheli kelime barındırması, TLD güvenilirliği vb.) ile birlikte interaktif bir şekilde sunar.
* ## 8. Canlı Test ve Çıkarım (Inference System)
Eğitilen ve optimize edilen nihai model (`phishing_model_final.pkl`), dışarıdan girilen herhangi bir URL'yi anlık olarak analiz edebilecek profesyonel bir yapıya (`PhishingDetector` sınıfı) dönüştürülmüştür.

Bu sistemin temel özellikleri şunlardır:
* **Hızlı Özellik Çıkarımı:** Verilen ham URL'den saniyeler içinde 30+ yapısal, istatistiksel ve sözcüksel özelliği otomatik olarak çıkarır.
* **Çift Modlu Çalışma:**
  * **Otomatik/Batch Test Modu:** Belirlenmiş bir URL listesini toplu olarak analiz edip başarı metriklerini raporlar.
  * **İnteraktif Test Modu:** Kullanıcının terminal üzerinden manuel URL girişi yapmasına olanak tanır ve anında güvenlik değerlendirmesi sunar.
* **Detaylı Raporlama:** Sistem sadece "Phishing" veya "Meşru" demekle kalmaz; aynı zamanda:
  * Modelin güven skorunu (Confidence),
  * URL uzunluğu, HTTPS kullanımı, IP adresi varlığı gibi belirgin özellikleri,
  * Şüpheli TLD (Top Level Domain) veya bilindik marka/servis (Örn: Google servisleri, güvenilir alan adları) içerip içermediğini şeffaf bir şekilde raporlar.

## 9. Model Performans Değerlendirmesi ve Sonuçlar
Gelişmiş alan adı (domain) özelliklerinin ve özellik mühendisliğinin eklenmesiyle eğitilen nihai model, yüksek performans metriklerine ulaşmıştır. Model değerlendirmesinde özellikle sahte negatifleri (False Negatives - gözden kaçan oltalama saldırıları) minimize etmek hedeflenmiştir. 

**Model Eğitim ve Seçim Süreci:**
* Logistic Regression, Random Forest, XGBoost ve LightGBM gibi farklı algoritmalar denenmiştir.
* Ağaç tabanlı modellerin (özellikle LightGBM ve XGBoost) özellik etkileşimlerini yakalamadaki üstün başarısı görülmüştür.
* Sınıflandırma eşiği (threshold) varsayılan değer olan 0.5 yerine, recall (duyarlılık) metriğini %92.9 seviyesine çıkaran **0.3** olarak optimize edilmiştir.

**Öne Çıkan Özellikler (Feature Importance):**
Modelin kararlarında en çok ağırlık verdiği özellikler şunlardır:
1. URL uzunluğu
2. Karakter çeşitliliği (Shannon Entropy ve Char Diversity)
3. Domain uzunluğu
4. Özel karakterlerin (nokta, tire vb.) yoğunluğu
5. Alan adında şüpheli kelime barındırması veya bilindik güvenilir domain/TLD listesinde olup olmaması (`is_trusted_domain`, `has_trusted_tld`).

---
> 
