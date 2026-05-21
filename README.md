# Home Assistant — Remote PC Reboot Button

[![Works with Home Assistant](https://img.shields.io/badge/works%20with-Home%20Assistant-41BDF5?style=for-the-badge&logo=home-assistant)](https://www.home-assistant.io)
A guide to adding a one-tap button in Home Assistant that reboots a Linux PC over SSH. Useful for machines that get stuck in sleep or become unresponsive without needing physical access.

---

## Prerequisites

- Home Assistant with the Terminal & SSH add-on (or web terminal access)
- A Linux PC on the same local network with SSH server installed
- The PC must be reachable while stuck (i.e. SSH still responds even when the display is frozen)

---

## Step 1: Enable SSH on the Linux PC

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## Step 2: Generate an SSH Key in Home Assistant

Open the HA web terminal and run:

```bash
ssh-keygen -t ed25519 -C "homeassistant" -f /root/.ssh/id_ed25519 -N ""
```

Copy the public key to the PC (replace `your_username` and `192.168.x.x`):

```bash
ssh-copy-id -i /root/.ssh/id_ed25519.pub your_username@192.168.x.x
```

You will be prompted for your PC password once. This is the last time it will be needed.

Test the connection:

```bash
ssh -i /root/.ssh/id_ed25519 your_username@192.168.x.x echo "it works"
```

---

## Step 3: Move the Key to the HA Config Directory

Home Assistant's shell commands run in a different context and cannot access `/root/.ssh/` directly. Copy the key somewhere HA can reach:

```bash
mkdir -p /config/.ssh
cp /root/.ssh/id_ed25519 /config/.ssh/id_ed25519
cp /root/.ssh/known_hosts /config/.ssh/known_hosts
chmod 600 /config/.ssh/id_ed25519
```

---

## Step 4: Allow Passwordless Reboot on the PC

SSH into the PC interactively:

```bash
ssh -i /config/.ssh/id_ed25519 your_username@192.168.x.x
```

Then run:

```bash
echo 'your_username ALL=(ALL) NOPASSWD: /sbin/reboot' | sudo tee /etc/sudoers.d/reboot-nopasswd
```

This grants passwordless sudo access **only** to the reboot command — nothing else.

Exit back to the HA terminal:

```bash
exit
```

Test it:

```bash
ssh -i /config/.ssh/id_ed25519 -o StrictHostKeyChecking=no your_username@192.168.x.x 'sudo reboot'
```

> ⚠️ This will immediately reboot the PC.

---

## Step 5: Add the Shell Command to Home Assistant

Edit `/config/configuration.yaml` and add:

```yaml
shell_command:
  restart_pc: "ssh -i /config/.ssh/id_ed25519 -o StrictHostKeyChecking=no your_username@192.168.x.x 'sudo reboot'"
```

Restart Home Assistant: **Settings → System → Restart → Quick Restart**

---

## Step 6: Verify in Developer Tools

Go to **Developer Tools → Actions**, enter `shell_command.restart_pc` as the action, and click **Perform Action**. The PC should reboot.

If you see a permission denied error, double-check that the key was copied to `/config/.ssh/` and not left in `/root/.ssh/`.

---

## Step 7: Add the Dashboard Button

In your dashboard's raw config editor, add:

```yaml
- type: button
  name: Reboot PC
  icon: mdi:desktop-classic
  tap_action:
    action: call-service
    service: shell_command.restart_pc
  confirmation:
    text: Are you sure you want to reboot the PC?
```

The `confirmation` dialog prevents accidental taps.

---

## Troubleshooting

**`Permission denied (publickey,password)`**
The SSH key path is wrong. Confirm the key exists at `/config/.ssh/id_ed25519` and that `configuration.yaml` points to that path.

**Button does nothing / no error**
Run the shell command from Developer Tools first to see the raw output. Check `stderr` in the response for clues.

**`sudo: a terminal is required`**
The sudoers rule is missing or malformed. SSH into the PC interactively and re-run the `tee` command in Step 4.

**PC is fully frozen and SSH won't connect**
SSH requires the PC's network stack to be responsive. If the machine is completely locked up, SSH will not reach it. In that case a smart plug (e.g. Shelly, TP-Link Kasa) with a power-cycle automation is the only remote option.

---

## Security Notes

- The sudoers rule grants passwordless `sudo reboot` only — no other commands are elevated
- SSH access still requires the private key, which is stored on the HA instance
- Use a static/reserved IP for the PC in your router to prevent the IP changing and breaking the connection
- If you have multiple users on the HA instance, be aware that anyone with dashboard access can trigger the reboot
