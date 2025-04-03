# TorrentBox Kubernetes Stack

This project deploys a suite of media management applications (Sonarr, Radarr, Lidarr, Bazarr, Prowlarr, qBittorrent, etc.) along with supporting services like Jellyfin and Jellyseerr onto a Kubernetes cluster.

It was originally converted from a Docker Compose setup. The Gluetun VPN sidecar was initially included but removed due to persistent connectivity issues in the target environment. **Therefore, applications currently run directly on the Kubernetes network without VPN protection.**

Configuration for each application is persisted using Kubernetes `hostPath` volumes, mounting subdirectories from the `./config` directory on the host machine where these manifests reside. Shared media and download data uses a PersistentVolumeClaim (`main-data-pvc`).

## Architecture

The stack consists of several Deployments, each managing pods for a specific application. Communication between applications happens via Kubernetes Services (mostly ClusterIP type). Services intended for direct user access (Jellyfin, Jellyseerr, Sabnzbd) are exposed via NodePort.

```mermaid
graph TD
    subgraph "Kubernetes Cluster (Namespace: torrentbox)"
        subgraph "User Access"
            U[User Browser]
            PF[kubectl port-forward]
            NP[Node IP + NodePort]
        end

        subgraph "Media Management Apps"
            QB_SVC[Service: qbittorrent (ClusterIP)]; QB_POD[Pod: qbittorrent (Deployment)];
            SO_SVC[Service: sonarr (ClusterIP)]; SO_POD[Pod: sonarr (Deployment)];
            RA_SVC[Service: radarr (ClusterIP)]; RA_POD[Pod: radarr (Deployment)];
            LI_SVC[Service: lidarr (ClusterIP)]; LI_POD[Pod: lidarr (Deployment)];
            BA_SVC[Service: bazarr (ClusterIP)]; BA_POD[Pod: bazarr (Deployment)];
            PR_SVC[Service: prowlarr (ClusterIP)]; PR_POD[Pod: prowlarr (Deployment)];
            QR_POD[Pod: qbitrr (Deployment)];
            DP_POD[Pod: doplarr (Deployment)];
            FS_SVC[Service: flaresolverr (ClusterIP)]; FS_POD[Pod: flaresolverr (Deployment)];
        end

        subgraph "Media Serving / Requesting"
            JF_SVC[Service: jellyfin (NodePort)]; JF_POD[Pod: jellyfin (Deployment)];
            JS_SVC[Service: jellyseer (NodePort)]; JS_POD[Pod: jellyseer (Deployment)];
            SZ_SVC[Service: sabnzbd (NodePort)]; SZ_POD[Pod: sabnzbd (Deployment)];
        end

        subgraph "Storage"
            PVC[PVC: main-data-pvc];
            HP[hostPath: ./config/*];
        end

        %% Connections
        SO_POD --> QB_SVC; RA_POD --> QB_SVC; LI_POD --> QB_SVC; QR_POD --> QB_SVC;
        BA_POD --> SO_SVC; BA_POD --> RA_SVC;
        PR_POD --> SO_SVC; PR_POD --> RA_SVC; PR_POD --> LI_SVC; PR_POD --> BA_SVC; PR_POD --> QB_SVC;
        DP_POD --> SO_SVC; DP_POD --> RA_SVC;
        JS_POD --> JF_SVC; JS_POD --> SO_SVC; JS_POD --> RA_SVC;

        %% Storage Mounts
        QB_POD -- "/config" --> HP; QB_POD -- "/data/torrents" --> PVC;
        SO_POD -- "/config" --> HP; SO_POD -- "/data" --> PVC;
        RA_POD -- "/config" --> HP; RA_POD -- "/data" --> PVC;
        LI_POD -- "/config" --> HP; LI_POD -- "/data" --> PVC;
        BA_POD -- "/config" --> HP; BA_POD -- "/data" --> PVC;
        PR_POD -- "/config" --> HP;
        QR_POD -- "/config" --> HP; QR_POD -- "/data" --> PVC;
        FS_POD -- "(no config)" --> HP;
        DP_POD -- "(no config)" --> HP;
        JF_POD -- "/config" --> HP; JF_POD -- "/data/media" --> PVC;
        JS_POD -- "/app/config" --> HP; JS_POD -- "/data/media" --> PVC;
        SZ_POD -- "/config" --> HP; SZ_POD -- "/data/usenet" --> PVC;

        %% External Access
        U -- "Port Forward" --> PF;
        PF --> QB_SVC; PF --> SO_SVC; PF --> RA_SVC; PF --> LI_SVC; PF --> BA_SVC; PF --> PR_SVC; PF --> FS_SVC;
        U -- "NodePort" --> NP;
        NP --> JF_SVC; NP --> JS_SVC; NP --> SZ_SVC;
    end

    style PVC fill:#f9f,stroke:#333,stroke-width:2px;
    style HP fill:#ccf,stroke:#333,stroke-width:2px;
```

## Prerequisites

1.  **kubectl:** Configured to connect to your Kubernetes cluster.
2.  **Host Path:** The directory `./config` relative to these manifests must exist on the Kubernetes node(s) where the pods will run.
3.  **Permissions:** The user/group ID used by the containers (PUID=1000, PGID=1000 by default, set in `01-configmap.yaml`) must have read/write permissions to the corresponding subdirectories within `./config` on the host machine.
4.  **Secrets:** The `02-secret.yaml` file is ignored by Git (`.gitignore`). You must create this file manually or have another process (e.g., CI/CD, secrets management tool) create the `torrentbox-secrets` Secret in the `torrentbox` namespace before deploying the applications. It should contain necessary API keys and passwords as defined in the deployment files.

## Deployment (Starting the Stack)

1.  Ensure the `torrentbox` namespace exists or create it:
    ```bash
    kubectl apply -f 00-namespace.yaml
    ```
2.  Ensure the `torrentbox-secrets` Secret exists (create/apply `02-secret.yaml` if necessary).
3.  Apply the rest of the manifests:
    ```bash
    # Apply everything EXCEPT the secrets file (if managed separately)
    kubectl apply -f . --exclude 02-secret.yaml
    # Or apply all if 02-secret.yaml is present and not ignored
    # kubectl apply -f .
    ```
4.  Monitor pod startup:
    ```bash
    kubectl get pods -n torrentbox -w
    ```
    Wait for all pods to show `READY 1/1` and `STATUS Running`.

## Stopping the Stack

To remove all deployed resources (except potentially the namespace and secrets if managed separately):

```bash
kubectl delete -f . --exclude 00-namespace.yaml --exclude 02-secret.yaml
# Or delete all including namespace and secrets (use with caution!)
# kubectl delete -f .
```

## Updating Container Images

To update an application to a new version:

1.  Find the relevant deployment file (e.g., `16-sonarr-deployment.yaml`).
2.  Locate the `image:` line for the application container (e.g., `image: lscr.io/linuxserver/sonarr:latest`).
3.  Change the tag from `:latest` to the specific version you want (e.g., `:4.0.1`). Using specific version tags is recommended over `:latest`.
4.  Apply the updated deployment file:
    ```bash
    kubectl apply -f <modified-deployment-file.yaml>
    ```
    Kubernetes will perform a rolling update to replace the old pod(s) with new ones using the updated image.

## Accessing Services

Applications are exposed in two ways:

1.  **NodePort Services:** For services intended for direct user access from outside the cluster.
    *   **Services:** Jellyfin, Jellyseerr, Sabnzbd
    *   **Find Ports:** Run `kubectl get svc -n torrentbox` and look for the ports listed under the `PORT(S)` column for these services (e.g., `8096:32744/TCP`). The port after the colon (`32744` in this example) is the NodePort.
    *   **Access:** Open `http://<your-node-ip>:<node-port>` in your browser. Replace `<your-node-ip>` with the IP address of your Kubernetes node.

2.  **ClusterIP Services (via Port Forwarding):** For internal services or development access.
    *   **Services:** qBittorrent, Sonarr, Radarr, Lidarr, Bazarr, Prowlarr, Flaresolverr
    *   **How:** Use `kubectl port-forward`. Open a separate terminal for each service you want to access.
    *   **Examples:**
        ```bash
        # Access qBittorrent on http://localhost:8080
        kubectl port-forward service/qbittorrent -n torrentbox 8080:8080

        # Access Sonarr on http://localhost:8989
        kubectl port-forward service/sonarr -n torrentbox 8989:8989

        # Access Radarr on http://localhost:7878
        kubectl port-forward service/radarr -n torrentbox 7878:7878

        # ...and so on for other ClusterIP services
        ```
    *   Keep the `kubectl port-forward` command running while you need access.

## Initial Application Configuration

After the pods are running, you need to configure the applications internally via their Web UIs:

*   **Download Clients:** Configure Sonarr, Radarr, Lidarr, etc., to connect to qBittorrent and/or Sabnzbd using their Kubernetes service names (found in `01-configmap.yaml`, e.g., `http://qbittorrent.torrentbox.svc.cluster.local:8080`).
*   **Paths:** Ensure applications use the correct paths within the shared `/data` volume (e.g., `/data/torrents/complete`, `/data/media/tv`, `/data/media/movies`). Configure Root Folders in Sonarr/Radarr/Lidarr accordingly.
*   **Prowlarr:** Add indexers and configure applications (Sonarr, Radarr, etc.) within Prowlarr using their service names.
*   **Jellyfin/Jellyseerr:** Configure libraries pointing to `/data/media/tv` and `/data/media/movies`. Connect Jellyseerr to Jellyfin and the *arr apps.

Refer to the specific application's documentation for detailed setup instructions.
