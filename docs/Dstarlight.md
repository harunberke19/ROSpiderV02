
```mermaid
flowchart TD
    %% Define Custom Color Scheme
    classDef init_style fill:#6c757d,stroke:#495057,stroke-width:2px,color:#fff; 
    classDef callback fill:#28a745,stroke:#1e7e34,stroke-width:2px,color:#fff; 
    classDef process fill:#007bff,stroke:#0056b3,stroke-width:2px,color:#fff; 
    classDef algorithm fill:#17a2b8,stroke:#117a8b,stroke-width:2px,color:#fff; 
    classDef decision fill:#ffc107,stroke:#d39e00,stroke-width:2px,color:#000; 
    classDef publish fill:#6f42c1,stroke:#543290,stroke-width:2px,color:#fff; 
    classDef data fill:#e2e3e5,stroke:#383d41,stroke-width:2px,color:#000;

    %% Node Initialization
    Start(["Başla"]) --> Init["Başlangıç Ayarları (Initialize):<br>Tüm g ve rhs değerleri = Sonsuz<br>rhs(Hedef) = 0<br>Hedefi Öncelik Kuyruğuna (U) Ekle"]

    %% Compute Shortest Path Loop
    Init --> ComputeLoop{"Planlama Gerekli mi?<br>(Başlangıç Tutarsız g != rhs<br>VEYA Kuyruk Top Key < Başlangıç Key)"}

    %% Planning Branch (Compute Shortest Path)
    ComputeLoop -- Evet --> PopU["Kuyruktan (U) En Düşük<br>Öncelikli Düğümü (u) Çıkar"]
    PopU --> CompareGRHS{"g(u) > rhs(u) mü?<br>(Aşırı Tutarlı Durum)"}

    CompareGRHS -- Evet --> OverConsistent["g(u) = rhs(u) Yap<br>Tüm Komşuların rhs Değerini Güncelle<br>(UpdateVertex)"]
    OverConsistent --> ComputeLoop

    CompareGRHS -- Hayır --> UnderConsistent["g(u) = Sonsuz Yap<br>u'nun ve Tüm Komşuların rhs'sini Güncelle<br>(UpdateVertex)"]
    UnderConsistent --> ComputeLoop

    %% Execution Branch
    ComputeLoop -- Hayır --> ExecutePhase["Rotayı Takip Et"]
    ExecutePhase --> CheckGoal{"Hedefe Ulaşıldı mı?"}

    CheckGoal -- Evet --> EndSuccess(["Bitiş: Hedefe Ulaşıldı!"])

    %% Sensor and Dynamic Obstacle Avoidance
    CheckGoal -- Hayır --> SensorScan[/"Sensör Verisini Oku (Lidar vb.)"/]
    SensorScan --> CheckChanges{"Haritada Değişiklik /<br>Yeni Bir Engel Var mı?"}

    CheckChanges -- Hayır --> MoveStep["Bir Adım İlerle<br>(Maliyeti en düşük düğüme geç)"]
    MoveStep --> CheckGoal

    %% Re-planning on Obstacle
    CheckChanges -- Evet --> UpdateGraph[("Graf/Grid Maliyetlerini Güncelle<br>km (Key Modifier) Değerini Artır")]
    UpdateGraph --> UpdateAffected["Etkilenen Kenarların ve Düğümlerin<br>rhs Değerlerini Yeniden Hesapla"]
    UpdateAffected --> ComputeLoop

    %% --- APPLY CLASSES SAFELY ---
    class Start,Init,EndSuccess init_style;
    class PopU,OverConsistent,UnderConsistent algorithm;
    class ComputeLoop,CompareGRHS,CheckGoal,CheckChanges decision;
    class ExecutePhase,MoveStep process;
    class UpdateGraph,UpdateAffected data;
    class SensorScan callback;
