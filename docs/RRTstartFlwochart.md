# Golem RRT* Planner - Flowchart

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
    Start([main]) --> Init[__init__: GolemRRTStarVisual\nSet Sim Time, Pubs/Subs, TF Broadcaster]

    %% Subscriptions (Entry Points)
    Init --> ScanTopic[/Topic: /scan/]
    Init --> OdomTopic[/Topic: /odom/]

    %% --- SCAN PROCESSING PIPELINE ---
    ScanTopic --> ScanCB[scan_cb: Receive LaserScan]
    ScanCB --> FilterScan[Filter ranges\n0.60 < r < max_view_dist]
    FilterScan --> WorldToGrid[world_to_grid: Convert coordinates]
    WorldToGrid --> Inflate[Apply Inflation Cells]
    Inflate --> UpdateGrid[(Update Occupancy Grid\nself.grid)]

    %% --- ODOMETRY & PLANNING PIPELINE ---
    OdomTopic --> OdomCB[odom_cb: Receive Odometry]
    
    %% TF and Pos Updates
    OdomCB --> ExtractPos[Extract Position & Orientation]
    ExtractPos --> BroadcastTF[Send TF Transform\nodom -> base_link]
    BroadcastTF --> UpdatePos[(Update Robot Pos & Yaw\nself.pos)]
    
    UpdatePos --> CheckReplan{Is it time to replan?\nor Path empty?}
    
    %% Planning Branch
    CheckReplan -- Yes --> Replan[replan: Triggered]
    Replan --> ClearFootprint[Clear robot's immediate footprint\nfrom Grid]
    ClearFootprint --> RRTStarAlgo[RRT* Algorithm\nSample, Steer, Check Collision, Rewire]
    
    RRTStarAlgo --> PubTreeProc[publish_tree: Generate Marker lines\nfor node tree]
    PubTreeProc --> PubTreeTopic[/Publish: /rrt_tree/]
    
    RRTStarAlgo --> PathValid{Path Found?}
    PathValid -- Yes --> SavePath[(Save to self.current_path)]
    SavePath --> FormatPath[publish_path: Convert Grid\nPoints to PoseStamped]
    FormatPath --> PubPath[/Publish: /planned_path/]
    
    PathValid -- No --> LogWarn(Log Warning: RRT* Rota Bulamadı!)
    
    %% Driving Branch (Merged back)
    CheckReplan -- No --> ExecDrive
    PubPath --> ExecDrive
    LogWarn --> ExecDrive
    PubTreeTopic --> ExecDrive
    
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
    class Start,Init init_style;
    class ScanTopic,OdomTopic,UpdateGrid,UpdatePos,SavePath data;
    class ScanCB,OdomCB callback;
    class FilterScan,WorldToGrid,Inflate,ExtractPos,BroadcastTF,ClearFootprint,FormatPath,PubTreeProc,LogWarn,ExecDrive,StopAtGoal,StopNoPath,Lookahead,TurnInPlace,MoveFwd process;
    class RRTStarAlgo algorithm;
    class CheckReplan,PathValid,DistGoal,PathLen,AngleErr decision;
    class PubPath,PubTreeTopic,PubCmd publish;
