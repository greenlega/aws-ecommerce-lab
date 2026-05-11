# ☁️ AWS High Availability E-Commerce Lab

> Automatyczne wdrożenie skalowalnej, odpornej na awarie infrastruktury e-commerce na Amazon Web Services, zintegrowanej z WordPress + WooCommerce oraz własnym interfejsem frontendowym.

---

## 📐 Architektura

Infrastruktura jest w pełni provisjonowana przez **AWS CloudFormation** i obejmuje następujące komponenty:

| Komponent | Opis |
|---|---|
| **VPC** | Sieć prywatna z podsieciami publicznymi i prywatnymi w wielu strefach dostępności (Multi-AZ) |
| **Application Load Balancer (ALB)** | Rozdziela ruch HTTP między serwery webowe |
| **Auto Scaling Group (ASG)** | Automatycznie skaluje instancje EC2 i zastępuje niesprawne węzły |
| **Amazon EFS** | Współdzielony system plików dla katalogu `wp-content` — każdy serwer serwuje te same pliki |
| **Amazon RDS (MySQL)** | Baza danych w prywatnej podsieci — bezpieczna i niezawodna |

---

## 🚀 Instrukcja wdrożenia

### 1. Provisjonowanie infrastruktury

1. Wejdź do **AWS Console → CloudFormation**.
2. Utwórz nowy stack przy użyciu dostarczonego szablonu (`1.yaml` / `template.json`).
3. Poczekaj, aż status stacka zmieni się na `CREATE_COMPLETE`.

### 2. Konfiguracja bazy danych

1. Przejdź do konsoli **RDS** i skopiuj **Endpoint** instancji bazy danych.
2. Połącz się z aktywną instancją EC2 przez **AWS Systems Manager → Session Manager**.
3. Skonfiguruj połączenie WordPress z bazą danych, edytując plik `wp-config.php`:

```bash
sudo nano /var/www/html/wp-config.php
```

> ⚠️ **Uwaga:** Wartość `DB_HOST` powinna zawierać **wyłącznie** adres endpointu — bez prefiksu `http://` i bez ukośników na końcu.

### 3. Instalacja WordPress i WooCommerce

1. Otwórz w przeglądarce adres DNS Application Load Balancera.
2. Przejdź przez standardowy kreator instalacji WordPress.
3. W panelu WordPress przejdź do **Wtyczki → Dodaj nową**.
4. Wyszukaj, zainstaluj i aktywuj wtyczkę **WooCommerce**.

### 4. Integracja własnego frontendu

1. W panelu WordPress przejdź do **Strony → Dodaj nową**.
2. Nadaj stronie nazwę (np. *„Cloud Store"*).
3. Dodaj blok **Własny HTML** i wklej kod z pliku `cloud-store-template.html` z tego repozytorium.
4. Przejdź do **Ustawienia → Czytanie** i ustaw *„Strona główna wyświetla"* → **Statyczna strona** → wybierz *„Cloud Store"*.
5. Aby własny design zajmował pełny ekran, przejdź do **Wygląd → Edytor (Site Editor)**, znajdź w widoku listy domyślny **Nagłówek** i **Stopkę** motywu, a następnie je usuń.

---

## 🛠️ Napotkane wyzwania i ich rozwiązania

Podczas wdrożenia napotkano kilka rzeczywistych problemów inżynierskich — poniżej opisano ich przyczyny i rozwiązania.

### 🔴 Problem 1: Zawieszenie połączenia z bazą danych

**Objaw:** WordPress nie mógł połączyć się z bazą danych — ekran instalacji zamrażał się.

**Przyczyna:** Endpoint RDS został błędnie sformatowany jako adres URL (np. `http://...rds.amazonaws.com/`). RDS wymaga wyłącznie surowej nazwy hosta DNS.

**Rozwiązanie:** Usunięto prefiks `http://` i końcowe ukośniki z wartości `DB_HOST` w pliku `wp-config.php`.

---

### 🔴 Problem 2: Błędy instalacji WooCommerce (`PCLZIP_ERR_MISSING_FILE` i `504 Gateway Timeout`)

**Objaw:** Instalacja ciężkiej wtyczki WooCommerce kończyła się błędem brakującego pliku lub przekroczeniem czasu.

**Przyczyna:** Klasyczny problem z Load Balancerem. ALB kierował żądanie pobrania do **Serwera A** (plik `.zip` trafiał do jego lokalnego `/tmp/`), a żądanie rozpakowania — do **Serwera B**, który nie posiadał tego pliku. Dodatkowo, rozpakowywanie archiwum bezpośrednio na wolumen EFS (sieciowy, wolniejszy niż lokalny SSD) przekraczało limit czasu ALB wynoszący 60 sekund.

**Rozwiązanie:** Tymczasowo zatrzymano jedną z instancji EC2 (skalowanie do 1 instancji), wymuszając kierowanie całego ruchu przez ALB do jednego serwera. Po pomyślnej instalacji instancję przywrócono.

---

### 🔴 Problem 3: „Biały ekran śmierci" po wklejeniu własnego HTML

**Objaw:** Wklejenie pliku `cloud-store-template.html` skutkowało pustą białą stroną.

**Przyczyna:** Oryginalny plik HTML zawierał pełną strukturę dokumentu (`<!DOCTYPE html>`, `<html>`, `<body>`). Osadzenie tych tagów wewnątrz strony WordPress (która sama generuje te elementy) powodowało konflikty renderowania w przeglądarce.

**Rozwiązanie:** Usunięto wrappers na poziomie dokumentu, pozostawiając wyłącznie tagi `<style>`, elementy DOM i logikę `<script>`. Następnie za pomocą WordPress Site Editor usunięto domyślny nagłówek i stopkę motywu.

---

## ✅ Testy wysokiej dostępności (HA)

Architektura została pomyślnie przetestowana pod kątem odporności na awarie:

1. Ręcznie zatrzymano aktywną instancję EC2.
2. **ALB natychmiast wykrył awarię** i przekierował cały ruch użytkowników do zdrowej instancji.
3. Serwis działał **bez żadnych przestojów**.
4. **Auto Scaling Group automatycznie uruchomił nową instancję EC2**, przywracając klaster do pełnej wydajności.

---

## 📸 Zrzuty ekranu

### 1. Kreator instalacji WordPress
![WordPress Install]
<img width="1920" height="896" alt="image" src="https://github.com/user-attachments/assets/28d6c2c4-5999-4f1f-ae90-435b0a0be6e2" />


Formularz konfiguracji WordPress dostępny przez DNS Application Load Balancera. Widać wypełnione pola: tytuł witryny (*Cloud Store*), nazwa użytkownika (*admin*) oraz silne hasło wygenerowane automatycznie. Instalacja uruchamiana jest bezpośrednio przez publiczny adres ALB — dowód, że infrastruktura CloudFormation działa poprawnie.

---

### 2. CloudFormation — oś czasu wdrożenia (`CREATE_COMPLETE`)
![CloudFormation Timeline]
<img width="1920" height="862" alt="image" src="https://github.com/user-attachments/assets/8669f771-afe3-49d9-9469-77b19f5301ae" />


Widok konsoli AWS CloudFormation ze statusem **CREATE_COMPLETE** dla stacku `Lab2-Ecommerce`. Oś czasu (`Timeline view`) pokazuje równoległe provisjonowanie wszystkich zasobów: ALBListener, ScalingPolicy, AutoScalingGroup, DBInstance (RDS — najdłuższy zasób), MountTargetA/B (EFS) oraz LaunchTemplate. Całe środowisko zostało wdrożone automatycznie z jednego szablonu YAML.

---

### 3. Konfiguracja serwera przez AWS Session Manager
![Session Manager]
<img width="1446" height="982" alt="image" src="https://github.com/user-attachments/assets/f93b6db5-6516-4c89-8f75-f18d22bfa520" />


Terminal AWS Systems Manager Session Manager połączony z instancją EC2 (`i-02d0a1ffdde894fdd`). Widoczna konfiguracja Apache (`mod_rewrite`) dla WordPress — tworzenie reguł przepisywania URL w pliku `.htaccess` oraz restart serwera HTTP. Całość bez potrzeby otwierania portów SSH ani zarządzania kluczami.

---

### 4. Instalacja WordPress przez adres ALB
![WordPress Setup via ALB](screenshots/04-wordpress-alb-setup.jpg)

Standardowy kreator instalacji WordPress dostępny przez publiczny DNS Application Load Balancera (`lab2-e-appli-...us-east-1.elb.amazonaws.com`). Potwierdza, że ALB poprawnie kieruje ruch HTTP do instancji EC2 już na etapie pierwszego uruchomienia.

---

### 5. Panel administracyjny WordPress
![WordPress Dashboard](screenshots/05-wordpress-dashboard.jpg)

Strona główna panelu administracyjnego WordPress po pomyślnej instalacji. Widoczna wersja **6.9.4** i nazwa witryny *Cloud Store* w pasku nawigacji. Od tego miejsca instaluje się WooCommerce i konfiguruje cały sklep.

---

### 6. Instalacja wtyczki WooCommerce
![WooCommerce Install](screenshots/06-woocommerce-install.jpg)

Widok panelu wtyczek WordPress z wynikiem wyszukiwania dla *WooCommerce* — ponad **7 milionów aktywnych instalacji**, kompatybilność z bieżącą wersją WordPress potwierdzona. Przycisk *Install Now* uruchamia instalację bezpośrednio z repozytorium WordPress.

---

### 7. Błąd 504 Gateway Time-out podczas instalacji WooCommerce
![504 Error](screenshots/07-504-gateway-timeout.jpg)

Rzeczywisty błąd napotkany podczas projektu: **504 Gateway Time-out** przy próbie instalacji WooCommerce. ALB kierował żądania do różnych instancji EC2 (pobranie na Serwer A, rozpakowanie na Serwer B), a zapis archiwum na wolumen EFS przekraczał domyślny limit czasu 60 sekund. Rozwiązanie: tymczasowe skalowanie do jednej instancji.

---

### 8. WordPress Block Editor — publikacja strony Cloud Store
![Block Editor](screenshots/08-block-editor-publish.jpg)

Edytor blokowy WordPress z opublikowaną stroną *Cloud Store*. W lewym panelu widoczne wyszukiwanie bloku **Custom HTML** — tu wklejono kod z pliku `cloud-store-template.html`. Po prawej stronie potwierdzenie publikacji z adresem strony przez ALB. W podglądzie widoczny niestandardowy footer sklepu z kategoriami produktów.

---

### 9. Gotowy sklep Cloud Store — strona główna
![Cloud Store Frontend](screenshots/09-cloud-store-live.jpg)

Finalny efekt projektu — w pełni działający sklep internetowy **Cloud Store** dostępny pod adresem ALB. Widoczny własny interfejs z nawigacją (Home, Shop, Cart, About), koszykiem oraz sekcją hero z hasłem *„Everything you need, delivered from the cloud."* Strona działa na infrastrukturze Multi-AZ z Auto Scalingiem i współdzielonym EFS.

---

## 🧰 Technologie

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![CloudFormation](https://img.shields.io/badge/CloudFormation-FF4F8B?style=for-the-badge&logo=amazon-aws&logoColor=white)
![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white)
![WooCommerce](https://img.shields.io/badge/WooCommerce-96588A?style=for-the-badge&logo=woocommerce&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)

---

## 📄 Licencja

Projekt stworzony w celach edukacyjnych. Możesz go swobodnie używać i modyfikować.

---

## 📁 Pliki w repozytorium

| Plik | Opis |
|---|---|
| `1.yaml` | Szablon CloudFormation — tworzy całą infrastrukturę AWS jednym kliknięciem |
| `cloud-store-template.html` | Niestandardowy frontend sklepu (HTML/CSS/JS) — wklejany jako blok Custom HTML w WordPress |

### `1.yaml` — co tworzy?

Jeden szablon YAML provisjonuje **38 zasobów AWS** w pełni automatycznie: VPC, 6 podsieci (2 publiczne, 2 prywatne APP, 2 prywatne DB), Internet Gateway, 2× NAT Gateway, tabele routingu, 4 Security Groups, Launch Template z UserData bootstrap, Auto Scaling Group (min 2, max 4 instancje) ze Scaling Policy (CPU 50%), RDS MySQL Multi-AZ, EFS z dwoma Mount Targets oraz ALB z TargetGroup i Listenerem HTTP:80.

### `cloud-store-template.html` — co zawiera?

Kompletny interfejs sklepu napisany w czystym HTML/CSS/JS (zero zewnętrznych frameworków): strona główna z sekcją hero i siatką produktów, filtrowanie po kategorii i cenie, widok szczegółowy produktu, koszyk z przeliczaniem cen i kodami promocyjnymi, wieloetapowy checkout (dane → płatność → potwierdzenie).

> ⚠️ Przed wklejeniem do WordPress usuń tagi `<!DOCTYPE html>`, `<html>`, `<head>` i `<body>` — zachowaj tylko `<style>`, elementy DOM i `<script>`.

---

## 🔍 Audyt szablonu CloudFormation

| Sprawdzenie | Wynik | Status |
|---|---|---|
| RDS MultiAZ | Włączone | ✅ |
| RDS PubliclyAccessible | `false` — baza w prywatnej podsieci | ✅ |
| AMI | Dynamiczne przez SSM (`al2023-ami-kernel-default`) — zawsze aktualne | ✅ |
| ASG DependsOn EFS | ASG czeka na oba MountTargety przed uruchomieniem EC2 | ✅ |
| ALB HealthCheck | Skonfigurowany na `/wp-admin/install.php` | ✅ |
| NAT Gateway | 2× (jeden na AZ) — pełna redundancja | ✅ |
| DB subnety bez explicit route table | Używają domyślnej tablicy VPC — wystarczające dla RDS | ℹ️ |
| `wp-config.php` | Wymaga **ręcznej konfiguracji** po deploymencie — patrz Krok 2 | ⚠️ |
| Hasło domyślne | `HasloDoBazy123!` w parametrze — **zmień `DBPassword` przed użyciem!** | ⚠️ |
| HealthCheck po instalacji WP | Po zakończeniu instalacji `/wp-admin/install.php` zwraca redirect — rozważ zmianę na `/wp-login.php` | ℹ️ |

