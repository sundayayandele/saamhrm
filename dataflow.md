sequenceDiagram
    participant App as Business App (SaamHRM / DORA / etc.)
    participant DB as PostgreSQL HA Cluster
    participant Deb as Debezium CDC
    participant K as Kafka
    participant KC as Kafka Connect
    participant OS as OpenSearch

    App->>DB: INSERT / UPDATE / DELETE (OLTP writes)
    DB-->>Deb: WAL / logical changes
    Deb->>K: Change events (topics: hrm.employees, payroll.tx, etc.)
    K-->>KC: Stream messages from topics
    KC->>OS: Index / update documents (employees_index, payroll_index, etc.)
    App->>OS: Search / dashboards / analytics queries
    OS-->>App: Fast search results