# LegalAi (Kâtip)

Hukuk büroları için Claude tabanlı bir yapay zekâ sistemi tasarımı. Şu an **tasarım aşamasındadır, uygulama kodu yoktur**.

## Fikir

Büronun verisi ofiste kalır; yalnızca akıl yürütme gereken metin parçaları Claude'a gider. Claude dışarıya kendiliğinden bir şey göndermez: her e-posta, fatura, dilekçe ve takvim daveti bir avukatın onay kuyruğundan geçer. Tasarım, yerel bir modelle çalışan bir büro sistemi örneğinden esinlenmiş ve Türk hukuk bürosu gerçeklerine (UYAP, HMK süreleri, AAÜT, KVKK, Avukatlık Kanunu) uyarlanmıştır.

## Tasarımdaki başlıklar

- Katmanlı mimari: giriş kanalları, yerel veri katmanı, gizlilik kalkanı, Claude çekirdeği, departmanlar, insan onayı.
- Sekiz departman alt-ajanı: kabul ve resepsiyon, çatışma ve vekalet, süreler ve duruşma, araştırma ve içtihat, dilekçe ve inceleme, e-posta ve müvekkil, zaman kaydı, faturalama ve emanet.
- Gizlilik seviyeleri (doğrudan, takma adlı, buluta hiç gitmeyen) ve denetim kaydı.
- Süre takibinde modelin değil deterministik bir hesaplayıcının kullanılması.

## Dosyalar

| Dosya | Ne var |
| --- | --- |
| `TASARIM.md` | Ayrıntılı tasarım belgesi |
| `katip-mimari.html` | Mimari diyagramı (tek dosyalık HTML, tarayıcıda açılır) |

## Teknoloji (tasarımda öngörülen)

Claude Agent SDK, MCP sunucuları, Postgres + pgvector, Docker Compose, yerel konuşma tanıma ve gömme modelleri. Bunlar henüz kurulmamıştır.

## Kurulum ve çalıştırma

Çalıştırılacak kod yok. Diyagramı görmek için `katip-mimari.html` dosyasını tarayıcıda açın.

## Ortam değişkenleri

Yok (kod bulunmadığı için).

## Lisans

Lisans belirtilmemiş.
