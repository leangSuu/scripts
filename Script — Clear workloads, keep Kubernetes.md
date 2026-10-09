# This script lets you choose which application namespaces to clear. It protects the core Kubernetes namespaces and offers two modes:

- Clear resources inside selected namespaces while keeping the namespaces.
- Delete selected namespaces completely.
- It does not uninstall RKE2.

=> It does not uninstall RKE2.

-----------------------------------------------------

Step1:

Create the script 👇:

nano clear-workloads.sh

-----------------------------------------------------

Step2:

Paste 👇:

#!/usr/bin/env bash
set -Eeuo pipefail

KUBECTL="sudo /var/lib/rancher/rke2/bin/kubectl"
KUBECONFIG="/etc/rancher/rke2/rke2.yaml"

if [[ ! -f "$KUBECONFIG" ]]; then
  echo "ERROR: RKE2 kubeconfig not found: $KUBECONFIG"
  exit 1
fi

k() {
  sudo KUBECONFIG="$KUBECONFIG" \
    /var/lib/rancher/rke2/bin/kubectl "$@"
}

echo "=== Current namespaces ==="
k get namespaces

echo
echo "Choose cleanup mode:"
echo "1) Delete application resources, keep namespaces"
echo "2) Delete selected application namespaces entirely"
read -rp "Enter 1 or 2: " MODE

case "$MODE" in
  1|2) ;;
  *) echo "Cancelled."; exit 1 ;;
esac

echo
echo "Enter application namespace names separated by spaces."
echo "Example: medusa sso testing"
read -rp "Namespaces: " INPUT

read -r -a NAMESPACES <<< "$INPUT"

if [[ ${#NAMESPACES[@]} -eq 0 ]]; then
  echo "No namespaces selected. Nothing changed."
  exit 1
fi

# Never permit deletion of core/system namespaces.
PROTECTED="default kube-system kube-public kube-node-lease"

for NS in "${NAMESPACES[@]}"; do
  for P in $PROTECTED; do
    if [[ "$NS" == "$P" ]]; then
      echo "ERROR: Protected namespace: $NS"
      exit 1
    fi
  done

  if ! k get namespace "$NS" >/dev/null 2>&1; then
    echo "ERROR: Namespace does not exist: $NS"
    exit 1
  fi
done

echo
echo "Selected namespaces: ${NAMESPACES[*]}"
echo "Mode: $MODE"
echo
echo "WARNING: Application data and secrets may be deleted."
read -rp "Type CLEAR to continue: " CONFIRM

[[ "$CONFIRM" == "CLEAR" ]] || {
  echo "Cancelled."
  exit 0
}

for NS in "${NAMESPACES[@]}"; do
  echo
  echo "=== Processing namespace: $NS ==="

  if [[ "$MODE" == "2" ]]; then
    echo "Deleting namespace $NS..."
    k delete namespace "$NS" --wait=false
    continue
  fi

  # Delete common workload and application resources.
  k delete deployments,statefulsets,daemonsets,jobs,cronjobs \
    --all -n "$NS" --ignore-not-found

  k delete pods,services,replicasets \
    --all -n "$NS" --ignore-not-found

  k delete ingress,configmaps \
    --all -n "$NS" --ignore-not-found

  echo "Namespace kept: $NS"
done

echo
echo "=== Remaining workloads ==="
k get pods -A
echo "Cleanup command completed."

-----------------------------------------------------

Step3:

Run it 👇:

chmod +x clear-workloads.sh
./clear-workloads.sh