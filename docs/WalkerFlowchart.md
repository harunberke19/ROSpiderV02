# Hexapod Autonomous Walker - Flowchart

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
    Start([main]) --> Init[__init__ HexapodAutonomousWalker\nSet Pubs Subs and Timer]

    %% Subscriptions & Timers
    Init --> CmdTopic[/Topic cmd_vel/]
    Init --> WalkTimer((0.1s Timer))

    %% --- CMD_VEL PIPELINE ---
    CmdTopic --> CmdCB[cmd_vel_callback Receive Twist]
    CmdCB --> UpdateVels[(Update Variables\nlinear_x and angular_z)]

    %% --- WALK CYCLE PIPELINE ---
    WalkTimer --> WalkCycle[walk_cycle Triggered every 0.1s]
    WalkCycle --> CheckMove{abs linear_x < 0.01\nAND\nabs angular_z < 0.01}

    %% Stopped State
    CheckMove -- Yes --> StopWalk[Return Do nothing - Robot stands still]

    %% Moving State
    CheckMove -- No --> CheckTurn{abs angular_z > 0.1}

    %% Calculate Directions
    CheckTurn -- Yes --> TurnCalc[Turning Logic\nfwd_R and fwd_L set to\nopposite values based on Z]
    CheckTurn -- No --> StraightCalc[Straight Logic\nfwd_R and fwd_L set to\nsame values based on X]

    TurnCalc --> DefineAngles
    StraightCalc --> DefineAngles

    %% Build the Gait
    DefineAngles[Define Leg Angles\nlift_R push_R lift_L push_L]
    DefineAngles --> CheckPhase{self.phase == 0}

    %% Tripod Gait Phase Toggle
    CheckPhase -- Yes --> Phase0[Set msg.data for Phase 0\nself.phase = 1]
    CheckPhase -- No --> Phase1[Set msg.data for Phase 1\nself.phase = 0]

    Phase0 --> PubJoints
    Phase1 --> PubJoints

    %% Publish Command
    PubJoints[/Publish Float64MultiArray to\njoint_group_controller/]

    %% --- APPLY CLASSES SAFELY ---
    class Start,Init init_style;
    class CmdTopic,UpdateVels data;
    class CmdCB callback;
    class WalkTimer,WalkCycle,StopWalk,TurnCalc,StraightCalc,DefineAngles,Phase0,Phase1 process;
    class CheckMove,CheckTurn,CheckPhase decision;
    class PubJoints publish;
