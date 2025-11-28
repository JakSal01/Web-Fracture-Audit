# 🛡️ WEB FRACTURE AUDIT: OWASP Juice Shop

![Type](https://img.shields.io/badge/Type-Security%20Audit-red)
![Target](https://img.shields.io/badge/Target-OWASP%20Juice%20Shop-orange)
![Methodology](https://img.shields.io/badge/Methodology-OWASP%20Top%2010-blue)
![Status](https://img.shields.io/badge/Status-Compromised-success)

## 📄 Wprowadzenie

**Web Fracture Audit** to dokumentacja techniczna przeprowadzonych testów penetracyjnych (Black-Box) na aplikacji *OWASP Juice Shop*. Projekt miał na celu identyfikację krytycznych punktów "pęknięcia" (fracture points) w zabezpieczeniach aplikacji, prowadzących do wycieku danych, ominięcia uwierzytelniania oraz całkowitego przejęcia kontroli nad systemem.

Analiza skupiała się na ręcznej weryfikacji wektorów ataku, wykorzystując błędy logiczne oraz luki w konfiguracji.

---

## 🧰 Wykorzystane Technologie i Narzędzia

W trakcie audytu wykorzystano następujący stos technologiczny:
* **Burp Suite (Intruder, Repeater):** Przechwytywanie i manipulacja ruchem HTTP.
* **CyberChef:** Analiza i dekodowanie ciągów znaków (Base85).
* **Kali Linux / Browser DevTools:** Środowisko testowe i inspekcja kodu klienckiego.
* **Techniki:** SQL Injection, XSS, IDOR, Mass Assignment, Cryptographic Analysis.

---

## 🔍 Przebieg Audytu i Analiza "Pęknięć" (Fracture Points)

### 1. Rekonesans: Wyciek Informacji (Information Disclosure)
Pierwszym punktem wejścia było zlokalizowanie niezabezpieczonych zasobów. Wykryto otwarty katalog FTP, który pozwalał na wgląd w pliki systemowe aplikacji.

**Dlaczego to zrobiliśmy?**
Pozostawienie domyślnych plików lub backupów (`.bak`, `.yml`) często ujawnia logikę aplikacji lub używane wersje bibliotek, co ułatwia dalsze planowanie ataku.

* **Dowód:** Otwarty indeks plików FTP.
<br>
<img src="Screenshots/ftp_directory_listening_1.png" alt="FTP Listing" width="600">
<br><br>

* **Analiza:** Przegląd plików pod kątem wrażliwych danych.
<br>
<img src="Screenshots/ftp_directory_listening_2.png" alt="FTP Analysis" width="600">

---

### 2. Kryptografia: Łamanie Niestandardowego Kodowania
Weryfikacja mechanizmu kuponów rabatowych ujawniła, że aplikacja nie używa losowych tokenów, lecz koduje dane w formacie **Base85**.

**Dlaczego to zrobiliśmy?**
Zrozumienie algorytmu generowania kuponów pozwala napastnikowi na stworzenie własnych, fałszywych kodów (Forgery), co prowadzi do bezpośrednich strat finansowych sklepu.

* **Identyfikacja:** Znalezienie podejrzanego ciągu znaków w interfejsie.
<br>
<img src="Screenshots/Base85_coupon.png" alt="Base85 Coupon" width="600">
<br><br>

* **Eksploitacja:** Dekodowanie i modyfikacja wartości przy użyciu CyberChef.
<br>
<img src="Screenshots/decoding.png" alt="CyberChef Decoding" width="600">

---

### 3. Injekcje: Przełamanie Barier Danych (Injection Attacks)
Najbardziej krytyczne błędy polegające na braku sanityzacji danych wejściowych.

#### A. SQL Injection (Ominięcie Uwierzytelniania)
Wykorzystano podatność w panelu logowania. Wstrzyknięcie payloadu `' OR 1=1--` w polu email pozwoliło na zalogowanie się jako administrator bez hasła.

* **Dowód:** Dostęp do konta administratora.
<br>
<img src="Screenshots/sql_injection.png" alt="SQL Injection" width="600">

#### B. Cross-Site Scripting (XSS)
Wstrzyknięcie złośliwego kodu JavaScript (`<iframe src="javascript:alert('xss')">`) w wyszukiwarce. Kod został wykonany po stronie klienta.

* **Dowód:** Wywołanie nieautoryzowanego alertu w przeglądarce.
<br>
<img src="Screenshots/xss_alert.png" alt="XSS Alert" width="600">

---

### 4. Kontrola Dostępu: Błędy Logiki Biznesowej

#### A. IDOR (Insecure Direct Object Reference)
Zidentyfikowano brak weryfikacji uprawnień przy odwoływaniu się do obiektów (koszyków) po ID. Zmieniając numer ID koszyka, uzyskano dostęp do zamówień innych użytkowników.

**Dlaczego to działa?**
Serwer "ufa" numerowi ID przesłanemu przez klienta, nie sprawdzając, czy zalogowany użytkownik jest właścicielem tego zasobu.

* **Atak:** Użycie Burp Intruder do enumeracji ID koszyków.
<br>
<img src="Screenshots/burp_intruder.png" alt="Burp Intruder" width="600">
<br><br>

* **Rezultat:** Przejęcie koszyka ofiary ("haker_basket").
<br>
<img src="Screenshots/haker_basket.png" alt="Haker Basket" width="600">
<br><br>

* **Szczegóły:** Podgląd zawartości cudzego zamówienia.
<br>
<img src="Screenshots/idor_basket.png" alt="IDOR Basket" width="600">

#### B. Mass Assignment (Eskalacja Uprawnień)
Podczas rejestracji przechwycono żądanie JSON i dodano parametr `"role": "admin"`, który nie był dostępny w formularzu GUI, ale został przetworzony przez backend.

**Dlaczego to zrobiliśmy?**
Jest to test na to, czy API filtruje parametry wejściowe. Błąd ten pozwolił na natychmiastowe utworzenie konta z pełnymi uprawnieniami administracyjnymi.

* **Dowód:** Utworzenie konta admina poprzez manipulację API.
<br>
<img src="Screenshots/mass_assignment_admin.png" alt="Mass Assignment" width="600">

---

## 🏆 Wyniki Audytu

Projekt **Web Fracture Audit** zakończył się pełnym sukcesem. Wszystkie zidentyfikowane wektory ataku zostały potwierdzone, a systemy bezpieczeństwa aplikacji – przełamane.

* **Status Scoreboard:** Potwierdzenie rozwiązania wyzwań.
<br>
<img src="Screenshots/scoreboard_proof.png" alt="Scoreboard Proof" width="600">

---

### 🛡️ Rekomendacje (Remediation)
W celu zabezpieczenia aplikacji przed powyższymi atakami zaleca się:
1.  Wdrożenie **Prepared Statements** (ochrona przed SQLi).
2.  Stosowanie **Context-aware Output Encoding** (ochrona przed XSS).
3.  Weryfikację uprawnień dostępu do każdego obiektu po stronie serwera (ochrona przed IDOR).
4.  Używanie **DTO (Data Transfer Objects)** w API, aby ściśle definiować przyjmowane pola (ochrona przed Mass Assignment).

---
> *Disclaimer: Niniejszy raport został sporządzony wyłącznie w celach edukacyjnych na środowisku treningowym OWASP Juice Shop.*
