# Primena neuronskih mreža za detekciju DoS napada

**Student:** Mihailo Obradović  
**Indeks:** 2022/0106  
**Predmet:** Duboko učenje i neuronske mreže  

---

## 1. Uvod i opis problema

Kod klasičnih pristupa detekciji mrežnih napada, poput pravila zasnovanih na pragovima ili jednostavnih statističkih metoda, pretpostavka je relativno jednostavna. Na primer, moglo bi se reći da što je veći broj paketa u kratkom vremenskom periodu, veća je šansa da je u pitanju napad.

U realnosti, DoS napadi su mnogo suptilniji. Napad ne mora da generiše ogromnu količinu saobraćaja da bi onesposobio server. Neki napadi, poput Slowloris-a, šalju mali broj zahteva ali ih namerno drže otvorenim kako bi iscrpili resurse servera. Dakle, anomalija nije uvek u visokom obimu saobraćaja, već u specifičnoj kombinaciji karakteristika – trajanju konekcije, broju bajtova, protokolu, vremenskim intervalima između paketa.

Glavni cilj ovog rada jeste primena veštačke neuronske mreže za automatsku detekciju DoS napada na osnovu karakteristika mrežnog saobraćaja, uz analizu uticaja različitih tehnika preprocesiranja na performanse modela.

Detekcija napada zahteva balansiranje između dva suprotstavljena cilja: maksimizacije bezbednosti (hvatanja što većeg broja napada) i minimizacije lažnih uzbuna (izbegavanje blokiranja legitimnog saobraćaja). Svaki propušten napad može izazvati nedostupnost servisa, dok svaki lažni alarm može ometati legitimne korisnike.

Kroz razvoj i analizu modela, istražićemo kako različite tehnike preprocesiranja utiču na ovaj kompromis, sa posebnim fokusom na:

- Uticaj balansiranja klasa putem SMOTE algoritma
- Poređenje StandardScaler i RobustScaler normalizacije na podacima sa visokim skewnessem
- Izbor odgovarajućih metrika evaluacije za nebalansirane skupove podataka

## 2. Struktura projekta

Projekat je organizovan u dva osnovna dela – priprema i obrada podataka, i razvoj modela za detekciju napada. Svaki korak pipeline-a je jasno odvojen u notebook-u kako bi se omogućila preglednost i lakše razumevanje toka analize.

Notebook je podeljen u sledeće celine:

* Učitavanje i osnovna analiza podataka
* Analiza distribucije klasa i feature-a
* Preprocesiranje – binarna konverzija, train/test split, normalizacija, SMOTE
* Analiza skaliranja – poređenje StandardScaler i RobustScaler
* Definisanje i trening neuronske mreže
* Evaluacija modela – confusion matrix, classification report, krive učenja

---

## 3. Podaci

**Izvor:** CICIDS2017 dataset – DoS Wednesday  
**Format:** Parquet  
**Veličina:** 584,991 redova, 77 feature-a, 0 null vrednosti  

### Distribucija klasa

| Klasa | Broj primera |
|---|---|
| Benign | 391,235 |
| DoS Hulk | 172,846 |
| DoS GoldenEye | 10,286 |
| DoS Slowloris | 5,385 |
| DoS Slowhttptest | 5,228 |
| Heartbleed | 11 |

Problem je formulisan kao **binarna klasifikacija** – Benign (0) vs. Napad (1). 

### Preprocesiranje

- Binarna konverzija Label kolone
- Train/test split: 80% / 20%, random_state=42
- Normalizacija: StandardScaler
- Balansiranje klasa: SMOTE (313,029 : 313,029)

### Analiza skaliranja

Analiza distribucije feature-a pokazala je da 58 od 77 kolona ima visok skewness (>1), što teorijski sugeriše korišćenje RobustScaler-a. Međutim, empirijsko testiranje oba skalera dalo je sledeće rezultate:

| Metrika | StandardScaler | RobustScaler |
|---|---|---|
| F1 (Napad) | 0.98 | 0.93 |
| Recall (Napad) | 1.00 | 0.87 |
| FN (propušteni napadi) | 143 | 4,884 |

Na osnovu rezultata odabran je **StandardScaler** – outlieri u mrežnom saobraćaju nose korisnu informaciju za detekciju napada i ne treba ih potiskivati.

---

## 4. Arhitektura modela

Feedforward neuronska mreža implementirana u Keras (TensorFlow backend):

| Sloj | Izlazna dimenzija | Aktivacija |
|---|---|---|
| Dense | 128 | ReLU |
| Dropout | – | 0.3 |
| Dense | 64 | ReLU |
| Dropout | – | 0.3 |
| Dense | 32 | ReLU |
| Dense | 1 | Sigmoid |

- **Optimizer:** Adam  
- **Loss funkcija:** Binary Crossentropy  
- **Ukupno parametara:** 20,353  

---

## 5. Trening

- **Broj epoha:** 10  
- **Batch size:** 512  
- **Validation split:** 20%  

Obe krive (train i validation) konvergiraju tokom treninga bez znakova overfittinga. Val accuracy osciluje zbog stohastične prirode Dropout-a i batch shufflinga, ali generalni trend je pozitivan.

---

## 6. Analiza hiperparametara

Testirani su sledeći hiperparametri:

- **Scaler:** StandardScaler vs RobustScaler – StandardScaler dao bolje rezultate
- **Dropout rate:** 0.3 – sprečava overfitting bez značajnog pada performansi
- **Arhitektura:** opadajući broj neurona po slojevima (128→64→32) omogućava postepenu ekstrakciju apstraktnih obrazaca

---

## 7. Rezultati evaluacije

| Metrika | Benign | Napad |
|---|---|---|
| Precision | 1.00 | 0.97 |
| Recall | 0.98 | 1.00 |
| F1-score | 0.99 | 0.98 |

**Confusion Matrix:**

|  | Predviđeno: Benign | Predviđeno: Napad |
|---|---|---|
| **Stvarno: Benign** | 76,996 | 1,210 |
| **Stvarno: Napad** | 143 | 38,650 |

---

## 8. Diskusija

Model postiže F1-score od 0.98 za klasu Napad, što ga čini pouzdanim za primenu u sistemima za detekciju upada (IDS). 

Ključni nalaz je da 143 False Negative slučaja (propuštenih napada) predstavljaju kritičnu metriku u kontekstu cyber bezbednosti – svaki propušten napad potencijalno može izazvati ozbiljne posledice. Ovo je kompromis koji treba uzeti u obzir pri deploymentu.

Interesantan nalaz je i da StandardScaler, uprkos teorijskoj inferiornosti za podatke sa visokim skewnessem, daje značajno bolje rezultate od RobustScaler-a na ovom datasetu.

---

## 9. Zaključak

Projekat je demonstrirao da feedforward neuronska mreža može efikasno detektovati DoS napade sa visokom preciznošću. Ključne lekcije:

- SMOTE je neophodan za balansiranje neravnomerne distribucije klasa
- Accuracy nije dovoljna metrika – F1-score i Recall su kritični za ovaj tip problema
- Empirijsko testiranje skalera je važnije od teorijskih pretpostavki
- Outlieri u mrežnom saobraćaju nose korisnu informaciju i ne treba ih potiskivati

---

## Tehnologije

- Python 3
- TensorFlow / Keras
- scikit-learn
- imbalanced-learn (SMOTE)
- pandas, numpy
- matplotlib, seaborn
- Google Colab
