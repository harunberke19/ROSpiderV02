
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
    Start(["Başla"]) --> Init["Başlangıç Ayarları:<br>Tüm Düğümlerin Uzaklığı = Sonsuz<br>Dist(Başlangıç) = 0"]

    %% Data Setup
    Init --> SetupData[("Öncelik Kuyruğu PQ Oluştur<br>Başlangıç Düğümünü Ekle")]

    %% Main Loop
    SetupData --> CheckPQ{"PQ Boş mu?"}
    
    %% Empty Queue / No Path or Done
    CheckPQ -- Evet --> EndNoPath(["Bitiş: Hedefe Ulaşılamadı / Tüm Düğümler Tarandı"])
    
    %% Extract Minimum Distance Node
    CheckPQ -- Hayır --> ExtractMin["PQ'dan Minimum Uzaklığa Sahip<br>U Düğümünü Çıkar"]
    
    %% Target Check (Early Exit for Path Planning)
    ExtractMin --> IsTarget{"U Hedef Düğüm mü?"}
    
    %% Target Reached
    IsTarget -- Evet --> ReconstructPath["Önceki Düğümleri İzleyerek<br>Rotayı Oluştur"]
    ReconstructPath --> EndSuccess(["Bitiş: Rota Bulundu"])
    
    %% Process Neighbors
    IsTarget -- Hayır --> GetNeighbors["U'nun Komşularını (V) Al"]
    GetNeighbors --> HasNeighbors{"İncelenecek Başka<br>Komşu (V) Var mı?"}
    
    %% Neighbor Loop
    HasNeighbors -- Hayır --> CheckPQ
    HasNeighbors -- Evet --> CalcDist["Geçici Uzaklığı Hesapla:<br>Alt = Dist(U) + Ağırlık(U, V)"]
    
    %% Relaxation Step
    CalcDist --> CompareDist{"Alt < Dist(V) mi?"}
    
    %% Update Values
    CompareDist -- Evet --> UpdateDist["Güncelle: Dist(V) = Alt<br>Önceki(V) = U"]
    UpdateDist --> UpdatePQ[("V'yi PQ'ya Ekle / Önceliğini Güncelle")]
    UpdatePQ --> HasNeighbors
    
    %% Skip Update
    CompareDist -- Hayır --> HasNeighbors

    %% --- APPLY CLASSES SAFELY ---
    class Start,Init,EndNoPath,EndSuccess init_style;
    class SetupData,UpdatePQ data;
    class ReconstructPath,GetNeighbors,UpdateDist process;
    class ExtractMin,CalcDist algorithm;
    class CheckPQ,IsTarget,HasNeighbors,CompareDist decision;
