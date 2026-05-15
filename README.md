   flowchart TD
    Start([Staff opens dashboard]) --> Auth{Valid JWT?}
    Auth -- No --> Login["/login — enter credentials"]
    Login --> Verify{Credentials OK?}
    Verify -- No --> LoginFail["Inline error<br/>audit: login_failed"] --> Login
    Verify -- Yes --> Token["JWT issued, role resolved<br/>audit: login_success"]
    Auth -- Yes --> Dash
    Token --> Dash[[Dashboard hub]]

    Dash --> Stats["See stats, SOP pipeline,<br/>collections, activity"]
    Stats --> Pick{What does the user do?}

    Pick -->|Open a collection| Detail["Collection detail:<br/>communication timeline + agent events"] --> Dash
    Pick -->|View audit / activity| AuditView["Audit log & agent runs"] --> Dash

    Pick -->|"Run Agent now (approver+)"| RoleChk{Role >= approver?}
    RoleChk -- No --> Denied["403 — control hidden/disabled"] --> Dash
    RoleChk -- Yes --> AgentRun

    Pick -->|"New collection (approver+)"| NewCol["Add invoice<br/>audit: collection_created"] --> Dash
    Pick -->|"Mark paid (approver+)"| MarkPaid["Status → resolved<br/>audit: collection_marked_paid"] --> Dash

    Pick -->|"Scheduler mode (admin)"| Sched["Set manual / user_input / autopilot<br/>audit: scheduler_mode_changed"] --> Dash
    Pick -->|"Manage users (admin)"| Users["Create users & roles"] --> Dash

    subgraph Engine["Automated SOP engine (LangGraph)"]
      direction TB
      Trigger{Trigger}
      Trigger -->|Manual button| AgentRun
      Trigger -->|"Scheduler (user_input / autopilot)"| AgentRun
      AgentRun["Read replies → classify →<br/>evaluate SOP stage → draft"] --> Gate{High-stakes stage<br/>OR user_input mode?}
      Gate -- No --> AutoSend["Send email<br/>audit: email_sent"]
      Gate -- Yes --> Queue["Hold as pending_approval"]
    end

    AutoSend --> Dash
    Queue --> PA["Pending Approvals panel"]
    PA --> Review{Approver decision}
    Review -->|Approve| Send2["Send email<br/>audit: approval_approved"] --> Dash
    Review -->|Reject| Rej["Mark rejected<br/>audit: approval_rejected"] --> Dash

    Dash --> Logout{Logout / token expiry?}
    Logout -- Yes --> Login
    Logout -- No --> Stats

    classDef gate fill:#7c3aed22,stroke:#7c3aed;
    classDef auto fill:#10b98122,stroke:#10b981;
    class Gate,Review,RoleChk gate
    class AutoSend,Send2 auto
