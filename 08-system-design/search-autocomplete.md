# 08i — System Design: Search Autocomplete (Typeahead)

## Gereksinimler

```
Functional:
  ✓ Kullanıcı yazdıkça öneri listesi (top 10)
  ✓ Popülerliğe göre sıralama (global + kişisel)
  ✓ Prefix matching ("java" → "java tutorial", "java spring")
  ✓ Multi-word/phrase completion ("python mac" → "python machine learning")
  ✓ Trending queries (son 1 saatte patlayan sorgular)
  ✓ Blacklist / içerik moderasyonu
  ✓ Multi-language / unicode normalizasyon
  ✓ Kişiselleştirilmiş öneri (kullanıcı geçmişi)
  ✗ Semantic search, spell correction (out of scope — ayrı katman)

Non-Functional:
  DAU: 50M
  Her kullanıcı: 10 arama/gün, her aramada 5 keystroke
  QPS: 50M × 10 × 5 / 86400 ≈ 29,000 QPS (read)
  Write (log): 50M × 10 / 86400 ≈ 5,787 arama log/sn
  Latency: < 100ms (P99 her keystroke'ta)
  Availability: 99.99%
  Öneri tazeliği: saatlik (trending: 5 dakika)
```

---

## Capacity Estimation

```
QPS hesabı:
  29,000 QPS ortalama, peak: 29,000 × 3 = 87,000 QPS.
  Debounce (150ms) ile: %60 azalır → 11,600 effective QPS.
  CDN (popüler prefix'ler): %40 daha azalır → 6,960 backend QPS.

Prefix cache boyutu:
  Unique prefix sayısı (1-8 karakter): ~10M.
  Her prefix → top 10 öneri (öneri başına ~50 byte metadata ile).
  Cache: 10M × 10 × 50B = 5 GB → Redis Cluster'a sığar.

Trie boyutu (in-memory):
  İngilizce: 26 harf, ortalama kelime uzunluğu 5.
  Türkçe: 29 harf + Latince.
  Unique kelime: 5M → Trie node başına ~100 byte → 500 MB.
  Her node top-10 list dahil: ~2 GB → makul.

Arama log storage:
  5,787 arama/sn × 200 byte = 1.1 MB/sn → 100 GB/gün (Kafka'da 7 gün: 700 GB).

Aggregation çıktısı:
  Top prefix-query çiftleri: ~100M kayıt → DFS gzip → ~5 GB.
  Trie build: Spark job, 100M kayıt → 30 dk.
```

---

## Trie Veri Yapısı (Detay)

### Temel Yapı

```
Trie = Prefix Tree (Retrieval Tree)

Örnek: "java", "javascript", "java spring" eklenmiş.

            (root)
              │
              j (children: a)
              │
              a (children: v)
              │
              v (children: a)
              │
              a ← "java" count:15000, top10:[java,javascript,java spring,...]
             / \
            s   (space)
            │     │
            c     s
            │     │
            r     p
            │     │
           ...   ...
       "javascript"  "java spring"
       count:8000     count:5000

Her node sakladığı:
  - char: bu node'un temsil ettiği karakter
  - children: Map<char, TrieNode>
  - isEnd: bu node bir kelimenin sonu mu?
  - count: bu prefix'e kadar gelen arama sayısı
  - topK: bu prefix için önceden hesaplanmış top-10 list (önbellek)
```

### Top-K Extraction Algoritması

```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    boolean isEnd;
    long count;
    List<Suggestion> topK = new ArrayList<>();  // önceden hesaplanmış
}

// Prefix'e göre top-K bul (topK önbellekli)
List<Suggestion> suggest(String prefix) {
    TrieNode node = traverse(prefix);   // prefix'e kadar in
    if (node == null) return emptyList();
    return node.topK;                   // önceden hesaplanmış → O(1)
}

// topK olmasa: DFS + min-heap
List<Suggestion> dfsTopK(TrieNode node, String current, int k) {
    PriorityQueue<Suggestion> heap = new PriorityQueue<>(
        Comparator.comparingLong(s -> s.count));  // min-heap

    dfs(node, current, heap, k);
    return new ArrayList<>(heap).reversed();
}

void dfs(TrieNode node, String prefix, PriorityQueue<Suggestion> heap, int k) {
    if (node.isEnd) {
        heap.offer(new Suggestion(prefix, node.count));
        if (heap.size() > k) heap.poll();  // k'yı aş → en küçüğü çıkar
    }
    // Budama: bu alt ağacın max count'u heap'in en küçüğünden düşükse atla
    // (count bilgisi subtree'de saklanıyorsa bu optimizasyon mümkün)
    for (var entry : node.children.entrySet()) {
        dfs(entry.getValue(), prefix + entry.getKey(), heap, k);
    }
}
```

### Distributed Trie (Parçalı Trie)

```
Sorun: Tek Trie instance → 2 GB → ölçeklenemiyor.
Çözüm: Trie'yi prefix'in ilk harfine göre shard et.

Trie Shard 1: a-f ile başlayan sorgular.
Trie Shard 2: g-m ile başlayan sorgular.
Trie Shard 3: n-s ile başlayan sorgular.
Trie Shard 4: t-z + sayı ile başlayan sorgular.

Routing:
  query[0] → hangi shard → o shard'a yönlendir.
  Türkçe özel harfler (ğ, ü, ş, ı, ö, ç) → shard 4.

Avantaj: her shard bağımsız, paralel sorgu.
Dezavantaj: cross-shard (q + boşluk olabilir, ama prefix için sorun değil).

İkinci seviye sharding:
  "a" prefix'i çok büyük → a-ae, af-ak, al-aq, ar-az.
  Consistent hashing ile dinamik shard.
```

---

## High-Level Mimari

```
         Client (browser / mobile)
              │ "jav" (debounced: 150ms)
              ▼
     ┌─────────────────┐
     │    CDN Edge     │  ← popüler prefix'ler cache'lendi (1sn TTL)
     └────────┬────────┘
              │ Cache MISS
              ▼
     ┌─────────────────┐
     │   API Gateway   │  ← rate limit, auth, routing
     └────────┬────────┘
              │
     ┌────────▼────────┐
     │  Autocomplete   │  ← stateless, horizontal scale
     │    Service      │
     └────┬───────┬────┘
          │       │
    ┌─────▼──┐  ┌─▼──────────┐
    │ Redis  │  │ Trie Service│  ← sharded, in-memory
    │ Cache  │  │ (cache miss)│
    └────────┘  └─────┬──────┘
                      │ miss
                      ▼
                 ┌──────────┐
                 │Trie Store│  ← S3 + RocksDB / DynamoDB
                 └──────────┘

Write / Data Pipeline:
  Search Logs → Kafka → [Flink stream: trending] → Redis trending cache
                      → [Spark batch: saatlik] → Trie Builder → Trie Store
                                                              → CDN + Redis invalidate

Blacklist Service:
  Tüm öneri → Blacklist check → temiz öneri → döndür.
```

---

## Read Path: Öneri Getirme

```
GET /autocomplete?q=jav&lang=tr&userId=u-123

Katmanlı cache stratejisi:

Katman 1 — CDN (< 5ms, global):
  CDN cache key: autocomplete:{lang}:{prefix}.
  TTL: 1sn (trending değişebilir) veya 60sn (stabil).
  HIT → döndür. MISS → origin'e git.

Katman 2 — Redis (< 2ms):
  Key: ac:{lang}:{prefix}
  HIT → top 10 JSON → döndür.
  MISS → Trie Service'e git.

Katman 3 — Trie Service (< 10ms):
  a. Shard routing: prefix[0] → hangi Trie node.
  b. traverse(prefix) → node.topK → sonuçlar.
  c. Redis'e yaz: SET ac:{lang}:{prefix} {json} EX 3600.
  d. Sonucu döndür.

Kişiselleştirme katmanı (< 5ms ek):
  Redis: ZREVRANGE user:{userId}:history 0 4 → son 5 arama.
  Global öneri listesine personal_boost uygula:
    personal_boost = user_search_count_for_query / max_count × 0.3
    final_score = global_score × (1 + personal_boost)
  Re-rank → ilk 10.

Blacklist filtresi (< 1ms):
  Öneri → Redis SET "blacklist" → SISMEMBER → yasak mı?
  Yasak → listeden çıkar → bir sonraki öneri ile doldur.

Sonuç:
  Top 10 öneri, sıralı, kişiselleştirilmiş, temiz.
  Toplam latency: CDN hit → 5ms, Redis hit → 10ms, Trie miss → 30ms.
```

---

## Write Path: Trie Güncelleme

### Batch Pipeline (Saatlik — Global Popülerlik)

```
1. Arama Logları → Kafka "search.events":
   {userId, sessionId, query: "java tutorial",
    resultClicked: "doc-123", timestamp, lang, country}

2. Spark Batch Job (saatlik, son 7 gün):
   a. Log dosyalarını S3'ten oku.
   b. Filtre: bot trafiği çıkar (User-Agent, hız anomalisi).
   c. Deduplication: aynı kullanıcı aynı sorguyu 1sn içinde → 1 say.
   d. Aggregation:
      query → count, click_through_rate, lang, country.
   e. CTR boost: tıklanan sonuç olan sorgular → score boost.
      score = count × (1 + 0.5 × ctr)
   f. Top 1000 sorgu per prefix → Trie build.

3. Trie Builder:
   Prefix-score çiftleri → Trie oluştur.
   Her node: topK list hesapla (DFS, min-heap).
   Trie → serialize → S3'e yaz (binary format, snappy compress).

4. Blue/Green Trie Swap:
   Yeni Trie: "trie-v2" dizinine yükle.
   Trie Service: yeni Trie'yi yükle → atomic pointer swap.
   Eski Trie: 30 dk sonra GC.

5. Cache Invalidation:
   Trie güncellendi → changed prefixes → Redis DEL (batch).
   CDN: Purge API → değişen prefix'lerin CDN cache'ini sil.
```

### Streaming Pipeline (5 Dakika — Trending)

```
Problem: Anlık viral sorgu (deprem, büyük haber) → saatlik batch çok geç.
Çözüm: Kafka → Flink → Redis trending cache.

Flink Streaming job (5 dakika sliding window):
  Input: Kafka "search.events".
  Window: son 1 saat, 5 dk slide.
  Aggregation: query → count, velocity (son 5 dk growth).
  Velocity = count_last_5min / count_previous_5min.
  Velocity > 3x: trending sinyal.

Trending score:
  trending_score = velocity_score × recency_weight
  recency_weight: son 5 dk → 1.0, 30 dk → 0.7, 1 saat → 0.4.

Redis trending cache:
  ZADD trending:{prefix}:{lang} {trending_score} {query}
  EXPIRE: 5 dk.
  
  Autocomplete: trending sonuçları normal öneri listesiyle merge et.
  Max 2 trending öneri (top 10'da), işaretlendi: "🔥 Trend".
  
  Saatlik batch güncellenmeden önce bile doğru trend gösterilir.
```

---

## Trie Depolama Alternatifleri

### Seçenek 1: Redis Hash (Basit, Production-Ready)

```
Key: ac:{lang}:{prefix}
Value: JSON list of top-10 [{query, score, meta}]

HSET ac:tr:jav
  suggestions '[{"q":"java","s":15000},{"q":"javascript","s":8000}]'
  updated_at 1705312800
EXPIRE ac:tr:jav 3600

Avantaj: basit, her prefix direkt key, operasyon O(1).
Dezavantaj: 10M prefix × memory → 5 GB (Redis Cluster ile yönetilebilir).
Uygulama: MVP ve orta ölçek.
```

### Seçenek 2: Elasticsearch Completion Suggester

```json
PUT /autocomplete-index
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "analyzer": "standard",
        "contexts": [
          {"name": "lang", "type": "category"},
          {"name": "country", "type": "category"}
        ]
      },
      "query_text": {"type": "keyword"},
      "score": {"type": "long"}
    }
  }
}

PUT /autocomplete-index/_doc/1
{
  "query_text": "java tutorial",
  "suggest": {
    "input": ["java tutorial", "java tut", "java tu"],
    "weight": 15000,
    "contexts": {"lang": ["en"], "country": ["US"]}
  }
}

GET /autocomplete-index/_search
{
  "suggest": {
    "autocomplete": {
      "prefix": "jav",
      "completion": {
        "field": "suggest",
        "size": 10,
        "contexts": {"lang": ["tr"]}
      }
    }
  }
}
```

```
Avantaj:
  ✓ Fuzzy matching (typo tolerance) dahil.
  ✓ Çok dilli, context-aware (ülke, dil).
  ✓ Highlight (eşleşen kısım bold).
  ✓ Filtering (adult content kaldır).
  ✓ Ölçeklenebilir (ES shard).

Dezavantaj:
  ✗ Daha yüksek latency (Redis'ten yavaş: ~10-30ms).
  ✗ Karmaşık operasyon.
  ✗ JVM memory.

Kullanım: büyük ölçek, fuzzy, çok dilli öneri.
```

### Seçenek 3: Trie In-Memory + Disk Snapshot

```
Process memory'de Trie → pointer traversal → en hızlı (< 1ms).
Periyodik: disk'e serialize → restart recovery.

Avantaj: O(L) arama (L = prefix uzunluğu), minimum overhead.
Dezavantaj:
  - Process restart → 2 GB Trie yükleme süresi (30sn).
  - Multi-instance: her pod kendi kopyası → memory çoğaltma.
  - Güncellemeler → her pod'u ayrı güncelle (coordinator gerekli).

DAWG (Directed Acyclic Word Graph) optimizasyonu:
  Trie: "nation", "national", "natural" → her node ayrı.
  DAWG: suffix paylaşımı → %50-70 bellek tasarrufu.
  "tion" suffix'i hem "na-tion" hem "na-tional" tarafından paylaşılır.

Kullanım: küçük/orta ölçek, maksimum hız, memory bütçesi var.
```

### Karşılaştırma

```
                Redis Hash    Elasticsearch    In-Memory Trie
Latency         < 2ms         10-30ms          < 1ms
Ölçek           Kolay (Clust) Kolay (Shard)    Zor (pod sync)
Fuzzy match     Hayır         Evet             Ek impl gerekir
Memory          5 GB (ext)    20+ GB           2 GB (per pod)
Güncelleme      Kolayca       Index update      Reload / swap
Multi-lang      Manual        Native            Manual
Öneri           Production    Büyük/kompleks    Yüksek perf
```

---

## Multi-Language / Unicode Normalizasyon

```
Sorun: Türkçe kullanıcı "şarkı" yazıyor, bazı klavyeler "sarki" yazıyor.
Çözüm: normalizasyon + ASCII folding.

Unicode normalizasyon:
  NFC: Türkçe özel harfleri tek code point → ş, ğ, ü, ö, ç, ı.
  Folding: ş → s, ğ → g, ü → u, ö → o, ç → c, ı → i.
  "şarkı" ve "sarki" → aynı normalized key: "sarki".

Index time normalizasyon:
  Sorgu index'e girerken: normalize et.
  "şarkı tutorial" → "sarki tutorial" olarak sakla.

Query time normalizasyon:
  Kullanıcı input: "şar" → normalize → "sar" → lookup.
  Öneriler: "sarki tutorial" → orijinal "şarkı tutorial" olarak göster.

Elasticsearch analyzer:
  "asciifolding" token filter:
  analysis.filter.turkish_ascii.type = asciifolding.
  "Ğüşçö" → "Gusco" (ASCII).

Dil bazlı trie:
  TR trie, EN trie, DE trie → ayrı ayrı.
  Kullanıcının tarayıcı dili → lang param → doğru trie.
  Mixed lang: "java tutorial" → en + tr → her ikisinde ara → merge.

Multi-word normalizasyon:
  "java  tutorial" (çift boşluk) → "java tutorial".
  "JAVA TUTORIAL" → lowercase → "java tutorial".
  Noktalama: "java, tutorial" → "java tutorial".
```

---

## Fuzzy Matching (Typo Tolerance)

```
Kullanıcı: "javva" yazdı (typo) → "java" önerin.
Kapsam dışı (out of scope) ama mimariye dahil etme yolu:

Edit Distance (Levenshtein):
  "javva" → "java": 1 edit (bir 'v' sil) → distance=1.
  Kabul: distance ≤ 1 (tek karakter hata) → öneri göster.
  Algoritma: DP, O(m×n) m,n = string uzunluğu.

BK-Tree (Burkhard-Keller Tree):
  Fuzzy index: kelimeler edit distance'a göre ağaç.
  Query: "javva" + threshold 1 → BK-Tree → O(log N) arama.
  Redis'te saklanamaz → ayrı fuzzy service.

Elasticsearch Fuzzy:
  "suggest": {"prefix": "javva", "completion": {"field": "suggest", "fuzzy": {"fuzziness": 1}}}
  Otomatik 1 edit mesafesi → ES halleder.

N-gram index:
  "java" → trigram: {jav, ava}.
  "javva" → trigram: {jav, avv, vva}.
  Overlap: {jav} → "java" aday → full match.
  Elasticsearch: ngram tokenizer → fuzzy-like davranış.

Phonetic (sesletim):
  "şarkı" ≈ "sarki" → Soundex/Metaphone → aynı fonetik kod.
  Türkçe için özel phonetic: ayrı implementation.

Autocomplete mimarisine entegrasyon:
  Ana path: exact prefix match (Redis/Trie) → hızlı.
  Fallback: sonuç < 3 → fuzzy service (ES/BK-Tree) → typo düzeltmeli.
  Latency budget: ana path: 10ms, fuzzy: +20ms = 30ms (hâlâ < 100ms).
```

---

## Trending Queries

```
"Günde 29,000 QPS ağırlıklı" → ama viral içerik anında trend olmalı.

Trending tespiti (Flink):
  5 dakika sliding window:
    query → (count_now, count_prev) → velocity = count_now / count_prev.
    velocity > 3.0 → trending.
    Mutlak minimum: count_now > 100 (küçük sayılarda sahte trend).

Trending cache (Redis):
  ZADD trending:global {velocity_score} {query} EX 300 (5 dk TTL).
  Prefix bazlı trending:
    "dep" prefix → "deprem ankara" trending → öne al.

Öneri listesine entegrasyon:
  Normal top-10 öneri + trending blend:
    trending_queries = ZREVRANGE trending:{prefix} 0 1 → top 2.
    final_list = trending_queries + global_top8 (dedupe).
  Trending sorgular: UI'de "🔥" ikonuyla işaretle.

Negatif trending (haber döngüsü sona erdi):
  velocity < 0.5 (düşüş) ve count_now < 1000 → trending listesinden çıkar.
  TTL zaten 5 dk → otomatik çıkar.

Trending manipulation önleme:
  Bot sorgular: aynı IP/session → 1 kez say.
  Rate limit: bir user 10 dk içinde aynı sorgu → 1.
  Velocity cap: max 10x (daha yüksekse bot şüphesi → inceleme).
```

---

## Blacklist / İçerik Moderasyonu

```
Problem: Viral kötü içerik → autocomplete'de çıkması → kullanıcı şikayeti.
Örnekler: hakaret içerikli sorgular, kişisel bilgi ifşası, illegal içerik.

Blacklist tipleri:
  Exact match: "xyzABC123" → yasaklı.
  Pattern match: regex ile (ör. kredi kartı numarası pattern).
  Category: "adult content" kategorisi → SafeSearch=ON → filtrele.

Redis SET (hızlı lookup):
  SADD blacklist "hakaret1" "hakaret2"
  Öneri listesi → her öneri → SISMEMBER blacklist → üyeyse çıkar.
  Lookup: O(1).
  Dinamik güncelleme: admin panel → SADD/SREM → anında etkili.

Bloom Filter (büyük blacklist):
  100M yasak kelime → 100M × SISMEMBER → hafıza sorun.
  Bloom filter: 1 GB → 100M kelime → %0.1 false positive kabul.
  False positive: bazen meşru öneri filtreleniyor → kabul edilebilir.

Otomatik tespit (ML):
  Arama logları: çok tıklanan ama şikayet alan sorgu → ML modeli işaretle.
  Human review: işaretlenen sorgular → moderatör onayı → blacklist.
  Reaction time: otomatik tespit 5 dk, human review 24 saat.

Ülke bazlı filtreleme:
  Bazı sorgular: bir ülkede yasak, diğerinde meşru.
  blacklist:{country} SET → ülke bazlı.
  Request: lang + country header → uygun blacklist.

Whitelist (özel durumlar):
  Blacklist'e girdi ama yanlış → whitelist → override.
  SISMEMBER whitelist → evetse blacklist check'i bypass et.
```

---

## Kişiselleştirme

```
Kullanıcı geçmişi:
  Son 30 günde "spring boot" arayan kullanıcı:
  "java" yazınca → "java spring boot" daha üstte çıksın.

Storage:
  Redis Sorted Set: user:{userId}:history
    ZADD user:{userId}:history {timestamp} {query}
    ZREMRANGEBYSCORE: 30 gün eskisini temizle.
    ZREVRANGE: son 20 sorguyu al.

Personal boost hesabı:
  user_queries: {"spring boot": 5, "java tutorial": 3, "maven": 2}
  Global öneri: [java, javascript, java tutorial, java spring boot]
  
  Her öneri için personal boost:
    "java spring boot" → user'ın "spring boot" geçmişi var → boost.
    boost = jaccard_similarity(user_history_terms, suggestion_terms).
    "java spring boot" ∩ {"spring boot"} = "spring boot" → yüksek overlap.
  
  final_score = global_score × (1 + boost × 0.5)
  Re-rank → kişiselleştirilmiş top-10.

Context-aware:
  Kullanıcı "elektronik" kategorisinde geziyorsa:
    "apple" → "apple iphone" (elektronik öncelikli), değil "apple elma suyu."
  Session context: Redis → son görülen kategori → autocomplete hint.

Coğrafi kişiselleştirme:
  Türkiye IP → Turkish queries boost: "java geliştirici kurs", "python öğren".
  Küresel popüler + yerel popüler blend.

Gizlilik:
  Kullanıcı: "Arama geçmişimi kullanma" → kişiselleştirme kapat.
  GDPR: kullanıcı silme → DEL user:{userId}:history.
  Anonim kullanıcı: kişiselleştirme yok → sadece global.
```

---

## Client-Side Optimizasyonlar

```
Debounce (ana optimizasyon):
  Her keystroke'ta istek ATMA → 150ms bekle → sonra at.
  "java" yazılırken: j → ja → jav → java → sadece "java" için istek.
  Etki: %60 QPS azalması.
  
  JavaScript:
  let timer;
  input.addEventListener('input', (e) => {
    clearTimeout(timer);
    timer = setTimeout(() => fetchSuggestions(e.target.value), 150);
  });

Throttle (debounce tamamlayıcısı):
  Max 1 istek / 100ms → hızlı yazıcı bile en fazla 10/sn.
  Debounce: bekleme → throttle: sınır.

Client-side cache (session):
  "jav" → öneri sonucu → localStorage / sessionStorage'a kaydet.
  Kullanıcı backspace yapıp "jav" tekrar yazarsa → cache'den.
  TTL: 60sn (tarayıcı cache).
  Cache key: {lang}:{prefix}.

Prefix filtering (yerel dar):
  "jav" → sonuçlar geldi: [java, javascript, ...].
  Kullanıcı "java" yazdı → yeni istek atmadan önce:
    client: "jav" sonuçlarından "java" ile başlayanları filtrele.
    Görüntüle → sonra arka planda "java" için gerçek istek at.
  Etki: anlık görüntüleme (0ms), gerçek sonuç arka planda gelir.

Abort previous request:
  "jav" isteği uçuşta → kullanıcı "java" yazdı → "jav" isteğini iptal et.
  AbortController API:
    const ctrl = new AbortController();
    fetch('/autocomplete?q=jav', {signal: ctrl.signal});
    // Yeni istek: ctrl.abort() → eski iptal.
  Network yükü azalır, race condition önlenir (yeni geldi eski geldi durumu).

Keyboard navigation:
  ↑ ↓ tuşları: öneri listesinde gezin.
  Enter: seçili öneriye git.
  Tab: ilk öneriyi kabul et (telefon klavyesi için önemli).
  Escape: listeyi kapat.
  Accessibility: ARIA live region → ekran okuyucu için.
```

---

## Ölçekleme

```
29,000 QPS (debounce sonrası 11,600 effective QPS) için:

CDN katmanı:
  Popüler prefix ("ja", "jav", "java"): milyonlarca kullanıcı aynı.
  CDN'de 1sn TTL → CDN hit oranı %70 → 8,120 QPS CDN.
  Backend: 3,480 QPS.

Redis Cluster:
  10M prefix × 5 GB → 3 primary + 3 replica cluster.
  Read: replica'lardan → 6 node → 3,480 / 6 = 580 QPS/node → trivial.
  Hit rate: %95 → backend: 174 QPS (Trie Service'e gider).

Trie Service:
  Stateless → horizontal scale.
  174 QPS → 2 instance → 87 QPS/instance → trivial.
  Trie: her instance kendi kopyası (in-memory, 2 GB).
  Güncelleme: saatlik → her instance yeni Trie yükle (rolling).

Auto-scaling trigger:
  Redis cache miss rate > %10 → Trie Service pod ekle.
  KEDA: Redis Queue length veya Kafka consumer lag → ölçek.

Geographic distribution:
  İstanbul + Frankfurt + Silicon Valley → GeoDNS → kullanıcı yakın region.
  Her region: kendi Redis + Trie Service + CDN.
  Trie build: global → S3 → her region indir → kendi Trie'sini yükle.
```

---

## A/B Testi ve Monitoring

```
A/B Test (ranking algoritması):
  Kontrol grubu (%50): global popularity score.
  Test grubu (%50): CTR boosted score (tıklanma oranı × count).
  
  Metrikler:
    Click-through rate (CTR): kullanıcı öneriyi seçti mi?
    Search completion rate: öneri ile arama tamamlandı mı?
    Position bias: #1 pozisyon her zaman daha çok tıklanır → düzeltme gerekir.
  
  İstatistiksel anlamlılık: p < 0.05, min 100K deney → 3-5 gün.
  Kazanan → %100'e çıkar.

Temel metrikler (Grafana dashboard):

  1. Click-through rate (CTR):
     Öneri sunuldu → tıklandı / sunuldu × 100.
     Hedef: > %30. < %20 → sıralama sorunu.

  2. Zero-result rate:
     Prefix için öneri yok → kullanıcı boş liste gördü.
     Hedef: < %5. Yüksekse → trie coverage eksik.

  3. Latency (P50, P95, P99):
     P99 < 100ms (SLA).
     Alarm: P99 > 150ms → scale out tetikle.

  4. Cache hit rate:
     Redis hit / toplam istek.
     Hedef: > %95. Düşükse → TTL uzat veya cache ısıt.

  5. Trending accuracy:
     Trending öneri gerçekten tıklandı mı?
     Düşük → trending algoritması ayarla.

Prometheus queries:
  rate(autocomplete_requests_total[5m])         → QPS
  histogram_quantile(0.99, autocomplete_latency) → P99
  rate(autocomplete_cache_hits[5m]) / rate(autocomplete_requests[5m]) → hit rate
  rate(autocomplete_zero_results[5m])           → zero-result rate
```

---

## Veri Modeli

```sql
-- Arama logları (ClickHouse — analitik, append-only)
CREATE TABLE search_events (
    event_id     UUID DEFAULT generateUUIDv4(),
    user_id      UUID,
    session_id   UUID,
    query        String,
    prefix       String,           -- query'nin prefix'i (aşamalı)
    lang         String,
    country      String,
    clicked_rank Int8,             -- NULL: tıklamadı, 0-9: tıklanan sıra
    clicked_doc  String,
    client_type  String,           -- WEB, ANDROID, IOS
    logged_at    DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(logged_at)
ORDER BY (lang, query, logged_at);

-- Aggregated query popularity (Spark batch çıktısı)
CREATE TABLE query_stats (
    query        VARCHAR(500) NOT NULL,
    lang         CHAR(5) NOT NULL,
    count_7d     BIGINT DEFAULT 0,
    count_24h    BIGINT DEFAULT 0,
    ctr          DECIMAL(5,4),        -- click-through rate
    score        DECIMAL(12,4),       -- final ranking score
    updated_at   TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (query, lang)
);

-- Redis key yapısı (referans):
-- ac:{lang}:{prefix}               → top-10 JSON (TTL: 3600sn)
-- trending:{lang}:{prefix}         → ZSET velocity_score → query (TTL: 300sn)
-- user:{userId}:history            → ZSET timestamp → query (max 20 item)
-- blacklist                        → SET (yasak sorgular)
-- blacklist:{country}              → SET (ülke bazlı yasak)
```

---

## Olası Sorunlar ve Çözümleri

### 1. Cold Start — Yeni Prefix İçin Öneri Yok

```
Sorun:
  Yeni dil eklendi (Japonca) veya yeni ürün kategorisi.
  Trie: henüz veri yok.
  Kullanıcı "ja" yazıyor → sonuç boş → kötü UX.

Çözüm:
  a) Seed data:
     Sözlük + Wikipedia'dan popüler kelimeler → başlangıç veri seti.
     E-ticaret: ürün katalogu → tüm ürün adları → Trie'ye ekle.
     "Yapay popülerlik": 100 başlangıç sayısı → kullanıcı aramasıyla gerçek sayı eklenir.

  b) Fallback katmanı:
     Trie sonuç yok → Elasticsearch full-text → "ja" ile başlayan döküman.
     ES: her zaman bir şey döndürür.
     Stale olabilir ama boş değil.

  c) Kopyala başlangıç:
     EN Trie hazır → JA Trie yok → EN önerileri göster.
     Yavaş yavaş JA Trie dolunca → EN'e olan bağımlılık azalır.

  d) Minimum threshold yerine progressive fill:
     count > 10 → Trie'ye ekle (gürültü filtresi).
     count > 0 → geçici Trie'ye ekle, yavaş yavaş promote et.
```

---

### 2. Trending Gecikti — Deprem Haberi 1 Saat Sonra Çıktı

```
Sorun:
  Sabah 08:00 büyük deprem → herkes "deprem istanbul" arıyor.
  Saatlik batch: 09:00'da trie güncellendi → 1 saat gecikme.
  Bu 1 saatte: autocomplete'de "deprem" önerisi yok → sıfır yardım.

Çözüm:
  a) Flink streaming (ana çözüm):
     5 dakika sliding window → velocity > 3x → trending.
     5 dk içinde "deprem istanbul" trending listesine giriyor.
     Autocomplete: trending blend → öneri listesinde görünür.

  b) Velocity eşiğini düşür (kritik event):
     Normal: velocity > 3x → trending.
     Acil durum keyword'ü (deprem, sel, yangın) → velocity > 1.5x → trending.
     Önceden bilinen acil durum kategorisi.

  c) Manual inject:
     İçerik ekibi: admin panelinden "deprem istanbul" → forced trending.
     Çok hızlı (< 1 dk) ama manuel.
     Kriz iletişimi için.

  d) News API entegrasyonu:
     Breaking news API → anahtar kelime çıkar → otomatik trending insert.
     Otomasyon + acil durum senaryosu.
```

---

### 3. Hot Key — "a" Prefix Milyonlarca İstek

```
Sorun:
  En çok kullanılan prefix: tek harf ("a", "i", "s").
  "a" → tüm dünyada milyonlarca kullanıcı → tek Redis key.
  Tek key → tek Redis node → hot key → bottleneck.

Çözüm:
  a) Key replication (local caching at autocomplete service):
     Autocomplete Service: her instance → kendi local cache (Caffeine).
     "ac:tr:a" → local cache'de (TTL 5sn).
     Redis'e gitmeden → local hit.
     10 instance × 5sn TTL → 10 farklı kopya → Redis yükü 10x azalır.

  b) Redis key sharding:
     "ac:tr:a" → 10 shard: "ac:tr:a:0" ... "ac:tr:a:9".
     Request gelince: random(0,9) → shard seç → okuma.
     10 node'a dağılır → 10x throughput.
     Yazma: tüm 10 shard güncelle (TTL-based eventual consistency).

  c) CDN (en etkili):
     "a", "i", "s" prefix → statik CDN'de (1sn TTL).
     Milyonlarca kullanıcı → CDN'den → Redis'e sıfır istek.
     CDN invalidation: Trie güncellendi → bu prefix'leri purge.

  d) Prefix'e göre tier:
     1-2 harf prefix: CDN + local cache (TTL 10sn, biraz stale OK).
     3-5 harf: Redis (TTL 3600sn).
     6+ harf: Trie Service (az kullanılan, cache'e alma).
```

---

### 4. Trie Build Çok Yavaş — Güncellemeler 3 Saatte Bir

```
Sorun:
  Veri büyüdü: 7 günlük log → 10B satır → Spark job 3 saat sürüyor.
  Saatlik güncelleme hedefi → 3 saate çıktı → veriler eskiyor.

Çözüm:
  a) Incremental update (delta):
     Tam rebuild yerine: son 1 saatteki değişiklikleri al.
     Değişen prefix'leri bul → sadece onları güncelle.
     Delta uygula → swap → çok daha hızlı.
     Sorun: incremental bug → yavaş yavaş yanlış birikir → haftada bir tam rebuild.

  b) Spark optimizasyonu:
     Partition: 7 günü günlük parçalara böl → paralel.
     Caching: her gün bir kez hesapla → son gün delta.
     Z-order clustering: (query, lang) sıralı → daha hızlı okuma.

  c) Veri azaltma:
     Count < 5 → trie'ye ekleme → gürültü düşürme + boyut küçülür.
     Bot trafiği: filtrele → %30 azalma.
     Deduplication: aynı session, aynı sorgu → 1 say.

  d) ClickHouse aggregation:
     Spark yerine ClickHouse → aynı aggregation 10x hızlı (OLAP için optimize).
     INSERT INTO query_stats SELECT query, count() FROM search_events
     WHERE logged_at >= now() - INTERVAL 7 DAY GROUP BY query.
     ClickHouse: saniyeler içinde → Trie build saniyeler.
```

---

### 5. Kötü Öneri — Spam / Saldırı Sorgular Öne Çıktı

```
Sorun:
  Botlar: "competitor sucks" sorgusunu 100,000 kez aradı → count yüksek.
  Trie: "competitor sucks" → top öneri → tüm kullanıcılar görüyor.
  Markaya zarar → PR krizi.

Çözüm:
  a) Bot filtresi (birincil savunma):
     Aynı IP → 1 saatte 1,000+ arama → bot.
     Aynı User-Agent → bot imzası → filtrele.
     Velocity anomaly: 10 dk'da 0 → 100,000 sorgu → anormal → sıfırla.
     Log'dan filtrele ÖNCE aggregation.

  b) CTR weight:
     count × ctr → score.
     Bot: arama yapar ama tıklamaz → CTR = 0 → score = 0.
     Gerçek popüler sorgu: tıklanır → CTR > 0.

  c) Velocity cap:
     Tek sorgunun count artışı: max 10,000/saat.
     Daha fazlası → cap'le → anormal artış skor'u etkilemesin.

  d) Human review queue:
     Hızla yükselen sorgu (velocity > 10x) → insan moderasyona gönder.
     Onay: modere et → trie'ye gir.
     Reddet → blacklist'e ekle.

  e) Reactive blacklist:
     Admin panel: şikayet alınan öneri → 1 tıkla → blacklist.
     Anında etkili (Redis SADD) → Trie rebuild beklemiyor.
```

---

### 6. Cache Invalidation Tutarsızlığı — Eski Öneri Gösterildi

```
Sorun:
  Trie güncellendi (saatlik): "corona" → artık öneri listesinde yok (moderasyon).
  CDN: eski "corona" öneri hâlâ cache'te (TTL: 1 saat).
  Sonraki 1 saat: "cor" yazınca "corona" görünüyor.

Çözüm:
  a) CDN Purge API:
     Trie güncellendi → hangi prefix'ler değişti? → CDN Purge.
     Cloudflare: POST /zones/{zoneId}/purge_cache → URL listesi.
     Hızlı: saniyeler içinde global propagation.

  b) Versioned cache key:
     CDN key: "ac:v42:tr:cor" (v42 = Trie versiyon numarası).
     Trie güncellendi → v43 → eski key'ler otomatik eskiyor.
     Dezavantaj: eski key'ler TTL bitene kadar CDN'de → yer kaplar.

  c) TTL stratejisi:
     Kısa TTL (30sn): moderasyona tabi içerik.
     Uzun TTL (1 saat): stabil, meşru prefix.
     "Riskli" prefix: dinamik TTL → daha kısa.

  d) Stale-while-revalidate:
     CDN: stale öneri hemen serve et + arka planda yenile.
     Cache-Control: s-maxage=60, stale-while-revalidate=30.
     Kullanıcı: anlık yanıt, 30-60sn içinde güncel veri.
```

---

### 7. Zero-Result Rate Yüksek — Kullanıcılar Boş Liste Görüyor

```
Sorun:
  Analitik: zero-result rate %15 → 7 kullanıcıdan 1'i boş liste.
  Kullanıcı: "jav sp" yazıyor → öneri yok → hayal kırıklığı → arama terk.
  Kapsam: uzun prefix'ler (6+ karakter) için Trie eksik.

Çözüm:
  a) Progressive prefix kısaltma:
     "jav sp" → öneri yok → "jav s" dene → yok → "jav " dene → sonuç.
     Client: giderek kısa prefix → bir şey bul → göster.
     "jav sp" için "jav s" sonuçlarını gösterirken "spring" vurgula.

  b) Trie coverage genişlet:
     Şu an: top-1000 query per prefix.
     Artır: top-5000 → uzun prefix'ler daha iyi kapsanır.
     Boyut tradeoff: 3 GB → 12 GB → Redis Cluster büyüt.

  c) Elasticsearch fallback:
     Trie: zero-result → ES full-text search → prefix match.
     ES: her zaman bir şey döndürür (wildcard veya phrase prefix).
     Latency: +20ms → kabul edilebilir (alternatif: hiç yok).

  d) Typo correction fallback:
     "jav sp" → sonuç yok → edit distance 1 → "java sp" → sonuç var.
     Öneri: "java spring" göster + "(java sp araması için)" açıklaması.

  e) Segmentation:
     "javaspring" (boşluksuz) → tokenize → "java spring" → Trie'de var.
     Segmentation library (Türkçe için zemberek) → kelime ayrıştırma.
```

---

## Trade-off Özeti

| Karar | Seçenek A | Seçenek B (Seçilen) | Gerekçe |
|-------|-----------|---------------------|---------|
| Cache backend | Memcached | Redis (Sorted Set + Hash) | Memcached: sadece basit kv; Redis: ZADD, expire, pub/sub |
| Trie update | Gerçek zamanlı | Batch (saatlik) + Flink trending | RT: yazma contention; Batch: basit, yeterli; Flink: trending hızı |
| Storage | In-memory Trie | Redis Hash + Trie Service | In-memory: sync sorunu; Redis: dağıtık, HA |
| Büyük ölçek | Trie only | CDN + Redis + Trie katmanı | Trie: 87K QPS kaldıramaz; CDN: %70 azaltır |
| Fuzzy matching | Trie exact | Trie exact + ES fallback | Exact: hızlı; Fallback: typo kullanıcısına da hizmet |
| Trending | Saatlik batch | Flink 5dk window | Saatlik: viral içerik 1h gecikmeli; Flink: 5dk |
| Kişiselleştirme | Yok | Global + user history boost | Yok: herkese aynı; Boost: hafif kişiselleştirme, az maliyet |
| Blacklist | Hardcoded list | Redis SET + dynamic admin | Hardcoded: deploy gerekir; Redis: anında etkili |
| Multi-language | Tek trie | Dil başına trie + normalize | Tek: dil karışımı; Ayrı: dil kalitesi |
| Bot filtreleme | Sonradan (trie'den) | Log aggregation öncesi | Sonradan: bot count trie'ye giriyor; Önce: temiz data |
| Client debounce | Yok | 150ms debounce + local cache | Yok: 87K QPS; Debounce+cache: 6K backend QPS |
