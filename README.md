# NETCONF Call-Home and RU Provisioning

This directory contains RESTCONF payloads and commands for allowing an O-RU to
call home to ONAP SDNC, checking its NETCONF session, and provisioning the
O-RAN interfaces, processing element, synchronization, and U-Plane.

The commands below assume:

- SDNC RESTCONF: `http://10.1.101.54:30267`
- SDNC namespace: `onap`
- SDNC pod: `onap-sdnc-0`
- RESTCONF credentials are supplied through `SDNC_AUTH`
- `curl` and `jq` are available on the machine running the commands

Do not commit `SDNC_AUTH` or an RU password to this repository. Export them in
the shell or inject them from a secret store.

## 1. Set the session variables

```bash
export SDNC_BASE="http://10.1.101.54:30267"
export SDNC_AUTH='admin:<password>'
export RU_NODE="RU0-169.254.0.2"
export RU_HOST="10.1.150.155"
export RU_KEY='<RU SSH host public key>'

export SDNC_DATA="${SDNC_BASE}/rests/data"
export CALLHOME_URL="${SDNC_DATA}/odl-netconf-callhome-server:netconf-callhome-server"
export TOPOLOGY_URL="${SDNC_DATA}/network-topology:network-topology/topology=topology-netconf"
```

`RU_NODE` is the SDNC device `unique-id`. `RU_HOST` is the source address
visible in the SDNC pod's established TCP connection list; replace it with the
address of the RU being checked.

## 2. Add / whitelist a single RU

The `RU_KEY` value must be the RU SSH host key advertised by that RU. The
username and password below are example RU credentials; replace them with the
credentials configured on the RU.

```bash
curl -fsS -u "$SDNC_AUTH" \
  -X PUT \
  -H 'Content-Type: application/yang-data+json' \
  -d '{
    "odl-netconf-callhome-server:device": [{
      "unique-id": "'"$RU_NODE"'",
      "ssh-client-params": {
        "credentials": {
          "username": "oranuser",
          "passwords": ["<RU password>"]
        },
        "host-key": "'"$RU_KEY"'"
      }
    }]
  }' \
  "$CALLHOME_URL/allowed-devices/device=$RU_NODE"
```

An HTTP `204 No Content` indicates that SDNC accepted the configuration.

## 3. View configured allowed devices

```bash
curl -fsS -u "$SDNC_AUTH" \
  "$CALLHOME_URL/allowed-devices?content=config" | jq .
```

To list only RU entries:

```bash
curl -fsS -u "$SDNC_AUTH" \
  "$CALLHOME_URL?content=config" |
  jq '."odl-netconf-callhome-server:netconf-callhome-server"
      ."allowed-devices".device[]
      | select(."unique-id" | startswith("RU"))'
```

## 4. Check NETCONF connection status

```bash
curl -fsS -u "$SDNC_AUTH" \
  "$TOPOLOGY_URL" |
  jq '."network-topology:topology"[0].node[]
      | {
          node_id: ."node-id",
          status: ."netconf-node-topology:connection-status",
          host: ."netconf-node-topology:host",
          port: ."netconf-node-topology:port"
        }'
```

Check the RU's call-home source address in the SDNC pod logs:

```bash
kubectl logs -n onap onap-sdnc-0 -c sdnc | grep -F -- "$RU_HOST"
```

Check established call-home sessions on port `4334`:

```bash
kubectl exec -n onap onap-sdnc-0 -c sdnc -- \
  ss -tnp | grep -E '(:4334|:.* 4334).*ESTAB'
```

If `ss` is unavailable in the container, use:

```bash
kubectl exec -n onap onap-sdnc-0 -c sdnc -- \
  netstat -tnp | grep ':4334 .*ESTABLISHED'
```

## 5. Read the RU U-Plane configuration

```bash
curl -fsS -u "$SDNC_AUTH" \
  "$TOPOLOGY_URL/node=$RU_NODE/yang-ext:mount/o-ran-uplane-conf:user-plane-configuration" |
  jq .
```

Read one TX carrier:

```bash
curl -fsS -u "$SDNC_AUTH" \
  -H 'Accept: application/yang-data+json' \
  "$TOPOLOGY_URL/node=$RU_NODE/yang-ext:mount/o-ran-uplane-conf:user-plane-configuration/tx-array-carriers=tx-carr0" |
  jq '."o-ran-uplane-conf:tx-array-carriers"[0]
      | {name, gain, active, type, center: ."center-of-channel-bandwidth"}'
```

## 6. Provision RU3

Set the target node and mount URL:

```bash
export RU_NODE="RU3-169.254.0.14"
export MOUNT_URL="${TOPOLOGY_URL}/node=${RU_NODE}/yang-ext:mount"
```

Apply the payloads in this order:

```bash
# 1. Physical and VLAN interfaces
curl -i -sS -u "$SDNC_AUTH" \
  -X PUT \
  -H 'Content-Type: application/yang-data+xml' \
  --data-binary @NETCONF-Callhome/ru3-interfaces.xml \
  "$MOUNT_URL/ietf-interfaces:interfaces"

# 2. Processing element and Ethernet transport flow
curl -i -sS -u "$SDNC_AUTH" \
  -X PUT \
  -H 'Content-Type: application/yang-data+xml' \
  --data-binary @NETCONF-Callhome/ru3-processing-elements.xml \
  "$MOUNT_URL/o-ran-processing-element:processing-elements"

# 3. PTP synchronization
curl -i -sS -u "$SDNC_AUTH" \
  -X PUT \
  -H 'Content-Type: application/yang-data+xml' \
  --data-binary @NETCONF-Callhome/ru3-sync.xml \
  "$MOUNT_URL/o-ran-sync:sync"

# 4. U-Plane carriers, endpoints, eAxC IDs, and links
curl -i -sS -u "$SDNC_AUTH" \
  -X PUT \
  -H 'Content-Type: application/yang-data+xml' \
  --data-binary @NETCONF-Callhome/ru3-uplane.xml \
  "$MOUNT_URL/o-ran-uplane-conf:user-plane-configuration"
```

Expected success is `HTTP/1.1 204 No Content`. A `500` containing
`TransactionCommitFailedException` means the RU callback rejected the
configuration during commit; it is not an XML transport or authentication
failure. Check the RU's advertised capabilities and the SDNC/O-RU logs before
retrying.

The U-Plane payload currently links the low-level TX and RX links to
`tx-carr0`/`rx-carr0`. Do not mark another carrier `ACTIVE` unless its required
links and endpoints are also provisioned and supported by the RU.

## 7. Verify the RU3 configuration

```bash
# Active TX carrier
curl -fsS -u "$SDNC_AUTH" \
  "$MOUNT_URL/o-ran-uplane-conf:user-plane-configuration/tx-array-carriers=tx-carr0" |
  jq .

# PTP synchronization
curl -fsS -u "$SDNC_AUTH" \
  "$MOUNT_URL/o-ran-sync:sync" | jq .

# Processing-element binding
curl -fsS -u "$SDNC_AUTH" \
  "$MOUNT_URL/o-ran-processing-element:processing-elements" | jq .
```

## 8. Change TX gain without replacing the U-Plane

The Metanoia Jura payloads in this directory use the documented fixed-point
scale: `240000.0` represents `24.0 dB`, `220000.0` represents `22.0 dB`, and
`200000.0` represents `20.0 dB`.

```bash
curl -i -sS -u "$SDNC_AUTH" \
  -X PATCH \
  -H 'Content-Type: application/yang-data+json' \
  -d '{
    "o-ran-uplane-conf:user-plane-configuration": {
      "tx-array-carriers": [
        {"name": "tx-carr0", "gain": 200000.0},
        {"name": "tx-carr1", "gain": 200000.0}
      ]
    }
  }' \
  "$MOUNT_URL/o-ran-uplane-conf:user-plane-configuration"
```

Verify the resulting values using the carrier query in section 7. Use a
targeted `PATCH` for a gain-only change instead of replaying the complete
U-Plane payload.
