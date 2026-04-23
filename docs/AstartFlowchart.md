# Golem Strategic Planner - Flowchart

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
    Start([main]) --> Init[__init__: GolemStrategicPlanner\nSet Map, Targets, Pubs & Subs]

    %% Subscriptions (Entry Points)
    Init --> ScanTopic[/Topic: /scan/]
    Init --> OdomTopic[/Topic: /odom/]

    %% --- SCAN PROCESSING PIPELINE ---
    ScanTopic --> ScanCB[scan_cb: Receive LaserScan]
    ScanCB --> FilterScan[Filter ranges\n0.60 < r < max_view_dist]
    FilterScan --> WorldToGrid1[world_to_grid: Convert coordinates]
    WorldToGrid1 --> Inflate[Apply Inflation Cells]
    Inflate --> UpdateGrid[(Update Occupancy Grid\nself.grid)]

    %% --- ODOMETRY & PLANNING PIPELINE ---
    OdomTopic --> OdomCB[odom_cb: Receive Odometry]
    OdomCB --> UpdatePos[(Update Robot Pos & Yaw\nself.pos)]
    
    UpdatePos --> CheckReplan{Is it time to replan?\nor Path empty?}
    
    %% Planning Branch
    CheckReplan -- Yes --> Replan[replan: Triggered]
    Replan --> ClearFootprint[Clear robot's immediate footprint\nfrom Grid]
    ClearFootprint --> AStarAlgorithm[A* Algorithm\nFind path start to goal]
    
    AStarAlgorithm --> PathValid{Path Found?}
    PathValid -- Yes --> SavePath[(Save to self.current_path)]
    SavePath --> FormatPath[publish_path: Convert Grid\nPoints to PoseStamped]
    FormatPath --> PubPath[/Publish: /planned_path/]
    
    PathValid -- No --> LogWarn(Log Warning: Rota hesaplanamadı!)
    
    %% Driving Branch (Merged back)
    CheckReplan -- No --> ExecDrive
    PubPath --> ExecDrive
    LogWarn --> ExecDrive
    
    ExecDrive[execute_drive: Calculate Movement]
    ExecDrive --> DistGoal{Distance to Target\n< 0.4m?}
    
    %% Arrived
    DistGoal -- Yes --> StopAtGoal[Set Twist to 0.0]
    StopAtGoal --> PubCmd[/Publish: /cmd_vel/]
    
    %% Moving
    DistGoal -- No --> PathLen{current_path > 1?}
    
    PathLen -- No --> StopNoPath[Set Twist to 0.0\nWait for path]
    StopNoPath --> PubCmd
    
    PathLen -- Yes --> Lookahead[Get Lookahead Point\nCalculate Angle Error]
    Lookahead --> AngleErr{abs error > 0.4?}
    
    AngleErr -- Yes --> TurnInPlace[Turn in place\nlinear=0, angular=high]
    AngleErr -- No --> MoveFwd[Move towards target\nlinear=0.35, angular=adj]
    
    TurnInPlace --> PubCmd
    MoveFwd --> PubCmd

    %% --- APPLY CLASSES SAFELY ---
    %% Note: 'init' is a reserved keyword in some Mermaid versions, renamed to 'init_style'
    class Start,Init init_style;
    class ScanTopic,OdomTopic,UpdateGrid,UpdatePos,SavePath data;
    class ScanCB,OdomCB callback;
    class FilterScan,WorldToGrid1,Inflate,Replan,ClearFootprint,FormatPath,LogWarn,ExecDrive,StopAtGoal,StopNoPath,Lookahead,TurnInPlace,MoveFwd process;
    class AStarAlgorithm algorithm;
    class CheckReplan,PathValid,DistGoal,PathLen,AngleErr decision;
    class PubPath,PubCmd publish;
