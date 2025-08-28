```mermaid
flowchart LR
    subgraph Dev["Desarrollador / PR / Tag"]
        A[(kcd_colombia_webapp_color<br/>App Repo)]
    end

    subgraph CI["GitHub Actions (CI/CD)"]
        B[Build Imagen<br/>Push a ECR]
        C[Actualizar image_tag<br/>en values]
    end

    subgraph Ops["Repos GitOps"]
        D[(kcd_colombia_env<br/>Env / Helm values)]
        E[(kcd_colombia_platform<br/>ArgoCD apps & addons)]
        F[(kcd_colombia_infra<br/>Terraform / Infra)]
    end

    subgraph Cluster["EKS + ArgoCD"]
        G[[ArgoCD Controller]]
        H[(Helm Release<br/>webapp-color)]
    end

    A -->|Push / Tag| CI
    CI --> B -->|Imagen| ECR[(ECR)]
    B --> C -->|"Commit values (image tag)"| D
    E -->|App & Project defs| G
    D -->|Valores + Chart ref| G
    F -->|Provisiona EKS/ALB/IRSA| Cluster

    G -->|Sync| H
    H -->|Deploy Pods usan| ECR

    classDef repo fill:#1d70b8,stroke:#0b3c61,color:#ffffff;
    classDef control fill:#555555,stroke:#333333,color:#ffffff;
    classDef artifact fill:#ffffff,stroke:#555555,color:#222222;
    class A,D,E,F repo
    class G control
    class B,C,H,ECR artifact
````