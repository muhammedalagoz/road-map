# 08i — System Design: Search Autocomplete (Typeahead)

## Gereksinimler

```
Functional:
  ✓ Kullanıcı yazdıkça öneri listesi göster (top 10 öneri)
  ✓ Popülerliğe göre sıralama
  ✓ Prefix matching ("java" → "java tutorial", "java spring")
  ✓ Kişiselleştirilmiş öneri (isteğe bağlı)
  ✗ Typo correction, semantic search (out of scope)

Non-Functional:
  DAU: 50M
  Her kullanıcı: 10 arama/gün, her aramada 5 keystroke ortalama
  QPS: 50M × 10 × 5 / 86400 ≈ 29,000 QPS
  Latency: < 100ms (her keystroke'ta)
  Availability: 99.99%
```

---

## Capacity Estimation

```
QPS: 29,000/sn (read-heavy, yüksek)
Peak: 29,000 × 3 = 87,000 QPS

Unique query (prefix) sayısı:
  1-8 karakter prefix → yaklaşık 10M unique prefix
  Her prefix → top 10 öneri (öneri başına ~30 byte)
  Cache boyutu: 10M × 10 × 30B = 3 GB → RAM'e sığar!

Arama log hacmi:
  50M × 10 = 500M arama/gün → aggregation pipeline için
```

---

## Trie Veri Yapısı

```
Trie = prefix tree

Örnek: "java", "javascript", "java spring"

              (root)
               │
               j
               │
               a
               │
               v
               │
               a ← "java" (count: 15000)
              / \
             s   (boşluk)
             |    │
             c    s
             |    │
             r    p
             |    │
             i    r
             ...  ...
          "javascript"  "java spring"
          (count: 8000)  (count: 5000)

Arama "jav" →
  root → j → a → v
  Bu node'dan alt ağacı tara → top K (count bazlı)

Insert/Update:
  "java" arandı → her node'a "java" count'unu ekle
  Sorun: Her arama için tüm prefix node'larını güncelle → yavaş

Çözüm: Her node top 10 sonucu cache'le
  Node "jav" → [{java: 15000}, {javascript: 8000}, ...]
  Yeni arama gelince sadece leaf'i güncelle, sonra parent'lara yayıl
```

---

## High-Level Tasarım

```
          Client (browser)
               │ "jav" keypress
               ▼
        ┌──────────────┐
        │   API        │
        │   Gateway    │
        │  (rate limit)│
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │  Autocomplete │
        │   Service     │
        └──────┬────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
 Redis      Trie        Cache
 (hot)    Service    (Memcached)
            │
            ▼
     ┌─────────────┐
     │  Trie Store │
     │  (prefix→   │
     │   top10)    │
     └─────────────┘

Data Pipeline (offline):
  Search Logs → Kafka → Aggregation → Trie Builder → Trie Store
  (Saatlik güncelleme)
```

---

## Read Path: Öneri Getirme

```
GET /autocomplete?q=jav&lang=tr

1. Redis'te key=autocomplete:jav → HIT?
   → Top 10 sonuçları döndür (< 1ms)

2. Cache MISS → Trie Service:
   a. Trie Store'dan "jav" node'unu bul
   b. Bu node'un precomputed top10 listesini al
   c. Redis'e yaz: SET autocomplete:jav [...] EX 3600

3. Sonuç client'a gönderilir

Client-side optimizasyon:
  Debounce: 150ms sonra istek at (her keystroke değil)
  Local cache: "ja" sonuçlarından "jav" filtrele (prefix match)
  → Gereksiz network request azaltılır
```

---

## Write Path: Trie Güncelleme

```
Batch güncelleme (gerçek zamanlı değil — latency fark etmez):

1. Arama logları → Kafka topic: "search.events"
   {userId, query: "java tutorial", timestamp, resultClicked: "..."}

2. Aggregation (Spark/Flink — saatlik):
   Son 7 günün arama logları → query → count
   SORT BY count DESC
   Top 1000 query for each prefix

3. Trie Builder:
   top queries → Trie'yi yeniden oluştur veya güncelle
   Yeni Trie → Blue/green switch
   (Tüm Trie yeniden build et → atomic swap)

4. Trie Store güncellendi → Redis cache invalidate (prefix'ler için)

Neden batch (gerçek zamanlı değil)?
  Popülerlik anlık değişmez — saatlik güncelleme yeterli
  Gerçek zamanlı güncellemede Trie yazma contention → karmaşık
  Daha az hata: toplu doğrulama → yanlış trend göstermez
```

---

## Trie Depolama Alternatifleri

### Seçenek 1: Redis Hash (Basit)

```
Key: "autocomplete:{prefix}"
Value: JSON list of top suggestions

SET autocomplete:jav '["java", "javascript", "java tutorial"]' EX 3600

Avantaj: Basit, prefix direkt anahtar
Dezavantaj: Her prefix için ayrı key → 10M prefix × memory
```

### Seçenek 2: Elasticsearch (Güçlü)

```
completion field type → native prefix autocomplete

PUT /search-index/
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "analyzer": "standard"
      }
    }
  }
}

PUT /search-index/_doc/1
{
  "suggest": {
    "input": ["java", "java tutorial", "java spring boot"],
    "weight": 15000
  }
}

GET /search-index/_search
{
  "suggest": {
    "query": {
      "prefix": { "field": "suggest", "value": "jav" }
    }
  }
}

Avantaj: Fuzzy matching, dil desteği, sıralama esnek
Dezavantaj: Daha fazla kaynak, daha karmaşık
```

### Seçenek 3: Trie In-Memory + Disk Snapshot

```
Process memory'de Trie → hızlı arama
Periyodik disk'e yaz → restart recovery

Avantaj: En hızlı (memory pointer traversal)
Dezavantaj: Process memory sınırlı, multi-instance'ta state senkronizasyonu
```

---

## Kişiselleştirme (Optional)

```
Kullanıcı geçmişi:
  Son 30 günde "spring boot" arayan kullanıcı →
  "java" yazınca "java spring boot" daha üste çıksın

Implementasyon:
  user_search_history: {userId, query, count, lastSearched}
  Autocomplete sonucu = global score × (1 + personal_boost)
  personal_boost = user_search_count / max_user_count

Coğrafi:
  Türkiye kullanıcısı → Türkçe sorgular daha üst
  query_by_country: {country, query, count} → ülke bazlı Trie
```

---

## Ölçekleme

```
29,000 QPS yüksek — nasıl kaldırılır?

1. Redis cluster:
   10M prefix × 30 bytes × 10 öneri ≈ 3 GB → 3 Redis node yeterli
   Read replicas: 5 slave → 29,000 QPS / 5 = 5,800 QPS/slave

2. Trie Service horizontal scaling:
   Stateless (Trie Store'dan oku) → 10 instance → 2,900 QPS/instance

3. Client-side debounce:
   150ms debounce → %60 request azalır → 11,600 actual QPS

4. CDN ile:
   Popüler prefix'leri CDN'de cache'le (1s TTL)
   "ja", "jav" → milyonlarca kullanıcı aynı prefix → CDN'de

5. Pre-computation:
   Her harf kombinasyonu için top 10 → Redis'te hazır
   İstek gelince Redis'ten al (Trie Service'e gitmez bile)
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B | Seçilen |
|-------|-----------|-----------|---------|
| Cache | Memcached | Redis | Redis (data yapısı zengin) |
| Trie update | Gerçek zamanlı | Batch (saatlik) | Batch (basit, yeterli) |
| Storage | Redis Hash | Elasticsearch | Redis (latency), ES (zengin) |
| Kişiselleştirme | Yok | Var | İkinci aşama (MVP sonrası) |
