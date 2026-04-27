# Konusma Notlari — The Other Half of the Context Problem

**ClawCon Ankara** · 29 Nisan 2026
**Konusmaci:** B. Mert Koseoğlu

---

## Slide 1 — Baslik

Herkese iyi aksamlar.

Baslamadan once — Veli, Furkan, Murat, ClawCon, OpenClaw, hepinize tesekkurler. Ankara'da olmak cok guzel bu aksam.

Ben Mert. context-mode diye bir open-source MCP server yapiyorum. Bu aksam sizinle bir seyden bahsetmek istiyorum. Bir suredir beni rahatsiz eden bir sey var. Sizi de rahatsiz ediyor aslinda, sadece henuz buna bir isim koymamissiniz.

---

## Slide 2 — AI agent'inizin hafizasi yok

Sunu soyleyeyim.

AI agent'iniz? Hicbir sey hatirlamiyor. Biliyorum, hatirliyormus gibi geliyor. Ama hatirlamiyor. Claude Code, Cursor, Copilot, Codex, Gemini CLI — hangisini kullanirsiniz farketmez. Kaputun altinda hepsi ayni. Model stateless. Turn'ler arasi sifir hafiza.

Hafiza gibi gorunen sey ne peki? Yeniden gonderme. Her turn'de butun konusma — her mesaj, her tool sonucu, her okunan dosya — paketlenip API'ye bastan gonderiliyor. Sifirdan. "Hafiza" bu.

Bunun pratikte nasil gorundugunun gostereyim.

---

## Slide 3 — Her turn'de her sey yeniden gonderiliyor

Arka planda gercekte ne oluyor bakalim.

Turn bir. Bir sey soruyorsunuz. Tool calisiyor. 60 KB donuyor. Tamam. Sorun yok.

Turn bes. Simdi API call'unda birden dorte kadar her sey var, ustelik yeni veriler de eklenmis. 300 KB. Ilk dort turn'un bedelini tekrar oduyorsunuz.

Turn on bes. Ayni hikaye. 600 KB. Eski tool sonuclari? Hala orada.

Turn otuz. 1.2 megabyte. Tek bir API call'unda.

Sonra turn otuz bir. Context window doldu. Agent panikliyor. Tum konusmayi ozetlemeye calisiyor, sikistiriyor, sonra asillari siliyor. Okudugu her sey, verdigi her karar, duzeltigi her hata, yarim kalan is — gitti. Geriye lossy bir ozet kaliyor.

Ve siz codebase'inizi bastan anlatiyorsunuz. Sifirdan.

---

## Slide 4 — Bir komut. 750,000 token.

Bunu somutlastirayim.

`gh issue list` calistiriyorsunuz. Tek komut. 59 KB JSON donuyor. Yaklasik 15,000 token. Tamam, makul dimi?

Ama kimsenin bahsetmedigi kisim su. O 15,000 token artik konusmanin parcasi. Sonraki turn'de tekrar gonderiliyor. Ondan sonraki turn'de tekrar. Agent o veriye artik bakmiyor bile. Cevabi coktan verdi. Ama protokol soyle diyor: her API call'unda tam konusma. Yani gondermeye devam ediyor.

Elli turn sonra? O tek komut size 750,000 token'a mal oldu.

Tek komut. 750,000 token.

Ve kimse bir session'da tek komut calistirmiyor. Yirmi, otuz komut calistiriyorsunuz. Ortalama cikti yaklasik 30 KB.

30 megabyte input token. Session basina. Sadece tool ciktisi. Promptlariniz degil, modelin cevaplari degil — sadece agent'in coktan isle yip bir daha bakmayacagi ham ciktilar.

Context'iniz buraya gidiyor. Ve evet — paraniz da buraya gidiyor.

---

## Slide 5 — Context dolunca her sey kayboluyor

Context doluyor. Simdi ne olacak?

Agent compact yapiyor. Ozetliyor. Ama ozetleme lossy. NoLiMa benchmark'u on iki modeli test etti — 32K token'dan sonra yuzde 50 dogruluk dususu. Model daha compact yapmadan once bile kotulesiyor. Sadece gurultuden.

Compaction'dan sonra daha da kotuye gidiyor. Okudugu dosyalar — artik tek satirlik bir ozet. Verdigi kararlar — duzlestirilmis. Buldugu ve duzeltigi hatalar — gitti. Projenizin nasil calistigini anlatmak icin harcadiginiz bes dakika — kayip.

Bu gercek bir session'da her yirmi-otuz dakikada oluyor. Her seferinde on bes dakika kaybediyorsunuz, her seyi bastan anlatarak.

Elli kisilik bir takim, kisi basi ayda yuz dolar, yilda 60K eder. Sizi surekli unutan bir agent icin.

---

## Slide 6 — Ya veri context'e hic girmeseydi?

*(bir an sessizlik)*

Ya veri context window'a en basindan hic girmeseydi?

Daha iyi sikistirma degil. Daha akilli ozetleme degil. Daha buyuk window degil. Ya `gh issue list`'ten gelen o 59 KB hic konusmanin parcasi olmasaydi?

---

## Slide 7 — Intercept

context-mode bunu yapiyor. Araya giriyor.

Bir MCP server. Agent ile toollari arasinda oturuyor. Her tool call bir routing engine'den geciyor. Kilit nokta — tool'lara degil, agent'in lifecycle'ina hook'laniyor.

Bes tane hook var. Her biri tek is yapiyor.

PreToolUse — tool calismadan once ateслeniyor. Tehlikeli seyleri yakaliyor. `curl`, `wget`, `rm -rf`. Engelliyor ya da sandbox'a yonlendiriyor.

PostToolUse — tool dondukten sonra atesleniyor. Ana olay bu. 59 KB JSON lokal FTS5 veritabanina gidiyor. Full-text search, BM25 siralama. Agent'e 1.1 KB'lik bir ozet donuyor. Ayni bilgi. Yuzde 98 daha az token.

Ve bu truncation degil. Verinin tamami hala orada. Hala aranabilir. Agent ihtiyaci olunca `ctx_search` ile sorguluyor. Sadece konusmada oturup elli kere yeniden gonderilmiyor.

Peki tek bir plugin on dort platformda nasil calisiyor? Uc yolla. Claude Code ve Codex'in native hook destegi var, routing otomatik. Cursor ve Copilot'un MCP destegi var ama hook yok, context-mode MCP server olarak calisiyor, `ctx_execute` ve `ctx_search` gibi tool'lar sunuyor. Yeni CLI'larda ikisi de yoksa routing kurallari system prompt'a enjekte ediliyor. Uc yaklasim, ayni sonuc.

---

## Slide 8 — Sandbox (Think in Code)

Sandbox. Bunu dogru yapmak en uzun suren kisim oldu.

Biz buna Think in Code diyoruz. Problem su.

Interception olsa bile agent hala veriyi context'e cekip "dusunmek" istiyor. Codebase'i anlamak icin 47 dosya okuyor. 700 KB. Hatalari bulmak icin `cat access.log` calisiyor. 45 KB. Hepsi context'te oturuyor.

Think in Code bunu tersine ceviriyor. Veriyi iceri cekmek yerine, kodu veriye gonderiyorsunuz. Agent kucuk bir script yaziyor. context-mode bunu sandbox'ta calisiyor. Sadece stdout geri donuyor.

Gercek ornek. Access log'lari. Oncesi: `cat access.log` — 45 KB context'e giriyor, sonsuza kadar orada kaliyor. Sonrasi: `ctx_execute`, agent 500 hatalari filtrelemek ve saymak icin bes satir yaziyor. 155 byte stdout. Log dosyasi konusmaya hic dokunmuyor.

200 kat daha kucuk. Ve durust olmak gerekirse, agent bu sekilde daha iyi is cikariyor. Gercek analiz kodu yazmak, bir dil modelinden log dosyasina gozle bakmasini istemekten daha guvenilir.

1.0.64 versiyonundan beri bu zorunlu. On dort platformun hepsinde. Oneri degil — routing kurallariyla zorunlu kilinan bir kural. Agent buyuk bir dosyayi analiz icin `Read()` ile okumaya kalkarsa, PreToolUse sandbox'a yonlendiriyor.

CPU'nuz bu isi bedavaya yapiyor. Token'lar para.

---

## Slide 9 — Index (Session persistence)

Index.

Agent'in dokundugu her sey lokal bir SQLite veritabanina gidiyor. FTS5, BM25 siralama. Tool ciktilari, web sayfalari, sandbox sonuclari. Hepsi aranabilir. Agent'in bu session'da gordugu her sey icin lokal bir arama motoru gibi.

Ama asil onemli kisim — session persistence.

PostToolUse sadece cikti yakalmiyor. Event'leri yakaliyor. On bes kategori, gercek zamanli. Dosya islemleri, git komutlari, hatalar ve duzeltmeler, sizin duzetmeleriniz, ortam detaylari, kararlar. Hepsi ayri bir SessionDB'ye gidiyor.

Context dolup agent compact yaptiginda — normalde is biter. Her sey gider. Ama context-mode'un PreCompact hook'u var. Silme isleminden hemen once yapilandirilmis bir snapshot olusturuyor. Ozet degil. Knowledge base'e yapilandirilmis referanslar.

Sonra bir sonraki turn'de SessionStart atesleniyor. Snapshot'u geri yukluyor. Index'lenmis bilgiyi restore ediyor. Agent kaldigi yerden devam ediyor.

26 event kategorisi her compaction'dan saglikli cikiyor. Promptlariniz, takip edilen dosyalar, proje kurallari, kararlar, git islemleri, hatalar, ortam bilgisi — hepsi.

Bastan anlatmayi birakiyorsunuz. Agent zaten biliyor. Sadece bakip buluyor.

---

## Slide 10 — Olculmus sonuclar

Tamam. Rakamlar.

Playwright browser snapshot. 56 KB ham DOM. context-mode ile? 299 byte. Yuzde 99.5 azalma.

GitHub Issues. 59 KB. context-mode ile? 1.1 KB. Yuzde 98.

Access log analizi. 45 KB. Think in Code ile? 155 byte. Yuzde 99.7.

Tam session, elli turn. Yeniden gonderilen 30 megabyte bire dusuyor. Ustelik output compression da var — input azalmasinin ustune yuzde 65-75 output token tasarrufu.

Yuzde 98-99.5 daha az context. Ayni is. Ayni cevaplar. Ama agent duvara carpmadan daha uzun gidiyor. Ve otuz turn oncesinden kalan bayat ciktilarla dolu olmadigi icin daha iyi kararlar veriyor.

---

## Slide 11 — Kapanis

context-mode. Open source. Hep oyle oldu.

95,000 kullanici. 79,000 npm indirme. 14 platform. 10,000 GitHub yildizi. Microsoft, Google, Meta, ByteDance, GitHub, Stripe, Datadog, NVIDIA, Supabase'deki takimlar kullaniyor.

14 adapter — Claude Code, Cursor, Codex, Gemini, Kiro, Cline ve daha fazlasi. Native bir OpenClaw adapter'i da dahil. Bunu burada bu aksam soylemek guzel hissettiriyor.

Neden open source? Cunku durust olmak gerekirse — bu problemin size token satan sirketler tarafindan cozulecegini dusunmuyorum. Daha az kullanmaniza yardim etmek icin sifir motivasyonlari var. Is modelleri bu. Yani bu bizden gelmeli.

context-mode.com.

Tesekkurler Veli, Furkan, Murat ve ClawCon. Herkese tesekkurler.
