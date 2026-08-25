# Fable Orchestration — Pro Tier Uyarlaması

Avenox'un [`avenoxskills`](https://github.com/avenoxai/avenoxskills) reposundaki `fable-orchestration` model-routing politikasının, **Claude Max ($200/ay)** yerine **Claude Pro ($20/ay)** planına göre uyarlanmış hali. Claude Code kullanan, aynı planda olan biri için doğrudan kullanılabilir.

## Neden bu uyarlama gerekti

Orijinal politika Opus'u **"unlimited" delege katmanı** olarak tanımlıyor ve context-gathering/keşif işini oraya yönlendiriyor. Bu varsayım Max planına dayanıyor. Claude Pro'da Opus kotası Sonnet'e göre belirgin biçimde daha dar (5 saatlik/haftalık limitler çok daha sıkı) — yani orijinal kuralı olduğu gibi uygulamak, kıt kaynağı (Opus) tam da politikanın korumak istediği işe değil, yüksek hacimli keşfe harcatıyor.

Bu repo aynı çekirdek ilkeyi ("kıt model ucuz iş yapmaz") koruyup delege katmanını Pro planının gerçek kotasına göre yeniden dağıtıyor.

## Ne değişti

| | Upstream (Max, $200) | Bu uyarlama (Pro, $20) |
|---|---|---|
| Ana döngü | Fable (varsayım) | Sonnet 5 (Claude Code'un fiilen çalıştırdığı model) |
| Context-gathering / keşif | Opus (sınırsız kabul edilir) | Sonnet 5 subagent; saf mekanik yüksek hacimli taramada Haiku 4.5 |
| Opus'un rolü | Rutin, yüksek hacimli delege | Nadir, açıkça istenen, yargı-ağırlıklı istisna — döngüde asla spawn edilmez |
| Codex gpt-5.x lanes | Değişmedi | Değişmedi (ayrı abonelik, `codex-fleet`/`omp-fleet` ile) |
| Trivial iş için subagent | Kural yok | Yasak — tek/iki adımlık işi ana döngü elle yapar (aşağıya bakın) |
| Büyük ham komut çıktısı | Kural yok | Context'e sokmadan önce dosyaya yaz ve özetle ("Context hygiene") |

### Sonradan eklenen: trivial subagent ve context hygiene kuralları

İlk yayından sonra, aynı hesapta gerçek bir kural ihlali gözlemlendi: bir smoke-test için atılan trivial bir subagent çağrısı, sabit Agent-tool overhead'i yüzünden ~33K token'a mal oldu — işin kendisinden çok daha pahalıya. Bu, politikanın kendi 1. kuralının ("delegasyonun kendi maliyeti var") reflex ihlaliydi. SKILL.md'ye iki kural eklendi:

- **Trivial işe subagent spawn etme** — tek/iki adımlık işi ana döngü elle yapar.
- **Context hygiene** — büyük ham komut çıktısı context'e girmeden önce dosyaya yazılıp özetlenir; Codex tarafındaki `Invoke-CodexQuiet.ps1` disipliniyle aynı mantık, Claude tarafında henüz otomatik bir sarmalayıcı yok.

## Kurulum

### Ön koşullar

| Gereksinim | Neden |
|---|---|
| Claude Code (CLI) | Skill'ler Claude Code özelliğidir |
| **Claude Pro ($20/ay) planı** | Politikanın tamamı Pro kotasına göre yazıldı. Max planındaysanız [upstream sürümü](https://github.com/avenoxai/avenoxskills) kullanın |
| `codex-fleet` / `omp-fleet` — **opsiyonel** | Politika Codex lane'lerine yönlendirme yapar. Bu skill'ler yoksa Codex kuralları basitçe uygulanmaz; geri kalan (Opus/Sonnet/Haiku yönlendirmesi, trivial-subagent yasağı, context hygiene) çalışır. Kurulum bunlara bağlı değildir |

### Kur

```bash
mkdir -p ~/.claude/skills/fable-orchestration
# SKILL.md dosyasını bu dizine kopyalayın:
#   ~/.claude/skills/fable-orchestration/SKILL.md
```

Proje bazlı, harness-agnostic bir kurulum için `.agents/skills/fable-orchestration/SKILL.md`
altına da konabilir (SKILL.md okuyan herhangi bir harness için).

> **İlk kurulumda dikkat.** Claude Code skill dizinlerini canlı izler; mevcut bir
> `~/.claude/skills/` içine dosya eklerseniz oturumu yeniden başlatmanız gerekmez.
> Ancak `~/.claude/skills/` dizini oturum başladığında **hiç yoksa** ve siz onu
> yeni yarattıysanız, Claude Code onu izlemeye alabilmek için **yeniden başlatılmalıdır.**
> İlk kez skill kuran herkes bu duruma düşer.

### Kurulduğunu doğrulayın

Bu skill görünmez bir yönlendirme politikasıdır — çalıştığını kendiliğinden fark etmezsiniz.
Üç adımla doğrulayın:

1. **Listede görünüyor mu?** Yeni bir Claude Code oturumunda `/` yazın ve menüde
   `fable-orchestration` girdisini arayın. Görünmüyorsa dosya yolu yanlıştır
   (`SKILL.md` adı ve dizin adı birebir olmalı).

2. **Doğrudan çağırın:** `/fable-orchestration` yazın. Skill içeriği yüklenmelidir.

3. **Otomatik tetiklendiğini görün:** Yeni bir oturumda şunu sorun —

   ```
   Bu işi hangi modele delege etmeliyim?
   ```

   Skill yüklüyse Claude bu kararı politikaya göre verir: ana döngü Sonnet 5'te kalır,
   Opus rutin delege olarak spawn edilmez, ve trivial iş için subagent açılmaz.
   Bu üçü olmuyorsa skill devrede değildir.

> **Kaldırmak için** `~/.claude/skills/fable-orchestration/` dizinini silin.
> Değişiklik oturum içinde algılanır.

## Doğrulama durumu

Sonnet 5 subagent'i ile küçük, salt-okunur bir smoke test çalıştırıldı: bir dizin listeleme ve bu dosyanın frontmatter'ını okuma görevi. 2 tool call, ~33K token, dosya değişikliği yok — delegasyon yolu (ana döngü → Sonnet subagent, Opus'a dokunmadan) çalışır durumda doğrulandı.

Doğrulanmayan şey: gerçek fleet ekonomisi. Pro planındaki Opus/Sonnet kota tüketimi karşılaştırmalı olarak ölçülmedi; bu repo bir maliyet/performans kıyaslaması iddia etmiyor, yalnızca plan kısıtına göre uyarlanmış bir routing politikası sunuyor.

## Sınırlar

- Bu bir kişisel operatörlük notu/uyarlamasıdır, resmi bir Avenox yayını değildir.
- Tek makinede, tek Claude Pro hesabında test edildi; farklı kullanım desenlerinde kota davranışı değişebilir.
- Politika, hesap Max'e geçerse yeniden değerlendirilmeli — dosyanın kendisi bunu not ediyor.

## Kaynak ve lisans

Bu proje Avenox'un `avenoxskills` reposundaki `fable-orchestration` skill'inin türevidir. Orijinal tasarım ve emek Avenox'a aittir:

- Kaynak skill: https://github.com/avenoxai/avenoxskills/blob/main/skills/fable-orchestration/SKILL.md
- Avenox: https://avenox.lol

Orijinal repo MIT lisanslıdır; bu repo da MIT altında paylaşılıyor ve orijinal telif bildirimi `LICENSE` dosyasında korunmuştur.
