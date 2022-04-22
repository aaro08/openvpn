NAME: openvpn
LAST DEPLOYED: Fri Apr 22 09:57:05 2022
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: openvpn/templates/config-openvpn.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openvpn
  labels:
    app: openvpn
    chart: openvpn-0.1.0
    release: openvpn
    heritage: Helm
data:
  setup-certs.sh: |-
    #!/bin/bash
    EASY_RSA_LOC="/etc/openvpn/certs"
    cd $EASY_RSA_LOC
    SERVER_CERT="${EASY_RSA_LOC}/pki/issued/server.crt"
    if [ -e "$SERVER_CERT" ]
    then
      echo "found existing certs - reusing"
    else
      cp -R /usr/share/easy-rsa/* $EASY_RSA_LOC
      ./easyrsa init-pki
      echo "ca\n" | ./easyrsa build-ca nopass
      ./easyrsa build-server-full server nopass
      ./easyrsa gen-dh
    fi
  newClientCert.sh: |-
      #!/bin/bash
      EASY_RSA_LOC="/etc/openvpn/certs"
      cd $EASY_RSA_LOC
      MY_IP_ADDR="$2"
      ./easyrsa build-client-full $1 nopass
      cat >${EASY_RSA_LOC}/pki/$1.ovpn <<EOF
      client
      nobind
      dev tun
      remote ${MY_IP_ADDR} 1194 udp


      redirect-gateway def1

      <key>
      `cat ${EASY_RSA_LOC}/pki/private/$1.key`
      </key>
      <cert>
      `cat ${EASY_RSA_LOC}/pki/issued/$1.crt`
      </cert>
      <ca>
      `cat ${EASY_RSA_LOC}/pki/ca.crt`
      </ca>
      EOF
      cat pki/$1.ovpn

  revokeClientCert.sh: |-
      #!/bin/bash
      EASY_RSA_LOC="/etc/openvpn/certs"
      cd $EASY_RSA_LOC
      ./easyrsa revoke $1
      ./easyrsa gen-crl
      cp ${EASY_RSA_LOC}/pki/crl.pem ${EASY_RSA_LOC}
      chmod 644 ${EASY_RSA_LOC}/crl.pem
  configure.sh: |-
      #!/bin/sh
      cidr2mask() {
         # Number of args to shift, 255..255, first non-255 byte, zeroes
         set -- $(( 5 - ($1 / 8) )) 255 255 255 255 $(( (255 << (8 - ($1 % 8))) & 255 )) 0 0 0
         [ $1 -gt 1 ] && shift "$1" || shift
         echo ${1-0}.${2-0}.${3-0}.${4-0}
      }
      cidr2net() {
          local i ip mask netOctets octets
          ip="${1%/*}"
          mask="${1#*/}"
          octets=$(echo "$ip" | tr '.' '\n')
          for octet in $octets; do
              i=$((i+1))
              if [ $i -le $(( mask / 8)) ]; then
                  netOctets="$netOctets.$octet"
              elif [ $i -eq  $(( mask / 8 +1 )) ]; then
                  netOctets="$netOctets.$((((octet / ((256 / ((2**((mask % 8)))))))) * ((256 / ((2**((mask % 8))))))))"
              else
                  netOctets="$netOctets.0"
              fi
          done
          echo ${netOctets#.}
      }
      /etc/openvpn/setup/setup-certs.sh




      iptables -t nat -A POSTROUTING -s 10.240.0.0/255.255.0.0 -o eth0 -j MASQUERADE
      mkdir -p /dev/net
      if [ ! -c /dev/net/tun ]; then
          mknod /dev/net/tun c 10 200
      fi

      if [ "$DEBUG" == "1" ]; then
          echo ========== ${OVPN_CONFIG} ==========
          cat "${OVPN_CONFIG}"
          echo ====================================
      fi

      intAndIP="$(ip route get 8.8.8.8 | awk '/8.8.8.8/ {print $5 "-" $7}')"
      int="${intAndIP%-*}"
      ip="${intAndIP#*-}"
      cidr="$(ip addr show dev "$int" | awk -vip="$ip" '($2 ~ ip) {print $2}')"

      NETWORK="$(cidr2net $cidr)"
      NETMASK="$(cidr2mask ${cidr#*/})"
      DNS=$(cat /etc/resolv.conf | grep -v '^#' | grep nameserver | awk '{print $2}')
      SEARCH=$(cat /etc/resolv.conf | grep -v '^#' | grep search | awk '{$1=""; print $0}')
      FORMATTED_SEARCH=""
      for DOMAIN in $SEARCH; do
        FORMATTED_SEARCH="${FORMATTED_SEARCH}push \"dhcp-option DOMAIN-SEARCH ${DOMAIN}\"\n"
      done
      cp -f /etc/openvpn/setup/openvpn.conf /etc/openvpn/
      sed 's|OVPN_K8S_SEARCH|'"${FORMATTED_SEARCH}"'|' -i /etc/openvpn/openvpn.conf
      sed 's|OVPN_K8S_DNS|'"${DNS}"'|' -i /etc/openvpn/openvpn.conf
      sed 's|NETWORK|'"${NETWORK}"'|' -i /etc/openvpn/openvpn.conf
      sed 's|NETMASK|'"${NETMASK}"'|' -i /etc/openvpn/openvpn.conf

      # exec openvpn process so it receives lifecycle signals
      exec openvpn --config /etc/openvpn/openvpn.conf
  openvpn.conf: |-
      server 10.240.0.0 255.255.0.0
      verb 3

      key /etc/openvpn/certs/pki/private/server.key
      ca /etc/openvpn/certs/pki/ca.crt
      cert /etc/openvpn/certs/pki/issued/server.crt
      dh /etc/openvpn/certs/pki/dh.pem



      key-direction 0
      keepalive 10 60
      persist-key
      persist-tun

      proto udp
      port  1194
      dev tun0
      status /tmp/openvpn-status.log

      user nobody
      group nogroup




      push "route NETWORK NETMASK"





      OVPN_K8S_SEARCH

      push "dhcp-option DNS OVPN_K8S_DNS"
---
# Source: openvpn/templates/certs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openvpn
  labels:
    app: openvpn
    chart: openvpn-0.1.0
    release: openvpn
    heritage: Helm
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "2M"
---
# Source: openvpn/templates/openvpn-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: openvpn
  labels:
    app: openvpn
    chart: openvpn-0.1.0
    release: openvpn
    heritage: Helm
spec:
  ports:
    - name: openvpn
      port: 1194
      targetPort: 1194
      protocol: UDP
  selector:
    app: openvpn
    release: openvpn
  type: LoadBalancer
---
# Source: openvpn/templates/openvpn-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openvpn
  labels:
    app: openvpn
    chart: openvpn-0.1.0
    release: openvpn
    heritage: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openvpn
      release: openvpn
  template:
    metadata:
      labels:
        app: openvpn
        release: openvpn
      annotations:
        checksum/config: 1a62a42fe7e1a917cdf40e714b1da8873f5b1583661201200b02f8a7a1f3fb30
    spec:
      containers:
      - name: openvpn
        image: "jfelten/openvpn-docker:latest"
        imagePullPolicy: IfNotPresent
        command: ["/etc/openvpn/setup/configure.sh"]
        ports:
        - containerPort: 1194
          name: openvpn
        securityContext:
          capabilities:
            add:
              - NET_ADMIN
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 10
          successThreshold: 5
          exec:
            command:
            - nc
            - -u
            - -z
            - 127.0.0.1
            - "1194"
        resources:
          requests:
            cpu: "300m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "128Mi"
        volumeMounts:
          - mountPath: /etc/openvpn/setup
            name: openvpn
            readOnly: false
          - mountPath: /etc/openvpn/certs
            name: certs
            readOnly: false
      volumes:
      - name: openvpn
        configMap:
          name: openvpn
          defaultMode: 0775
      - name: certs
        persistentVolumeClaim:
          claimName: openvpn

NOTES:
OpenVPN is now starting.

Please be aware that certificate generation is variable and may take some time (minutes).

Check pod status with the command:

  POD_NAME=$(kubectl get pods --namespace "default" -l app=openvpn -o jsonpath='{ .items[0].metadata.name }') && kubectl --namespace "default" logs $POD_NAME --follow

LoadBalancer ingress creation can take some time as well. Check service status with the command:

  kubectl --namespace "default" get svc

Once the external IP is available and all the server certificates are generated create client key .ovpn files by pasting the following into a shell:

  POD_NAME=$(kubectl get pods --namespace "default" -l "app=openvpn,release=openvpn" -o jsonpath='{ .items[0].metadata.name }')
  SERVICE_NAME=$(kubectl get svc --namespace "default" -l "app=openvpn,release=openvpn" -o jsonpath='{ .items[0].metadata.name }')
  SERVICE_IP=$(kubectl get svc --namespace "default" "$SERVICE_NAME" -o go-template='{{ range $k, $v := (index .status.loadBalancer.ingress 0)}}{{ $v }}{{end}}')
  KEY_NAME=kubeVPN
  kubectl --namespace "default" exec -it "$POD_NAME" /etc/openvpn/setup/newClientCert.sh "$KEY_NAME" "$SERVICE_IP"
  kubectl --namespace "default" exec -it "$POD_NAME" cat "/etc/openvpn/certs/pki/$KEY_NAME.ovpn" > "$KEY_NAME.ovpn"

Revoking certificates works just as easy:
  KEY_NAME=<name>
  POD_NAME=$(kubectl get pods -n "default" -l "app=openvpn,release=openvpn" -o jsonpath='{.items[0].metadata.name}')
  kubectl -n "default" exec -it "$POD_NAME" /etc/openvpn/setup/revokeClientCert.sh $KEY_NAME

Copy the resulting $KEY_NAME.ovpn file to your open vpn client (ex: in tunnelblick, just double click on the file).  Do this for each user that needs to connect to the VPN.  Change KEY_NAME for each additional user.