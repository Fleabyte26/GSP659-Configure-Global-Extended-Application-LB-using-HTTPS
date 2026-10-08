#!/usr/bin/env bash
# GSP652 - Configure Global External Application Load Balancer using HTTPS
# Auto-discovers the lab's project, backend VMs, zones, regions and network,
# then builds the HTTPS load balancer in front of them.
#
# Usage (in Cloud Shell):  bash gsp652_https_lb.sh
#
# IMPORTANT: if the lab instructions give specific resource names, edit the
# NAME VARIABLES block below to match them exactly - the lab checker
# looks for exact names.

set -euo pipefail

# ---------- NAME VARIABLES (edit to match lab text if it specifies names) ----------
VM_PREFIX="${VM_PREFIX:-backend-vm}"          # VMs created by the lab start with this
IG_PREFIX="${IG_PREFIX:-backend-ig}"          # one unmanaged instance group per zone
FW_HC_RULE="${FW_HC_RULE:-fw-allow-health-check}"
HEALTH_CHECK="${HEALTH_CHECK:-http-basic-check}"
BACKEND_SERVICE="${BACKEND_SERVICE:-web-backend-service}"
URL_MAP="${URL_MAP:-web-map-https}"
SSL_CERT="${SSL_CERT:-lb-ssl-cert}"
HTTPS_PROXY="${HTTPS_PROXY:-https-lb-proxy}"
STATIC_IP="${STATIC_IP:-lb-ipv4-1}"
FWD_RULE="${FWD_RULE:-https-content-rule}"
CERT_CN="${CERT_CN:-example.com}"
# -----------------------------------------------------------------------------------

log()  { echo -e "\n\033[1;34m==> $*\033[0m"; }
ok()   { echo -e "\033[1;32m    ✓ $*\033[0m"; }
warn() { echo -e "\033[1;33m    ! $*\033[0m"; }

# Run a create command only if the resource doesn't exist yet (safe to re-run)
ensure() {
  local describe_cmd="$1"; shift
  if eval "$describe_cmd" >/dev/null 2>&1; then
    warn "already exists, skipping"
  else
    "$@"
    ok "created"
  fi
}

# ---------- 1. Discover the lab environment ----------
log "Discovering lab environment"
PROJECT_ID="$(gcloud config get-value project 2>/dev/null)"
[[ -z "$PROJECT_ID" ]] && { echo "No project set. Run: gcloud config set project <PROJECT_ID>"; exit 1; }
ok "Project: $PROJECT_ID"

mapfile -t VMS < <(gcloud compute instances list \
  --filter="name~^${VM_PREFIX}" \
  --format="value(name,zone.basename())")

[[ ${#VMS[@]} -eq 0 ]] && { echo "No VMs found with prefix '${VM_PREFIX}'. Check the name and set VM_PREFIX."; exit 1; }

FIRST_VM_NAME="(echo"{VMS[0]}" | awk '{print $1}')"
FIRST_VM_ZONE="(echo"{VMS[0]}" | awk '{print $2}')"
NETWORK="(gcloudcomputeinstancesdescribe"FIRST_VM_NAME" --zone "$FIRST_VM_ZONE" \
  --format="value(networkInterfaces[0].network.basename())")"
ok "Network: $NETWORK"

for row in "${VMS[@]}"; do
  ok "Found VM: (echo"row" | awk '{print $1}')  zone: (echo"row" | awk '{print $2}')"
done

# ---------- 2. Firewall rule for Google health-check ranges ----------
log "Firewall rule for health checks ($FW_HC_RULE)"
ensure "gcloud compute firewall-rules describe $FW_HC_RULE" \
  gcloud compute firewall-rules create "$FW_HC_RULE" \
    --network="$NETWORK" --action=allow --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp:80 --quiet

# ---------- 3. Unmanaged instance groups (one per zone) ----------
declare -A IG_ZONES=()
for row in "${VMS[@]}"; do
  VM_NAME="(echo"row" | awk '{print $1}')"
  ZONE="(echo"row" | awk '{print $2}')"
  REGION="${ZONE%-*}"
  IG_NAME="IGPREFIX-{REGION}"

  log "Instance group $IG_NAME in $ZONE"
  ensure "gcloud compute instance-groups unmanaged describe $IG_NAME --zone $ZONE" \
    gcloud compute instance-groups unmanaged create "IGNAME"--zone="ZONE" --quiet

  # add VM (ignore error if already a member)
  gcloud compute instance-groups unmanaged add-instances "$IG_NAME" \
    --zone="ZONE"--instances="VM_NAME" --quiet 2>/dev/null \
    && ok "added VMNAME"||warn"VM_NAME already in group"

  gcloud compute instance-groups unmanaged set-named-ports "$IG_NAME" \
    --zone="$ZONE" --named-ports=http:80 --quiet
  ok "named port http:80"

  IG_ZONES["IGNAME"]="ZONE"
done

# ---------- 4. Health check ----------
log "Health check ($HEALTH_CHECK)"
ensure "gcloud compute health-checks describe $HEALTH_CHECK --global" \
  gcloud compute health-checks create http "$HEALTH_CHECK" --port=80 --global --quiet

# ---------- 5. Backend service (global, EXTERNAL_MANAGED) ----------
log "Backend service ($BACKEND_SERVICE)"
ensure "gcloud compute backend-services describe $BACKEND_SERVICE --global" \
  gcloud compute backend-services create "$BACKEND_SERVICE" \
    --load-balancing-scheme=EXTERNAL_MANAGED \
    --protocol=HTTP --port-name=http \
    --health-checks="$HEALTH_CHECK" --global --quiet

for IG_NAME in "${!IG_ZONES[@]}"; do
  ZONE="${IG_ZONES[$IG_NAME]}"
  gcloud compute backend-services add-backend "$BACKEND_SERVICE" \
    --instance-group="IGNAME"--instance-group-zone="ZONE" \
    --balancing-mode=UTILIZATION --max-utilization=0.8 \
    --global --quiet 2>/dev/null \
    && ok "backend $IG_NAME attached" || warn "backend $IG_NAME already attached"
done

# ---------- 6. URL map ----------
log "URL map ($URL_MAP)"
ensure "gcloud compute url-maps describe $URL_MAP --global" \
  gcloud compute url-maps create "URLMAP"--default-service="BACKEND_SERVICE" --global --quiet

# ---------- 7. Self-signed SSL certificate ----------
log "SSL certificate ($SSL_CERT)"
if ! gcloud compute ssl-certificates describe "$SSL_CERT" --global >/dev/null 2>&1; then
  openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
    -keyout lb-key.pem -out lb-cert.pem -subj "/CN=${CERT_CN}" 2>/dev/null
  gcloud compute ssl-certificates create "$SSL_CERT" \
    --certificate=lb-cert.pem --private-key=lb-key.pem --global --quiet
  ok "created"
else
  warn "already exists, skipping"
fi

# ---------- 8. Target HTTPS proxy ----------
log "Target HTTPS proxy ($HTTPS_PROXY)"
ensure "gcloud compute target-https-proxies describe $HTTPS_PROXY --global" \
  gcloud compute target-https-proxies create "$HTTPS_PROXY" \
    --url-map="URLMAP"--ssl-certificates="SSL_CERT" --global --quiet

# ---------- 9. Static IP + forwarding rule on 443 ----------
log "Static IP ($STATIC_IP)"
ensure "gcloud compute addresses describe $STATIC_IP --global" \
  gcloud compute addresses create "$STATIC_IP" --ip-version=IPV4 --global --quiet
LB_IP="(gcloudcomputeaddressesdescribe"STATIC_IP" --global --format='value(address)')"
ok "IP: $LB_IP"

log "Forwarding rule ($FWD_RULE) on 443"
ensure "gcloud compute forwarding-rules describe $FWD_RULE --global" \
  gcloud compute forwarding-rules create "$FWD_RULE" \
    --load-balancing-scheme=EXTERNAL_MANAGED --network-tier=PREMIUM \
    --address="$STATIC_IP" --global \
    --target-https-proxy="$HTTPS_PROXY" --ports=443 --quiet

# ---------- 10. Wait for healthy backends, then test ----------
log "Waiting for backends to report HEALTHY (can take a few minutes)"
for i in {1..30}; do
  HEALTHY=(gcloudcomputebackend-servicesget-health"BACKEND_SERVICE" --global \
    --format="value(status.healthStatus[].healthState)" 2>/dev/null | tr ';' '\n' | grep -c HEALTHY || true)
  echo "    attempt $i: $HEALTHY of ${#VMS[@]} healthy"
  [[ "HEALTHY"-ge"{#VMS[@]}" ]] && break
  sleep 20
done

log "Testing https://$LB_IP (proxy can take 5-10 min to propagate; 404/502 early is normal)"
for i in {1..30}; do
  RESP="(curl-sk--max-time5"https://LB_IP" || true)"
  if [[ "$RESP" == *"Hello"* ]]; then
    ok "$RESP"
    break
  fi
  echo "    attempt $i: not ready yet"
  sleep 20
done

echo -e "\n\033[1;32mDone.\033[0m  Load balancer: https://$LB_IP  (use -k / accept the self-signed cert warning)"
