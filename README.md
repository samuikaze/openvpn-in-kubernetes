# Host OpenVPN in Kubernetes

以下會說明如何將 OpenVPN 架到 Kubernetes 中

> 請注意，OpenVPN 的容器將會是 `特權 (priviledged)` 容器

## 開始前準備

- Podman

    > 或是使用 docker 也行

- git
- 文字編輯器

## 套用 SELinux 策略

在裝有 SELinux 的系統中，必須要套用策略，否則容器將無法正常啟動

在套用 SELinux 策略之前，你需要先決定要把 OpenVPN 的設定檔存在哪個資料夾中，本範例使用 `/var/lib/openvpn` 資料夾。

> `/var/lib/openvpn` 是基礎資料夾，實際上容器會在此資料夾下再建立一個 `data` 資料夾，所有設定檔案都會在放在這邊

> 使用指令建立資料夾時請建立到 `/var/lib/openvpn` 這層就好，裡面那層資料夾**必須**由容器自己建立，否則權限會不正確。

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
7. 繼續建置 OpenVPN 的 config 檔案，若仍無法，重複 4. ~ 6. 步驟直到可以正常建置為止

### 關閉 SELinux (不推薦)

若要直接關閉 SELinux，請依據以下步驟進行關閉:

1. 執行指令 `sudo setenforce 0`
2. 修改 `/etc/selinux/config`，修改 `SELINUX=disabled` 後存檔
3. 重啟系統後完成。
    > **極不推薦**關閉 SELinux，這會讓伺服器暴露於危險之中，且[會讓 Dan Walsh 傷心](https://stopdisablingselinux.com/)。

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

2. 先將以下內容儲存為 `deployment.yaml`，並將 `<` 與 `>` 包起來的變數更改為正確的值

    > 部署上去後容器必定起不來，這是正常現象，我們只是先讓容器把 `/var/lib/openvpn/data` 的資料夾建起來而已。

    > 若你的 Service 曾經同時暴露 1194 為 TCP 和 UDP 連接埠，請先移除該 Service 宣告後再重新部署一次，否則會無法部署

    > TCP 與 UDP 差別在於封包掉包問題，OpenVPN 雖然較推薦使用 UDP 協定，但這邊還是比較建議使用 TCP 協定，可以避免很多不必要的問題

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
        path: /var/lib/openvpn
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
          initContainers:
            - name: enable-ip-forward
              image: busybox:latest
              securityContext:
                privileged: true
              command:
              - /bin/sh
              args:
              - -c
              - sysctl -w net.ipv4.ip_forward=1
          containers:
            - name: <OPENVPN_APP_NAME>
              image: <YOUR_REGISTRY_URI>/docker-openvpn:latest
              securityContext:
                privileged: true
                capabilities:
                  add: ["NET_ADMIN"]
              ports:
                - name: ovpn-port
                  containerPort: 1194
                  # Or `UDP` if you want
                  protocol: TCP
              startupProbe:
                tcpSocket:
                  port: ovpn-port
                initialDelaySeconds: 6
                periodSeconds: 10
              livenessProbe:
                tcpSocket:
                  port: ovpn-port
                initialDelaySeconds: 5
                periodSeconds: 60
              volumeMounts:
                - name: openvpn-volume
                  mountPath: /etc/openvpn
                  subPath: data
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
        # To use UDP protocol only, comment out TCP block and uncomment UDP block
        - protocol: TCP
          port: <PROXIED_PORT_NUMBER>
          targetPort: 1911
        # - protocol: UDP
        #   port: <PROXIED_PORT_NUMBER>
        #   targetPort: 1911
      selector:
        app_name: <OPENVPN_APP_NAME>
      type: ClusterIP

    ```

3. 執行以下指令將伺服器部署到 Kubernetes 上

    ```console
    $ kubectl apply -f deployment.yaml
    ```

4. 看到容器啟動失敗後可以先將 Deployment 中的 `replicas` 數量降為 0 個

5. 執行以下指令建立 OpenVPN 設定

    > 若權限不足，請使用 `sudo` 執行

    > 這邊會建立伺服器憑證的金鑰密碼，請**務必**保存好該密碼，否則後續的連線設定檔將會無法建置

    > 憑證過期後可以透過這個指令簽發新的憑證，舊的憑證會先被刪除後，新的憑證才會被簽發

    > 若不是使用 `/var/lib/openvpn` 路徑當作 OpenVPN 放置設定的資料夾，請將此路徑取代為你自己的資料夾路徑

    - TCP 使用此指令:

        ```console
        $ `podman run -v /var/lib/openvpn/data:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u tcp://<YOUR_HOST_IP_OR_DOMAIN>`
        ```

    - UDP 使用此指令:

        ```console
        $ `podman run -v /var/lib/openvpn/data:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://<YOUR_HOST_IP_OR_DOMAIN>`
        ```

6. 使用 TCP 作為連線協定者，這邊還需要打開 `/var/lib/openvpn/data/openvpn.conf` 檔案，找到 `proto tcp`，將其修改為 `proto tcp-server`

    > 使用 UDP 作為連線協定者可以跳過此步驟

    > 若不是使用 `/var/lib/openvpn` 路徑當作 OpenVPN 放置設定的資料夾，請將此路徑取代為你自己的資料夾路徑

7. 執行以下指令初始化憑證

    > 若權限不足，請使用 `sudo` 執行

    > 若不是使用 `/var/lib/openvpn` 路徑當作 OpenVPN 放置設定的資料夾，請將此路徑取代為你自己的資料夾路徑

    ```console
    $ podman run -v /var/lib/openvpn/data:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
    ```

8. 若 4. 有將 `replicas` 降回 0 個，這邊請將 `replicas` 恢復成 1 個，若 4. 未設定 `replicas` 數量，這邊請直接透過 `Deployment` 的方式重起一個 Pod，這邊的 Pod 就會正常啟動了

## 設定 Ingress

範例所使用的 Ingress 為 `nginx-ingress`，若非使用 `nginx-ingress` 者，需要自行研究如何開啟 `tcp` 或 `udp` 的 `1194` 連接埠。

> 這邊推薦使用 `TCP` 協定，較不易遺失封包，`UDP` 設定方式與 TCP 類似，可以參考[官方文件說明](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)

1. 將以下檔案儲存成 `ingress.yaml` 並將以 `<` 與 `>` 包起來的變數更改為正確的值

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      # If you want to expose port on udp protocol, change this name to udp
      name: nginx-ingress-controller-tcp
      namespace: nginx-ingress
    data:
      '1194': <NAMESPACE_NAME>/<APPLICATION_NAME>:1194
    ```

2. 編輯既有的 `nginx-ingress-service` yaml 內容，在 `spec.ports` 中新增以下宣告後儲存

    ```yaml
    - appProtocol: https
      name: openvpn
      # Or `UDP` if you want
      protocol: TCP
      port: 1194
      targetPort: 1194
    ```

3. 針對 `nginx-ingress` 的 `Deployment` 中的啟動參數 (args) 加入 `--tcp-services-configmap=ingress-nginx/nginx-ingress-controller-tcp`，如下範例:

    ```yaml
    args:
        - /nginx-ingress-controller
        - --tcp-services-configmap=ingress-nginx/nginx-ingress-controller-tcp
    ```

4. 完成

## 建立客戶端連線設定

1. 執行以下指令新增客戶端設定

    > 若執行有權限問題，請使用 `sudo` 執行

    > 若不需要設定密碼保護金鑰，請將 `nopass` 放到指令最後方，私有 VPN 建議還是透過密碼保護金鑰比較安全

    > 過程中第一次會先設定**連線設定檔的密碼**，第二次之後才會詢問**伺服器憑證的金鑰密碼**，每次密碼都要輸入兩次避免打錯，設定時請務必不要搞錯

    ```console
    $ podman run -v /var/lib/openvpn/data:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full <CLIENT_NAME>
    ```

2. 匯出 ovpn 檔案

    > 若執行有權限問題，請使用 `sudo` 執行

    ```console
    $ podman run -v /var/lib/openvpn/data:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient <CLIENT_NAME> > <FILENAME>.ovpn
    ```

3. 若是使用 TCP 協定者，請打開匯出的 ovpn 檔案，找到 `remote <IP_OR_DOMAIN> 1194 tcp` 將 `tcp` 取代為 `tcp-client`
4. 若暴露到外網的連接埠不是 `1194` 者，請打開匯出的 ovpn 檔案，找到 `remote <IP_OR_DOMAIN> 1194` 這行，把 `1194` 取代為暴露到外網的連接埠號碼

## 路由轉發設定

路由轉發可由伺服器端與客戶端兩種方式設定，可以選擇其中一種方式設定即可

### 設定路由轉發

1. 不論使用哪種設定，請都先打開 ovpn 檔案，滾到最下方找到 `redirect-gateway def1`，把它刪除或以 `#` 註解掉。
2. 若有聯外網需求，請務必打開 ovpn 檔案，增加 `pull-filter ignore "block-outside-dns"` 設定，外網才不會頻頻找不到網域。
3. 完成上述前置作業後，再進行下面其中一種設定:

- 伺服器端設定

    1. 打開 `/var/lib/openvpn/data/openvpn.conf`

        > 若不是使用 `/var/lib/openvpn` 路徑當作 OpenVPN 放置設定的資料夾，請將此路徑取代為你自己的資料夾路徑

    2. 在檔案最下方的 `### Push Configurations Below` 區塊最後面加上要轉發的路由，以下是指定 192.168.127.0/24 都要經過 VPN 閘道

        > 請注意，不要將 `route-nopull` 推送到客戶端，否則這邊的設定都會無效

        ```conf
        push "route 192.168.127.0 255.255.255.0 vpn_gateway"
        ```

    3. 完成。

- 客戶端設定

    1. 打開 ovpn 檔案
    2. 加上 `route <FORWARDING_IP> <SUBNET>`，以下是指定 192.168.127.0/24 都要經過 VPN 閘道

        > 若不希望從伺服器收到路由轉發設定，請在整個 route 最後方加上 `route-nopull`

        > 承上，千萬不要把 `route-nopull` 加在任何一條 route 設定之前，否則 OpenVPN 會在連線時報錯

        ```conf
        route 192.168.127.0 255.255.255.0 vpn_gateway
        ```

    3. 存檔完成

### 測試

- 使用 Windows 進行測試:

    1. 打開 Powershell
    2. 執行 `tracert <EXTERNAL_IP_OR_DOMAIN>`，其中 `<EXTERNAL_IP_OR_DOMAIN>` 為不轉發的路由，檢查每個跳點有沒有經過 VPN 的伺服器，沒有經過才算測試成功
    3. 執行 `tracert <INTERNAL_IP_OR_DOMAIN>`，其中 `<INTERNAL_IP_OR_DOMAIN>` 為要轉發的路由，檢查每個跳點有沒有經過 VPN 的伺服器，有經過才算測試成功

- 使用 Linux 進行測試

    1. 打開終端機
    2. 執行 `traceroute <EXTERNAL_IP_OR_DOMAIN>`，其中 `<EXTERNAL_IP_OR_DOMAIN>` 為不轉發的路由，檢查每個跳點有沒有經過 VPN 的伺服器，沒有經過才算測試成功
    3. 執行 `traceroute <INTERNAL_IP_OR_DOMAIN>`，其中 `<INTERNAL_IP_OR_DOMAIN>` 為要轉發的路由，檢查每個跳點有沒有經過 VPN 的伺服器，有經過才算測試成功

- 使用 Android 進行測試

    1. 先安裝 `Termux` 應用程式
    2. 打開 `Termux` 後執行 `pkg install traceroute` 安裝 `traceroute` 套件
    3. 執行 `traceroute <EXTERNAL_IP_OR_DOMAIN>`，其中 `<EXTERNAL_IP_OR_DOMAIN>` 為不轉發的路由，檢查每個跳點有沒有經過 VPN 的伺服器，沒有經過才算測試成功
    4. 執行 `traceroute <INTERNAL_IP_OR_DOMAIN>`，其中 `<INTERNAL_IP_OR_DOMAIN>` 為要轉發的路由，檢查每個跳點有沒有經過 VPN 的伺服器，有經過才算測試成功

- Mac 使用者

    由於手邊沒有相關裝置，若要進行測試，方法同上面都是檢查不轉發的路由是不是直接走外網到目的地，要轉發的路由是不是會經過 VPN 的閘道才到目的地。

## 壓縮問題

客戶端與伺服器端的壓縮設定必須相同，否則連線會失敗，又官方文件指出**壓縮會有資料加密的問題**，因此建議不要打開壓縮設定，若需打開，客戶端與伺服器端的設定請務必相同。

如要在客戶端關閉壓縮，請打開 `ovpn` 檔案，並於上面區塊加上 `comp-lzo no`，類似於下面這個樣子

```conf
client
nobind
dev tun
comp-lzo no
remote-cert-tls server
```

## 多重連線問題

一個客戶端設定僅可有一個裝置連線，若有第二個裝置連線，前一個裝置的連線會被中斷，若服務架設於私有網路環境，這是比較推薦的設定。

## 更新 OpenVPN 版本

只要重新建置映像並重新部署到 Kubernetes 上，OpenVPN 版本就會被更新。

## 不使用 Priviledged 容器

可以參考[這篇解答](https://stackoverflow.com/a/70101383)的做法，在本機節點上先新增 `tun` 的網卡後，把這張網卡透過 volumeMount 的方式綁到容器裡，這樣一來就不需要啟用容器的 `priviledged` 模式了。

## 參考資料

- [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn)
- [十分鐘 OpenVPN server 架設 – docker 手把手教學](https://koding.work/10-minutes-build-open-vpn-server/)
- [OpenVPN Client in Kubernetes Pod](https://stackoverflow.com/a/70101383)
- [How to Install OpenVPN on Docker {7 Steps}](https://phoenixnap.com/kb/openvpn-docker)
- [卷 - hostPath](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#hostpath)
- [Can a PVC be bound to a specific PV?](https://stackoverflow.com/a/34323691)
- [關於 SELinux Policy](https://medium.com/%E4%B8%80%E5%80%8B%E5%B0%8F%E5%B0%8F%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E9%9A%A8%E6%89%8B%E7%AD%86%E8%A8%98/%E9%97%9C%E6%96%BC-selinux-policy-15466ef928b4)
- [mknod: /dev/net/tun Operation not permited on kubernetes cluster using openvpn/stable](https://stackoverflow.com/a/58488544)
- [mknod: /dev/net/tun: Operation not permitted #498](https://github.com/kylemanna/docker-openvpn/issues/498#issuecomment-708613329)
- [How to run a CRI-O container in priviliged mode ? #2363](https://github.com/cri-o/cri-o/issues/2363#issuecomment-492227812)
- [PKI procedure: using a separate CA system](https://community.openvpn.net/openvpn/wiki/EasyRSA3-OpenVPN-Howto#PKIprocedure:usingaseparateCAsystem)
- [Exposing TCP and UDP services](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/)
- [Comments in OpenVPN client config files?](https://serverfault.com/a/679237)
- [Overriding a pushed "route" in the client's config throws an error](https://openvpn.net/faq/overriding-a-pushed-route-in-the-clients-config-throws-an-error/)
- [OpenVpn doesn't connect via TCP](https://superuser.com/a/1744267)
- [OpenVPN - Getting started How-To](https://community.openvpn.net/openvpn/wiki/GettingStartedwithOVPN)
- [Configure Linux as a Router (IP Forwarding)](https://www.linode.com/docs/guides/linux-router-and-ip-forwarding/)
- [OpenVPN: 設定路由讓特定 IP Address 通過 VPN](https://blog.sakamoto.cat/she-ding-openvpn-lu-you-jiang-qi/)
- [Setting up routing](https://openvpn.net/community-resources/setting-up-routing/)
- [詳解-OpenVPN中的routing](https://cheaster.blogspot.com/2009/11/openvpnrouting.html)
- [在 Kubernetes 叢集中使用 sysctl](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/sysctl-cluster/)
- [Container has net.ipv4.ip_forward enabled but when run it is disabled](https://discuss.kubernetes.io/t/container-has-net-ipv4-ip-forward-enabled-but-when-run-it-is-disabled/21850)
- [What Is the TUN Interface Used For?](https://www.baeldung.com/linux/tun-interface-purpose)
- [How to disable comp-lzo and understand the logs?](https://forums.openvpn.net/viewtopic.php?t=33419)
