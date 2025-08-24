# no-network-on-container-vm-when-using-proxmox-on-ovh
Proxmox LXC NAT network setup (step‑by‑step)

This guide explains how to provide internet access to an LXC container on a Proxmox host that has a single public IP (OVH) by using NAT (MASQUERADE). Replace  with your container ID and adjust the private subnet if desired. Commands are run on the Proxmox host unless otherwise noted.
Overview

    Create a private subnet for LXC containers (example: 10.254.254.0/24).
    Configure the container to use a static IP from that subnet and set its gateway.
    Enable IP forwarding on the host.
    Add iptables NAT and FORWARD rules to allow containers to reach the internet.
    Persist sysctl and iptables rules across reboots.

1. Stop the container

Stop the container before changing its network:
pct stop 
2. Configure the container network

Attach the container to the host bridge (vmbr0) and set a private IP/gateway:
pct set  -net0 name=eth0,bridge=vmbr0,ip=10.254.254.2/24,gw=10.254.254.1

Start the container:
pct start 

Verify inside the container (from host):
pct exec  -- ip a
pct exec  -- ip route
pct exec  -- cat /etc/resolv.conf

If DNS is missing or not working, set a reliable DNS inside the container:
pct exec  -- bash -c 'echo "nameserver 1.1.1.1" > /etc/resolv.conf'
3. Enable IP forwarding on the host

Enable immediately:
sysctl -w net.ipv4.ip_forward=1

Make persistent across reboots:
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-proxmox-ff.conf
sysctl --system
4. Add NAT (MASQUERADE) and FORWARD rules (temporary)

Run these commands to allow traffic from the private subnet to use the host public interface (vmbr0):
iptables -t nat -A POSTROUTING -s 10.254.254.0/24 -o vmbr0 -j MASQUERADE
iptables -A FORWARD -i vmbr0 -o vmbr0 -s 10.254.254.0/24 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -o vmbr0 -i vmbr0 -d 10.254.254.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

Notes:

    These rules take effect immediately but are not persistent across reboots.

5. Add host-side gateway IP for the private subnet (required)

Add the gateway address to the bridge so the host has an L3 presence on the container subnet:
ip addr add 10.254.254.1/24 dev vmbr0

Then verify the host can reach the container:
ping -c 3 10.254.254.2
6. Verify connectivity

From the host:
ping -c 3 10.254.254.2

From the container:
pct exec  -- ping -c 3 8.8.8.8
pct exec  -- curl -sS ifconfig.co   # should show your public IP
7. Make iptables rules persistent

Option A — iptables-persistent (Debian/Proxmox)
apt update && apt install -y iptables-persistent
During install choose to save current rules OR:

iptables-save > /etc/iptables/rules.v4

Option B — systemd restore service
iptables-save > /etc/iptables/rules.v4
Create /etc/systemd/system/iptables-restore.service with:
[Unit]
Description=Restore iptables
After=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/iptables/rules.v4

[Install]
WantedBy=multi-user.target

Enable the service:
systemctl enable --now iptables-restore.service
8. Troubleshooting checklist

    Confirm IP forwarding is enabled: cat /proc/sys/net/ipv4/ip_forward
    Check NAT rules: iptables -t nat -L -n -v
    Check FORWARD chain: iptables -L FORWARD -n -v
    Ensure container IP and route: pct exec -- ip a pct exec -- ip route
    Test host <-> container link: ping -c 3 10.254.254.2
    If host cannot reach the container, ensure the container is attached to vmbr0 and host sees the veth pair: bridge link show ip link show type veth

9. Example commands summary

(paste-and-run, replace )

pct stop 
pct set  -net0 name=eth0,bridge=vmbr0,ip=10.254.254.2/24,gw=10.254.254.1
pct start 
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-proxmox-ff.conf
sysctl --system
iptables -t nat -A POSTROUTING -s 10.254.254.0/24 -o vmbr0 -j MASQUERADE
iptables -A FORWARD -i vmbr0 -o vmbr0 -s 10.254.254.0/24 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -o vmbr0 -i vmbr0 -d 10.254.254.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip addr add 10.254.254.1/24 dev vmbr0
iptables-save > /etc/iptables/rules.v4   # or use iptables-persistent/systemd method