# OSINT Ders Notu — TryHackMe Sakura Room

> Bu not, Sakura Room'u çözerken kullanılan teknikleri ve komutları içerir.
> Amaç: aynı yöntemleri başka OSINT işlerinde tekrar uygulayabilmek.

---

## 0. Genel Metodoloji

OSINT'te tek bir "kanıt" nadiren yeterlidir. Yöntem şudur:

1. Eldeki artefakttan bir **pivot** (kullanıcı adı, e-posta, cüzdan adresi, SSID) çıkar
2. O pivotu başka platformlarda ara
3. Yeni pivotlar topla, tekrarla
4. **Her bulguyu bağımsız bir kaynakla doğrula**

**En kritik kural:** Bir aracın (Sherlock, AI özeti, arama motoru) verdiği çıktı kanıt değildir, ipucudur. Doğrulanana kadar hipotez olarak tut.

---

## 1. Görsel Metadata Analizi

### Amaç
Bir dosyanın içine gömülü bilgiden (yazar, yazılım, dosya yolu, GPS) kimlik çıkarmak.

### Komutlar

```bash
# Temel metadata çıkarma
exiftool dosya.svg

# Sadece belirli alanları görmek
exiftool -Author -Creator -GPS* dosya.jpg

# Bir klasördeki tüm dosyalar
exiftool -r /klasor/yolu/

# Çıktıyı dosyaya kaydet
exiftool dosya.svg > metadata.txt
```

### Bakılacak alanlar
| Alan | Ne sızdırır |
|---|---|
| `Author` / `Creator` / `Artist` | İsim veya kullanıcı adı |
| `Export-filename` | **Dosya yolu → kullanıcı adı** (`/home/KULLANICI/...`) |
| `Software` / `Version` | Kullanılan program, OS ipucu |
| `Create Date` | Zaman çizelgesi |
| `GPS*` | Konum (telefon fotoğraflarında) |

### Format özel notu: SVG
SVG aslında XML'dir. exiftool olmadan da okunur:

```bash
cat dosya.svg | less
strings dosya.svg | grep -i "home\|author\|creator"
grep -i "dc:creator\|inkscape" dosya.svg
```

**Sakura'da bulunan:** `Export-filename: /home/SakuraSnowAngelAiko/Desktop/pwnedletter.png`

---

## 2. Kullanıcı Adı Pivotu (Username Enumeration)

### Sherlock

```bash
# Kurulum
sudo apt install sherlock
# veya
pipx install sherlock-project

# Kullanım
sherlock KullaniciAdi

# Sonuçları dosyaya yaz
sherlock KullaniciAdi -o sonuclar.txt

# Sadece belirli siteler
sherlock KullaniciAdi --site GitHub --site Reddit

# Zaman aşımını artır (yavaş siteler için)
sherlock KullaniciAdi --timeout 10
```

### Alternatif araçlar
- **WhatsMyName** — https://whatsmyname.app (tarayıcı, kurulum yok)
- **Namechk** — https://namechk.com
- **Maigret** — Sherlock'un daha kapsamlı forku

```bash
pipx install maigret
maigret KullaniciAdi
```

### Manuel arama (araçların kaçırdıkları için)

```
"KullaniciAdi"                    → tam eşleşme
site:github.com "KullaniciAdi"
site:linkedin.com "KullaniciAdi"
inurl:KullaniciAdi
```

### ⚠️ False Positive Uyarısı
Sherlock çıktısının çoğu **gerçek hesap değildir.** Bazı siteler var olmayan profil için 404 yerine 200 döndürür.

**Doğrulama yöntemi:**
- Profili aç, gerçekten içerik var mı?
- Hesap açılış tarihi olayla uyuşuyor mu?
- Avatar / dil / konu ilgili mi?

*Sakura'da:* Sherlock 30+ site listeledi, gerçekte anlamlı olan sadece GitHub'dı. `@AiKOABE3` adlı benzer bir hesap tamamen alakasız bir kişiye aitti (2022'de açılmış, Japonca otome oyunu içeriği).

---

## 3. GPG / PGP Anahtar Analizi

### Amaç
Public key bloğunun içindeki **User ID** alanından isim ve e-posta çıkarmak.

### Adımlar

```bash
# 1. Anahtarı dosyaya kaydet (GitHub'da "Raw" görünümünden kopyala)
nano publickey.asc
# Yapıştırma: Ctrl+Shift+V  (terminalde Ctrl+V çalışmaz)
# Kaydet: Ctrl+O → Enter → Ctrl+X

# 2. Doğrula — BEGIN ve END satırları tam olmalı
cat publickey.asc
head -1 publickey.asc    # -----BEGIN PGP PUBLIC KEY BLOCK-----
tail -1 publickey.asc    # -----END PGP PUBLIC KEY BLOCK-----

# 3. İçe aktar
gpg --import publickey.asc

# 4. Listele — uid satırına bak
gpg --list-keys

# Alternatif: import etmeden okuma
gpg --show-keys publickey.asc
```

### Örnek çıktı
```
pub   rsa3072 2021-01-23 [SC] [expired: 2023-01-22]
      A6519F273BF88E9126B0F4C5ECDD0FD294110450
uid           [ expired] SakuraSnowAngel83@protonmail.com
```

`uid` satırı = aradığın bilgi. Süresi dolmuş olması sorun değil.

### Tarayıcıdan alternatif
https://cirw.in/gpg-decoder — bloğu yapıştır, parse edilmiş çıktıyı oku.

### Ek pivot
E-postadaki sayı (`83`) genelde **doğum yılı**dır ve **ayrı bir arama terimi**dir. Ana kullanıcı adının bulunmadığı platformlarda bu varyant çıkabilir.

---

## 4. Git Geçmişi — Silinen Veriyi Kurtarma

### Prensip
Git'te bir dosyayı düzenleyip gizli bilgiyi silmek **veriyi yok etmez.** Eski hâli commit geçmişinde durur.

### GitHub web arayüzünden

1. Repo → dosyaya tıkla
2. Sağ üstte **History** (saat ikonu) veya **Blame**
3. Eski commit'i aç → diff'te:
   - 🔴 Kırmızı satır (`-`) = **silinen orijinal veri**
   - 🟢 Yeşil satır (`+`) = temizlenmiş yeni hâl
4. Alternatif: commit yanındaki `<>` butonu → repoyu o andaki hâliyle gez

### Terminalden

```bash
git clone https://github.com/kullanici/repo.git
cd repo

# Tüm commit geçmişi
git log --oneline

# Bir dosyanın tüm değişiklikleri (içerikle birlikte)
git log -p dosyaadi

# Belirli iki commit arası fark
git diff COMMIT1 COMMIT2 -- dosyaadi

# Silinmiş dosyaları da bul
git log --diff-filter=D --summary

# Commit metadata'sı (yazar adı + e-posta!)
git log --format='%an <%ae>'
```

> `git log` çıktısındaki **committer adı ve e-postası** başlı başına bir kimlik pivotudur.

### Stratum URL yapısı (mining scriptlerinde)

```
stratum://CÜZDAN.WORKERID:PAROLA@HAVUZ:PORT
```

Örnek:
```
stratum://0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef.Aiko:pswd@eu1.ethermine.org:4444
          └────────── ETH cüzdan adresi ──────────┘ └──┘      └──────┬──────┘ └─┬┘
                                                  worker           havuz      port
```

Tek satırdan 4 bilgi: para birimi, cüzdan, worker adı, mining havuzu.

---

## 5. Blockchain Takibi

### Etherscan kullanımı

1. https://etherscan.io → arama kutusuna `0x...` cüzdan adresi
2. **Transactions** sekmesi = ETH transferleri
3. **Internal Transactions** = kontrat kaynaklı hareketler
4. **Token Transfers (ERC-20)** = USDT, USDC gibi tokenlar

### Okuma ipuçları

| Görünen | Anlamı |
|---|---|
| `IN` + From: **Ethermine** | Mining ödemesi alınmış |
| `OUT` + To: **Nobitex / Binance** | Borsaya çekim (etiketli adresler) |
| `OUT` + To: **Tether: USDT** | Stablecoin'e çevirme |
| **FUNDED BY** | Cüzdanı ilk fonlayan adres — önemli pivot |

### Tarih filtreleme
- `Age` sütununa tıkla → tarih formatına geçer
- Çok işlem varsa `Show: 50/100 Records` ve sayfa gezinme
- URL'ye `&p=2` ekleyerek sayfa atlama

### Diğer zincirler
| Zincir | Explorer |
|---|---|
| Bitcoin | blockchain.com/explorer, blockchair.com |
| Ethereum | etherscan.io |
| BSC | bscscan.com |
| Polygon | polygonscan.com |

---

## 6. Arşiv — Silinmiş İçeriğe Erişim

### Wayback Machine

```
# Bir URL'nin tüm snapshot'ları
https://web.archive.org/web/*/twitter.com/KULLANICI

# Belirli yıl
https://web.archive.org/web/2021*/twitter.com/KULLANICI

# Belirli tarih (yyyyMMddHHmmss)
https://web.archive.org/web/20210131/twitter.com/KULLANICI
```

⚠️ **Dikkat:** Domain `archive.org` — `archieve.org` DEĞİL. Yanlış yazım typosquat reklam sitesine gider.

### Terminal'den arşiv sorgulama

```bash
# Bir domain için arşivlenmiş tüm URL'ler
curl "http://web.archive.org/cdx/search/cdx?url=ornek.com*&output=text&fl=original&collapse=urlkey"

# waybackurls aracı
go install github.com/tomnomnom/waybackurls@latest
echo "ornek.com" | waybackurls
```

### Snapshot boş gelirse
- **Diğer capture tarihlerini dene** (5 capture varsa 5'ini de)
- Profil sayfası yerine **tekil post URL'sini** dene, veya tersi
- **Ctrl+U** ile kaynak koda bak — render edilmeyen link metinde olabilir

### Diğer arşiv kaynakları
- archive.today / archive.ph
- Google cache: `cache:site.com/sayfa`
- Nitter instance'ları (X için, login gerektirmez — çalışırlığı değişken)

---

## 7. WiFi Coğrafi Konumlandırma — WiGLE

### Amaç
Bir **SSID** (ağ adı) → **BSSID** (MAC adresi) → **fiziksel konum**

### Adımlar
1. https://wigle.net → **ücretsiz hesap aç** (zorunlu)
2. Üstte **View** → **Advanced Search**
3. **SSID** alanına ağ adını yaz
4. Sonuç tablosunda **BSSID** sütunu = `XX:XX:XX:XX:XX:XX`
5. Sonuca tıkla → **harita üzerinde konum**

### Neden güçlü
BSSID sonucu haritada bir noktaya düşer. Bu, o access point'in fiziksel adresidir — dolaylı çıkarım değil, doğrudan konum verisi.

### Filtreleme
- Çok sonuç gelirse ülke/bölge ile daralt
- Ama **çok agresif daraltma** yapma — beklediğin bölge yanlış olabilir

---

## 8. Görsel İstihbarat (IMINT)

### Reverse image search

| Araç | Güçlü olduğu alan |
|---|---|
| **Google Lens** | Ürünler, metin içeren görseller |
| **Yandex** | **Yüzler, binalar, manzara** — genelde en iyisi |
| **TinEye** | Görselin ilk yayınlandığı yeri bulma |
| **Bing Visual** | Bazen diğerlerinin kaçırdığını yakalar |

> Manzara ve bina tanımada **Yandex'i önce dene.**

### Fotoğraftan konum çıkarma — bakılacaklar

- **Tabela ve yazılar** (dil, alfabe, marka)
- **Havayolu logoları** → hangi hub? (JAL Sakura Lounge → Japonya)
- **Elektrik direkleri, trafik levhaları** → ülkeye özgü tasarım
- **Araç plakaları** (formatı, rengi)
- **Bitki örtüsü, güneş açısı, gölge yönü**
- **Dağ siluetleri** → PeakVisor, Google Earth ile eşleştirme

### Uydu görüntüsünden bölge tanımlama
1. Belirgin özellik seç (göl şekli, kıyı çizgisi, ada)
2. Google Maps / Earth'te uydu görünümüne geç
3. Kıyı ve göl şekillerini karşılaştır
4. Reverse image search de dene — bazen doğrudan cevap verir

### Havalimanı kodları
- **IATA** = 3 harf (HND, DCA, NRT) — bilet ve bagaj etiketlerinde
- **ICAO** = 4 harf (RJTT, KDCA) — havacılık operasyonlarında
- Soru formatı `***` ise → IATA isteniyor

---

## 9. Tor Erişimi

### Brave üzerinden (en hızlı)
Menü → **New private window with Tor**

### Brave kurulumu (Kali)

```bash
sudo apt install curl

sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg arch=amd64] \
  https://brave-browser-apt-release.s3.brave.com/ stable main" \
  | sudo tee /etc/apt/sources.list.d/brave-browser-release.list

sudo apt update && sudo apt install brave-browser
```

### Tor Browser (daha güvenli)

```bash
sudo apt install torbrowser-launcher
torbrowser-launcher
```

> **Not:** Brave'in Tor modu trafiği Tor üzerinden geçirir ama tarayıcı parmak izini Tor Browser kadar sertleştirmez. CTF için yeterli, gerçek anonimlik gerektiren iş için değil.

---

## 10. Sakura Room — Doğrulanmış Cevaplar

| # | Soru | Cevap |
|---|---|---|
| 2 | Kullanıcı adı | `SakuraSnowAngelAiko` |
| 3 | E-posta | `SakuraSnowAngel83@protonmail.com` |
| 3 | Gerçek isim | Aiko Abe |
| 4 | Kripto para | ETHEREUM |
| 4 | Cüzdan | `0xa102397dbeeBeFD8cD2F73A89122fCdB53abB6ef` |
| 4 | Mining havuzu | Ethermine |
| 4 | Takas edilen para | Tether |
| 5 | Güncel Twitter | `SakuraLoverAiko` |
| 5 | Ev WiFi BSSID | `84:AF:EC:34:FC:F8` |
| 6 | Kalkış havalimanı | DCA (Washington) |
| 6 | Son aktarma | HND (Tokyo Haneda) |
| 6 | Göl | Lake Inawashiro (Fukushima) |
| 6 | Ev şehri | *(WiGLE haritasından)* |

---

## 11. Hata Yapmamak İçin Kontrol Listesi

- [ ] Bulduğum şeyi **bağımsız bir kaynakla** doğruladım mı?
- [ ] Sherlock/araç çıktısındaki hesap **gerçekten** aynı kişiye mi ait?
  → Açılış tarihi, dil, içerik, avatar kontrolü
- [ ] Ekran görüntüsündeki bilgi **o anki** hâl mi, **güncel** hâl mi?
  → Kullanıcı adları değişir; eski screenshot güncel cevap değildir
- [ ] "Temizlenmiş" görünen dosyanın **geçmişine** baktım mı?
- [ ] Domain adını **doğru** yazdım mı? (archive vs archieve)
- [ ] AI özeti / arama snippet'i mi okuyorum, **birincil kaynak** mı?

### Son ve en önemli
> **AI çıktısı, arama motoru özeti ve tahmin — hepsi hipotezdir.**
> Kontrol edilebilir bir referansa (CTF'in kendi doğrulayıcısı, birincil belge,
> bağımsız ikinci kaynak) karşı test edilmeden cevap sayılmaz.

Sakura Room'da bu defalarca doğrulandı: hem AI özetleri hem de "mantıklı görünen"
çıkarımlar yanlış çıktı. Her seferinde doğru olan yöntem, cevabı odanın kendi
kontrol mekanizmasına sormaktı.

---

## 12. Sonraki Adımlar

**Aynı seviyede odalar:**
- OSINT Dojo'nun diğer odaları
- TryHackMe: Searchlight-IMINT (görsel istihbarat ağırlıklı)
- TryHackMe: OhSINT (kısa, metadata odaklı)

**Pratik kaynaklar:**
- OSINT Framework — https://osintframework.com
- Bellingcat Online Investigation Toolkit
- Trace Labs CTF'leri (gerçek kayıp kişi vakaları)
