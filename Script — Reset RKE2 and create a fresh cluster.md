# This is the more destructive option. It removes the existing RKE2 installation and cluster data while keeping Ubuntu installed.

# It does not automatically preserve Vault data, Kubernetes secrets, workloads, or local persistent application data.

-----------------------------------------------------

1. Create the script

Run 👇:

nano reset-rke2-complete.sh

-----------------------------------------------------

Paste the following 👇:

#!/usr/bin/env bash
set -Eeuo pipefail

# Combined RKE2 reset and reinstall for Ubuntu.
# Run as a normal user with sudo privileges.
# Uses the current stable RKE2 release on reinstall.

if [[ $EUID -eq 0 ]]; then
  echo "Run as your normal user, not with sudo."
  exit 1
fi

echo "=========================================="
echo " RKE2 CLUSTER RESET + REINSTALL"
echo "=========================================="
echo
echo "WARNING: Existing Kubernetes workloads and"
echo "cluster state will be destroyed."
echo "Back up data and custom RKE2 configuration."
echo
read -rp 'Type RESET-AND-REINSTALL to continue: ' CONFIRM

if [[ "$CONFIRM" != "RESET-AND-REINSTALL" ]]; then
  echo "Cancelled."
  exit 0
fi

if ! sudo -v; then
  echo "Could not obtain sudo privileges."
  exit 1
fi

echo
echo "[1/7] Checking Ubuntu and existing RKE2..."
. /etc/os-release
if [[ "${ID:-}" != "ubuntu" ]]; then
  echo "This script is intended for Ubuntu."
  exit 1
fi

if [[ ! -x /usr/local/bin/rke2-uninstall.sh ]]; then
  echo "RKE2 uninstall script not found."
  echo "Cannot safely identify the existing installation."
  exit 1
fi

echo
echo "[2/7] Uninstalling existing RKE2..."
sudo systemctl stop rke2-server 2>/dev/null || true
sudo systemctl stop rke2-agent 2>/dev/null || true

if [[ -x /usr/local/bin/rke2-killall.sh ]]; then
  sudo /usr/local/bin/rke2-killall.sh
fi

sudo /usr/local/bin/rke2-uninstall.sh

echo
echo "[3/7] Verifying old RKE2 service is removed..."
if sudo systemctl cat rke2-server >/dev/null 2>&1; then
  echo "ERROR: RKE2 server service still exists."
  echo "Stopping here; inspect the installation manually."
  exit 1
fi

echo
echo "[4/7] Installing RKE2..."
sudo apt-get update
sudo apt-get install -y curl

curl -sfL https://get.rke2.io -o /tmp/install-rke2.sh
sudo INSTALL_RKE2_TYPE=server sh /tmp/install-rke2.sh
rm -f /tmp/install-rke2.sh

echo
echo "[5/7] Enabling and starting RKE2..."
sudo systemctl enable --now rke2-server

echo
echo "[6/7] Waiting for Kubernetes API..."
READY=0
for i in $(seq 1 60); do
  if sudo test -f /etc/rancher/rke2/rke2.yaml &&
     sudo /var/lib/rancher/rke2/bin/kubectl \
       --kubeconfig /etc/rancher/rke2/rke2.yaml \
       get nodes >/dev/null 2>&1; then
    READY=1
    break
  fi
  sleep 10
done

if [[ "$READY" -ne 1 ]]; then
  echo "Kubernetes API did not become available within 10 minutes."
  echo "Inspect: sudo journalctl -u rke2-server -n 100 --no-pager"
  exit 1
fi

echo
echo "[7/7] Checking node and system pods..."
sudo /var/lib/rancher/rke2/bin/kubectl \
  --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes -o wide

sudo /var/lib/rancher/rke2/bin/kubectl \
  --kubeconfig /etc/rancher/rke2/rke2.yaml get pods -A

echo
echo "=========================================="
echo " RKE2 REINSTALL FINISHED"
echo "=========================================="
echo "Check that the node STATUS is Ready."
echo "System pods may need extra time to start."
echo "Old apps, Vault data, and custom settings"
echo "must be restored or redeployed separately."

2. Run it 👇:

chmod +x reset-rke2-complete.sh
./reset-rke2-complete.sh

-----------------------------------------------------

3. Verify the new cluster

After the script finishes 👇:

sudo /var/lib/rancher/rke2/bin/kubectl \
  --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes -o wide



Expected result:

NAME                    STATUS   ROLES
your-server-hostname    Ready    control-plane,etcd,master