# Host OpenVPN in Kubernetes

以下會說明如何將 OpenVPN 架到 Kubernetes 中

## 開始前準備

- Podman

    > 或是使用 docker 也行

- git
- 文字編輯器

## 套用 SELinux 策略

在裝有 SELinux 的系統中，必須要套用策略，否則容器將無法正常啟動

在套用 SELinux 策略之前，你需要先決定要把 OpenVPN 的設定檔存在哪個資料夾中，本範例使用 `/var/lib/openvpn` 資料夾。

> 如欲關閉 SELinux，請跳至下一大項

### 套用策略

1. 將以下策略儲存為 `openvpnpolicy.te` 檔

    > 若非使用範例的資料夾，則請打開 `openvpnpolicy.te` 檔，並將 `container_var_lib_t` 取代為相對應的策略。

    ```te

    module openvpnpolicy 1.0;

    require {
        type cgroup_t;
        type container_t;
        type container_var_lib_t;
        type iptables_t;
        type unlabeled_t;
        class dir { add_name create getattr ioctl link lock open read remove_name rename reparent rmdir search setattr unlink write };
        class file { create open getattr setattr read write append rename link unlink ioctl lock };
    }

    #============= container_t ==============

    #!!!! This avc is allowed in the current policy
    allow container_t container_var_lib_t:dir { add_name create getattr ioctl link lock open read remove_name rename reparent rmdir search setattr unlink write };

    #!!!! This avc is allowed in the current policy
    allow container_t container_var_lib_t:file { create open getattr setattr read write append rename link unlink ioctl lock };

    #============= iptables_t ==============

    #!!!! This avc is allowed in the current policy
    allow iptables_t cgroup_t:dir { ioctl };

    ```

2. 使用下面指令套用 SELinux 策略

    > 若 `sudo` 無作用, 請切換登入的使用者為 root ，切換到正確的資料夾路徑後再執行下面的指令

    ```console
    # checkmodule -M -m -o openvpnpolicy.mod openvpnpolicy.te
    # semodule_package -o openvpnpolicy.pp -m openvpnpolicy.mod
    # semodule -i openvpnpolicy.pp
    ```

3. 若在進行 config 檔建立時遇到權限問題，請繼續進行 4. ~ 6. 步驟
4. 切換使用者為 `root`

    ```console
    $ su -
    ```

5. 使用指令匯出需要允許的策略

    ```console
    # audit2allow -a -M allowpolicy < /var/log/audit/audit.log
    ```

6. 打開 `allowpolicy.te` 與 `allowregistrypolicy.te` 做比較，看 `container_var_lib_t` 在 `allowpolicy.te` 檔中是什麼名稱，把它取代成該名稱後，進行步驟 2. 重新套用策略。
7. 繼續建置 OpenVPN 的 config 檔案，若仍無法，重複 4. ~ 6. 步驟直到可以正常推送為止

### 關閉 SELinux (不推薦)

若要直接關閉 SELinux，請依據以下步驟進行關閉:

1. 執行指令 `sudo setenforce 0`
2. 修改 `/etc/selinux/config`，修改 `SELINUX=disabled` 後存檔
3. 重啟系統後完成。
    > **極不推薦** 關閉 SELinux，這會讓伺服器暴露於危險之中，且[會讓 Dan Walsh 傷心](https://stopdisablingselinux.com/)。

## 建立映像

依據以下步驟將 OpenVPN 的映像建立起來

> 你也可以直接使用 Docker Hub 上的 kylemanna/docker-openvpn 映像

1. 從 [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn) 將專案複製下來
2. 執行指令以下指令建置映像

    > 若未在這邊將 `<YOUR_REGISTRY_URI>` 修改為你的 registry 位址，你需要自己再次執行 `podman image tag <ORIGINAL_IMAGE_FULL_NAME> <YOUR_REGISTRY_URI>/docker-openvpn` 將映像加上標籤

    ```console
    $ podman build -t <YOUR_REGISTRY_URI>/docker-openvpn -f Dockerfile --no-cache
    ```

3. 執行以下指令將映像推送到映像倉庫中

    > 若映像倉庫 (registry) 沒有 SSL 設定，請在指令後面加上 `--tls-verify=false` 參數。

    ```console
    $ podman push <YOUR_REGISTRY_URI>/docker-openvpn
    ```

## 部署伺服器

1. 在任意位置建立一個資料夾儲存 OpenVPN 設定

    > 範例這邊建立在 `/var/lib/openvpn`

2. 執行以下指令建立 OpenVPN 設定

    > 若權限不足，請使用 `sudo` 執行

    ```console
    $ `podman run -v /var/lib/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://<YOUR_HOST_IP_OR_DOMAIN>`
    ```

3. 執行以下指令初始化憑證

    > 若權限不足，請使用 `sudo` 執行

    ```console
    $ podman run -v /var/lib/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
    ```

4. 將以下內容儲存為 `deployment.yaml`，並將 `<` 與 `>` 包起來的變數更改為正確的值

    ```yaml
    # Before issuing `kubectl apply` command,
    # modify the value contains "<" and ">" to correct value

    # namespace-deploy.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: <NAMESPACE_NAME>

    ---
    # persistent-volume.yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: openvpn-pv
    spec:
      storageClassName: ""
      capacity:
        storage: <SIZE_OF_PV>
      hostPath:
        # You can use the path which is used in README file.
        # Or you'll need to create and apply SELinux policy by yourself.
        path: <PATH_TO_OPENVPN_FOLDER>
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      claimRef:
        namespace: <NAMESPACE_NAME>
        name: openvpn-pvc

    ---
    # persistent-volume-claim.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: openvpn-pvc
    namespace: <NAMESPACE_NAME>
    spec:
      storageClassName: ""
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          # This value must less then or equal to persistent volumes total size
          storage: <SIZE_OF_PVC>

    ---
    # deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: <OPENVPN_APP_NAME>
      namespace: <NAMESPACE_NAME>
      labels:
        app_name: <OPENVPN_APP_NAME>
    spec:
      replicas: 1
      strategy:
        type: Recreate
      selector:
        matchLabels:
          app_name: <OPENVPN_APP_NAME>
      template:
        metadata:
          labels:
            app_name: <OPENVPN_APP_NAME>
        spec:
          volumes:
            - name: openvpn-volume
              persistentVolumeClaim:
                claimName: openvpn-pvc
          containers:
            - name: <OPENVPN_APP_NAME>
              image: <YOUR_REGISTRY_URI>/docker-openvpn:latest
              securityContext:
                privileged: true
                # capabilities:
                #   add: ["NET_ADMIN"]
              ports:
                - containerPort: 1194
                  protocol: UDP
              volumeMounts:
                - name: openvpn-volume
                  mountPath: /etc/openvpn
                  subPath: registry
              imagePullPolicy: Always

    ---
    # service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: <OPENVPN_APP_NAME>
      namespace: <NAMESPACE_NAME>
    spec:
      ports:
        - protocol: UDP
          port: <PROXIED_PORT_NUMBER>
          targetPort: 1911
      selector:
        app_name: <OPENVPN_APP_NAME>
      type: ClusterIP

    ```

5. 執行以下指令將伺服器部署到 Kubernetes 上

    ```console
    $ kubectl apply -f deployment.yaml
    ```

## 設定 Ingress

範例所使用的 Ingress 為 `nginx-ingress`，若非使用 `nginx-ingress` 者，需要自行研究如何開啟 `udp` 的 `1194` 連接埠。

1. 將以下檔案儲存成 `ingress.yaml` 並將以 `<` 與 `>` 包起來的變數更改為正確的值

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: nginx-ingress-controller-udp
      namespace: nginx-ingress
    data:
      '1194': <NAMESPACE_NAME>/<APPLICATION_NAME>:1194
    ```

2. 編輯既有的 `nginx-ingress-service` yaml 內容，在 `spec.ports` 中新增以下宣告後儲存

    ```yaml
    - appProtocol: https
      name: openvpn
      protocol: UDP
      port: 1194
      targetPort: 1194
    ```

3. 完成

## 建立客戶端連線設定

1. 執行以下指令新增客戶端設定

    > 若執行有權限問題，請使用 `sudo` 執行

    > 若不需要設定密碼保護金鑰，請將 `nopass` 放到指令最後方

    ```console
    $ podman run -v /var/lib/openvpn/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full <CLIENT_NAME>
    ```

2. 匯出 ovpn 檔案

    > 若執行有權限問題，請使用 `sudo` 執行

    ```console
    $ podman run -v /var/lib/openvpn/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient <CLIENT_NAME> > <FILENAME>.ovpn
    ```

## 多重連線問題

一個客戶端設定僅可有一個裝置連線，若有第二個裝置連線，前一個裝置的連線會被中斷，若服務架設於私有網路環境，這是比較推薦的設定。

## 參考資料

- [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn)
- [十分鐘 OpenVPN server 架設 – docker 手把手教學](https://koding.work/10-minutes-build-open-vpn-server/)
- [How to Install OpenVPN on Docker {7 Steps}](https://phoenixnap.com/kb/openvpn-docker)
- [卷 - hostPath](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#hostpath)
- [Can a PVC be bound to a specific PV?](https://stackoverflow.com/a/34323691)
- [關於 SELinux Policy](https://medium.com/%E4%B8%80%E5%80%8B%E5%B0%8F%E5%B0%8F%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E9%9A%A8%E6%89%8B%E7%AD%86%E8%A8%98/%E9%97%9C%E6%96%BC-selinux-policy-15466ef928b4)
- [mknod: /dev/net/tun Operation not permited on kubernetes cluster using openvpn/stable](https://stackoverflow.com/a/58488544)
- [mknod: /dev/net/tun: Operation not permitted #498](https://github.com/kylemanna/docker-openvpn/issues/498#issuecomment-708613329)
- [How to run a CRI-O container in priviliged mode ? #2363](https://github.com/cri-o/cri-o/issues/2363#issuecomment-492227812)
- [PKI procedure: using a separate CA system](https://community.openvpn.net/openvpn/wiki/EasyRSA3-OpenVPN-Howto#PKIprocedure:usingaseparateCAsystem)
