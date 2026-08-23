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

## Kurulum

`SKILL.md` dosyasını Claude Code'un skill dizinine kopyalayın:

```
~/.claude/skills/fable-orchestration/SKILL.md
```

Proje bazlı, harness-agnostic bir kuruluma ihtiyacınız varsa `.agents/skills/fable-orchestration/SKILL.md` altına da konabilir (SKILL.md okuyan herhangi bir harness için).

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
