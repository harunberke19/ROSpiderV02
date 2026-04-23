```mermaid
graph TD
    classDef callback fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px,color:#000;
    classDef process fill:#e8f5e9,stroke:#43a047,stroke-width:2px,color:#000;
    classDef decision fill:#fff9c4,stroke:#fbc02d,stroke-width:2px,color:#000;
    classDef action fill:#ffebee,stroke:#e53935,stroke-width:2px,color:#000;
    classDef rrt fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px,color:#000;

    subgraph LaserScan_Processing [Asynchronous Callback: /scan]
        SCAN_START(RECEIVE LaserScan) ::: callback
        SCAN_ITERATE[Iterate Ranges] ::: process
        SCAN_VALID{Valid Range?<br/>0.60m < r < 6.0m} ::: decision
        SCAN_WORLD[Calculate World Coordinates<br/>ox, oy] ::: process
        SCAN_GRID[Convert to Grid Coordinates<br/>gx, gy] ::: process
        SCAN_INFLATE[Apply 5-Cell Inflation] ::: process
        SCAN_UPDATE[(Update Occupancy Grid)] ::: action

        SCAN_START --> SCAN_ITERATE
        SCAN_ITERATE --> SCAN_VALID
        SCAN_VALID -- Yes --> SCAN_WORLD
        SCAN_VALID -- No --> SCAN_ITERATE
        SCAN_WORLD --> SCAN_GRID
        SCAN_GRID --> SCAN_INFLATE
        SCAN_INFLATE --> SCAN_UPDATE
    end

    subgraph Odometry_and_Main_Loop [Asynchronous Callback: /odom]
        ODOM_START(RECEIVE Odometry) ::: callback
        ODOM_TF[Broadcast TF:<br/>odom to base_link] ::: action
        ODOM_CALC[Calculate Robot Position<br/>x, y, yaw] ::: process
        ODOM_TIME_CHECK{Replan Needed?<br/>Interval > 2.0s OR Path Empty} ::: decision

        ODOM_START --> ODOM_TF
        ODOM_TF --> ODOM_CALC
        ODOM_CALC --> ODOM_TIME_CHECK
    end

    subgraph RRT_Star_Algorithm [Sub-process: replan]
        RRT_CLEAR[Clear 7x7 Grid Area<br/>Around Robot] ::: rrt
        RRT_INIT[Initialize Node List<br/>Start Node] ::: rrt
        
        RRT_LOOP{Max Iterations Reached?<br/>i < 2000} ::: decision
        RRT_SAMPLE[Sample Random Point<br/>5% Goal Bias] ::: rrt
        RRT_NEAREST[Find Nearest Node<br/>in Tree] ::: rrt
        RRT_STEER[Steer Towards Sample<br/>Step Size = 0.4] ::: rrt
        RRT_COLLISION{Collision Free?<br/>Point & Line Check} ::: decision
        RRT_ADD[Add Node to Tree] ::: rrt
        RRT_REWIRE[Rewire Near Nodes<br/>Optimize Cost] ::: rrt
        RRT_GOAL_CHECK{Distance to Goal<br/><= 0.5m?} ::: decision
        
        RRT_EXTRACT[Generate Final Course<br/>Backtrack Parents] ::: rrt
        RRT_PUB_TREE[Publish RRT Tree Marker<br/>/rrt_tree] ::: action
        RRT_PATH_CHECK{Valid Path Found?} ::: decision
        RRT_SAVE_PATH[Save current_path<br/>Publish to /planned_path] ::: action
        RRT_FAIL[Log Warning:<br/>'RRT* Rota Bulamadı!'] ::: action

        ODOM_TIME_CHECK -- Yes --> RRT_CLEAR
        RRT_CLEAR --> RRT_INIT
        RRT_INIT --> RRT_LOOP
        
        RRT_LOOP -- No --> RRT_SAMPLE
        RRT_SAMPLE --> RRT_NEAREST
        RRT_NEAREST --> RRT_STEER
        RRT_STEER --> RRT_COLLISION
        RRT_COLLISION -- No --> RRT_LOOP
        RRT_COLLISION -- Yes --> RRT_ADD
        RRT_ADD --> RRT_REWIRE
        RRT_REWIRE --> RRT_GOAL_CHECK
        RRT_GOAL_CHECK -- No --> RRT_LOOP
        RRT_GOAL_CHECK -- Yes --> RRT_EXTRACT
        
        RRT_LOOP -- Yes --> RRT_PUB_TREE
        RRT_EXTRACT --> RRT_PUB_TREE
        
        RRT_PUB_TREE --> RRT_PATH_CHECK
        RRT_PATH_CHECK -- Yes --> RRT_SAVE_PATH
        RRT_PATH_CHECK -- No --> RRT_FAIL
    end

    subgraph Execute_Drive [Sub-process: execute_drive]
        DRIVE_DIST{Distance to Goal<br/>< 0.4m?} ::: decision
        DRIVE_STOP_1[Stop Robot<br/>v=0, w=0] ::: action
        DRIVE_PATH_CHECK{Valid Path Exists?<br/>> 1 Node} ::: decision
        DRIVE_STOP_2[Stop Robot<br/>v=0, w=0] ::: action
        
        DRIVE_LOOKAHEAD[Calculate Lookahead<br/>Max 3 Steps] ::: process
        DRIVE_ERROR[Calculate Heading Error<br/>err = target_yaw - current_yaw] ::: process
        DRIVE_ERR_CHECK{Absolute Error<br/>> 0.4 rad?} ::: decision
        
        DRIVE_ROTATE[Rotate in Place<br/>v=0.0, w=±0.8] ::: action
        DRIVE_FORWARD[Move Forward & Adjust<br/>v=0.35, w=err*1.5] ::: action
        DRIVE_PUB[Publish Twist<br/>/cmd_vel] ::: action

        ODOM_TIME_CHECK -- No --> DRIVE_DIST
        RRT_SAVE_PATH --> DRIVE_DIST
        RRT_FAIL --> DRIVE_DIST

        DRIVE_DIST -- Yes --> DRIVE_STOP_1
        DRIVE_STOP_1 --> DRIVE_PUB
        
        DRIVE_DIST -- No --> DRIVE_PATH_CHECK
        DRIVE_PATH_CHECK -- No --> DRIVE_STOP_2
        DRIVE_STOP_2 --> DRIVE_PUB
        
        DRIVE_PATH_CHECK -- Yes --> DRIVE_LOOKAHEAD
        DRIVE_LOOKAHEAD --> DRIVE_ERROR
        DRIVE_ERROR --> DRIVE_ERR_CHECK
        
        DRIVE_ERR_CHECK -- Yes --> DRIVE_ROTATE
        DRIVE_ERR_CHECK -- No --> DRIVE_FORWARD
        
        DRIVE_ROTATE --> DRIVE_PUB
        DRIVE_FORWARD --> DRIVE_PUB
    end

    SCAN_UPDATE -. "Occupancy Grid Read By" .-> RRT_COLLISION
```
