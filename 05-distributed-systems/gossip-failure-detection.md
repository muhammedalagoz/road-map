# 05c — Gossip Protokolü & Failure Detection

---

## 1. Gossip Protokolü

### Ne?

**Gossip (Epidemic) Protocol:** Node'ların birbirlerine periyodik olarak durum bilgisi "fısıldadığı" dağıtık bilgi yayma protokolü. Her node rastgele bir komşusuna bilgi gönderir; bu bilgi salgın gibi tüm cluster'a yayılır.

```
Cluster: 6 node (A, B, C, D, E, F)

A → B: "C düştü, token ring güncellendi"
B → D: "C düştü"
B → E: "C düştü"
D → F: "C düştü"
... birkaç round sonra tüm node'lar bilir
```

### Neden?

**Çözdüğü problem:** Merkezi bir koordinatör olmadan cluster durumunu tüm node'lara dağıtmak. Merkezi koordinatör = SPOF; gossip = tamamen dağıtık.

```
Merkezi broadcast alternatifi:
  Leader → 1000 node'a mesaj → 1000 TCP bağlantısı → leader bottleneck
  Leader çökerse → bilgi yayılmaz

Gossip:
  Her node sadece k komşuya (genellikle 3) mesaj gönderir
  O(log N) round'da tüm cluster'a ulaşır
  Herhangi bir node çökerse → bilgi diğer yollardan yayılır
```

### Nasıl?

```
Round tabanlı çalışma (her T ms'de bir gossip turu):

Round 1:
  A → B: {A: alive, C: suspect}
  D → F: {D: alive, E: alive}

Round 2:
  B → C: {A: alive, C: suspect}  ← C kendini düzeltebilir
  B → G: {A: alive, C: suspect}
  F → A: {D: alive, E: alive}

... log(N) round sonra tüm cluster'a ulaştı

Gossip mesajı içeriği:
  {
    nodeId: "node-A",
    generation: 3,        // kaç kez restart edildi
    version: 142,         // monoton artan, her değişiklikte +1
    state: {
      "node-A": {status: "UP", load: 0.3, tokens: [...]},
      "node-C": {status: "DOWN", lastSeen: 1234567890},
      ...
    }
  }

Versiyon karşılaştırması:
  Yüksek version = daha güncel bilgi
  Her node aldığı bilgiyi kendi state'iyle merge eder
```

**Kullanan sistemler:**

```
Cassandra:
  - Node membership ve token ring bilgisi gossip ile yayılır
  - Her node 1 saniyede 3 komşuya gossip gönderir
  - nodetool gossipinfo → cluster gossip state'i gösterir

Redis Cluster:
  - Slot bilgisi, node durumu gossip ile paylaşılır
  - CLUSTER NODES → her node'un gördüğü cluster state

Consul:
  - Serf kütüphanesi (SWIM-based gossip) kullanır
  - Service health gossip ile cluster'a yayılır

Amazon DynamoDB:
  - Node membership gossip tabanlıdır

Kubernetes (Calico, Flannel):
  - BGP gossip ile routing bilgisi yayılır
```

### Trade-off?

| Avantaj | Dezavantaj |
|---------|-----------|
| SPOF yok (tamamen dağıtık) | Eventual consistency (anlık tutarsızlık) |
| O(log N) ölçeklenme | Yanlış bilgi da yayılabilir (gossip "dedikodu" gibi) |
| Ağ bölünmesine dayanıklı | Tam senkronizasyon süresi belirsiz |
| Node ekleme/çıkarma kolay | Network trafiği (sürekli gossip mesajları) |
| Basit implementasyon | Mesaj boyutu N ile büyür (tam state göndermek zor) |

---

## 2. Failure Detection

### Ne?

**Failure Detector:** Bir node'un çöküp çökmediğine karar veren mekanizma. "Cevap vermiyorsa çöktü mü yoksa yavaş mı?" sorusunu cevaplar.

### Neden?

Dağıtık sistemlerde ağ gecikmesi ve node yavaşlıkları ayırt edilemez. Çok erken "çöktü" demek → yanlış failover. Çok geç demek → system unavailable kalır.

```
Timeout-based detection sorunu:
  Node A → Node B'ye ping
  Node B cevap vermedi (5 saniye)
  Neden?
    a) Node B gerçekten çöktü
    b) Network gecikmesi (GC pause, yüksek yük)
    c) Partial partition

  Sabit timeout: ya çok erken (yanlış pozitif) ya da çok geç
```

### Nasıl?

#### 1. Heartbeat (Basit)

```
Yöntem 1: Active heartbeat
  Node B → her 1 saniyede Node A'ya ping atar
  Node A: "5 saniyedir ping yok → B çöktü"

Yöntem 2: Gossip-based heartbeat
  Her node kendi "heartbeat counter"ını gossip ile yayar
  Node A: "B'nin counter'ı 30 saniyedir güncellenmedi → suspect"

Sorun: Sabit threshold → network spike'larda yanlış pozitif
```

#### 2. Phi Accrual Failure Detector (Cassandra)

Sabit timeout yerine, geçmiş heartbeat istatistiklerine göre "çökme olasılığı" (φ) hesaplar.

```
Temel fikir:
  Heartbeat aralıklarını gözlemle (mean, std deviation)
  Şu anki gecikmeye göre istatistiksel olasılık hesapla

φ formülü:
  φ(t) = -log₁₀(P(T_now - T_last_heartbeat))
  
  P: bu kadar gecikme ne olasılıkla normal?
  Normal dağılım varsayımıyla hesaplanır

φ değeri ne anlama gelir?
  φ < 1  → neredeyse kesinlikle sağlıklı
  φ = 1  → %10 olasılıkla çöktü
  φ = 2  → %1 olasılıkla çöktü
  φ = 3  → %0.1 olasılıkla çöktü (genellikle suspect eşiği)
  φ = 8  → %0.000001 → kesinlikle çöktü (down eşiği)

Cassandra varsayılanları:
  phi_convict_threshold = 8  → bu değerin üstünde → "DOWN"

Avantaj:
  Ağ gecikmesi arttıysa → threshold otomatik yükseliyor
  GC pause sırasında yanlış "DOWN" kararı vermiyor
```

```java
// Cassandra benzeri Phi Accrual implementasyonu (konsept)
class PhiAccrualFailureDetector {
    
    private final Map<String, HeartbeatHistory> histories = new ConcurrentHashMap<>();
    private static final double PHI_THRESHOLD = 8.0;
    
    void heartbeat(String nodeId) {
        histories.computeIfAbsent(nodeId, k -> new HeartbeatHistory())
            .add(System.currentTimeMillis());
    }
    
    boolean isAlive(String nodeId) {
        HeartbeatHistory history = histories.get(nodeId);
        if (history == null) return true;
        
        double phi = history.calculatePhi(System.currentTimeMillis());
        return phi < PHI_THRESHOLD;
    }
    
    static class HeartbeatHistory {
        private final Deque<Long> intervals = new ArrayDeque<>();
        private long lastTimestamp = -1;
        
        void add(long now) {
            if (lastTimestamp > 0) {
                intervals.addLast(now - lastTimestamp);
                if (intervals.size() > 1000) intervals.pollFirst(); // max 1000 sample
            }
            lastTimestamp = now;
        }
        
        double calculatePhi(long now) {
            if (intervals.isEmpty() || lastTimestamp < 0) return 0;
            
            long timeSinceLast = now - lastTimestamp;
            double mean = intervals.stream().mapToLong(Long::longValue).average().orElse(1000);
            double stdDev = calculateStdDev(mean);
            
            // Normal dağılım CDF approximation
            double y = (timeSinceLast - mean) / stdDev;
            double e = Math.exp(-y * (1.5976 + 0.070566 * y * y));
            double p = timeSinceLast > mean ? e / (1 + e) : 1 - 1 / (1 + e);
            
            return -Math.log10(p);
        }
        
        private double calculateStdDev(double mean) {
            double variance = intervals.stream()
                .mapToDouble(i -> Math.pow(i - mean, 2))
                .average().orElse(1000);
            return Math.sqrt(variance);
        }
    }
}
```

#### 3. SWIM (Scalable Weakly-consistent Infection-style Membership) — Consul/Serf

```
Gossip + Failure Detection kombinasyonu:

Ping-Ack:
  A → B: ping
  B → A: ack (OK)

Ping-Request (indirect):
  A → B: ping (cevap yok)
  A → C, D, E: "B'ye ping at bakalım"
  C → B: ping
  B → C: ack
  C → A: "B'den ack aldım" → B sağlıklı (ağ A-B arası bozuk olabilir)

Suspect:
  A → C, D, E: ping-req gönderdi, hiçbirinden ack yok
  A: "B suspect" → gossip ile yay
  B 10 saniye içinde kendini "alive" olarak yaymazsa → "DEAD"

Avantajlar:
  - False positive az (indirect check ile doğrulama)
  - Gossip ile ölçeklenir (N node, O(log N) detection time)
  - Consul, Serf bu protokolü kullanır
```

---

## Ne zaman?

```
Gossip kullan:
✓ Büyük cluster (100+ node) membership ve state yayma
✓ Merkezi koordinatör istemiyorsan
✓ Eventual consistency kabul edilebiliyorsa
✓ Service discovery (Consul, Redis Cluster)
✓ Distributed database node durumu (Cassandra)

Phi Accrual kullan:
✓ Değişken ağ gecikmesi olan ortamlar
✓ GC pause'un sık olduğu JVM uygulamaları
✓ Yanlış pozitif "down" kararından kaçınmak

Basit heartbeat yeterli:
✓ Küçük cluster (< 20 node)
✓ Ağ gecikmesi sabit ve düşük
✓ Kubernetes liveness probe (basit HTTP check)
```

---

## Trade-off?

| | Gossip | Phi Accrual | SWIM |
|-|--------|-------------|------|
| **Ölçekleme** | O(log N) | Peer-to-peer | O(log N) |
| **False Positive** | Orta | Düşük | Çok düşük |
| **Hız** | Orta | Hızlı | Hızlı |
| **Karmaşıklık** | Düşük | Orta | Yüksek |
| **Kullanım** | Cassandra, Redis | Cassandra, Akka | Consul, Serf |
