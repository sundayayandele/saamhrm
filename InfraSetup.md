flowchart LR
    subgraph DNS[Global DNS / Anycast DNS]
        HC[Health checks & failover]
    end

    DNS -->|Traffic to healthy region(s)| R1LB[Region A Entry (Traefik)]
    DNS -->|Traffic to healthy region(s)| R2LB[Region B Entry (Traefik)]

    subgraph R1[Region A - SSDNodes]
        subgraph R1K8s[Kubernetes / K3s Cluster A]
            R1Traefik[Traefik Ingress]
            subgraph AppsA[Business Apps (Pods)]
                HRM_A[SaamHRM / SaamHCM]
                DORA_A[DORA Portal]
                LMS_A[Moodle]
                NC_A[Nextcloud]
                OtherA[Other Services]
            end

            R1Traefik --> HRM_A
            R1Traefik --> DORA_A
            R1Traefik --> LMS_A
            R1Traefik --> NC_A
            R1Traefik --> OtherA

            subgraph DataA[Data Layer A]
                PG_A[(PostgreSQL HA Cluster)]
                Kafka_A[(Kafka Cluster)]
                Debezium_A[Debezium CDC]
                KC_A[Kafka Connect]
                OS_A[(OpenSearch Cluster)]
                MinIO_A[(MinIO S3 Storage)]
            end

            HRM_A --> PG_A
            DORA_A --> PG_A
            LMS_A --> PG_A
            NC_A --> PG_A

            PG_A --> Debezium_A --> Kafka_A
            Kafka_A --> KC_A --> OS_A

            PG_A -->|Backups| MinIO_A
            OS_A -->|Snapshots| MinIO_A

            subgraph ObsA[Observability A]
                PromA[Prometheus + Alertmanager]
                GrafA[Grafana]
                LogsA[Fluent Bit/Filebeat]
            end

            AppsA --> PromA
            DataA --> PromA
            AppsA --> LogsA --> OS_A

            subgraph BackupA[Backup & DR A]
                Velero_A[Velero]
            end

            R1K8s --> Velero_A
            Velero_A --> MinIO_A
        end

        subgraph PanelA[Optional Classic Hosting A]
            VM_A[Virtualmin / CyberPanel VM]
            MailA[Mail, DNS, WP Sites]
        end

        R1LB --> R1Traefik
    end

    subgraph R2[Region B - SSDNodes]
        subgraph R2K8s[Kubernetes / K3s Cluster B]
            R2Traefik[Traefik Ingress]
            AppsB[Replica Apps (Pods)]
            PG_B[(PostgreSQL HA Cluster B)]
            Kafka_B[(Kafka Cluster B)]
            Debezium_B[Debezium CDC B]
            KC_B[Kafka Connect B]
            OS_B[(OpenSearch Cluster B)]
            MinIO_B[(MinIO S3 Storage B)]
            PromB[Prometheus B]
            GrafB[Grafana B]
            LogsB[Fluent Bit/Filebeat B]
            Velero_B[Velero B]
        end

        R2LB --> R2Traefik
        AppsB --> PG_B
        PG_B --> Debezium_B --> Kafka_B --> KC_B --> OS_B
        PG_B -->|Backups| MinIO_B
        OS_B -->|Snapshots| MinIO_B
        AppsB --> PromB
        DataB[(Data Layer B)]:::invisible
        DataB --- PG_B
        DataB --- Kafka_B
        DataB --- OS_B
        DataB --- MinIO_B
        AppsB --> LogsB --> OS_B
        R2K8s --> Velero_B
        Velero_B --> MinIO_B

        PanelB[Optional Panel B]
    end

    classDef invisible stroke-width:0,fill:none,color:none

    PG_A <-->|Async Replication / DR| PG_B
    OS_A <-->|Cross Cluster Replication (CCR)| OS_B
    Kafka_A <-->|MirrorMaker 2| Kafka_B