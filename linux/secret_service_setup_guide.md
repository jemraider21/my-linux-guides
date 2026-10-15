# Resolving Secret Service / Keyring Issues on Non-GNOME Desktops

## Overview

Modern Linux GUI applications—particularly those built on Electron, Chromium, or GTK—frequently rely on the Freedesktop [Secret Service API](https://specifications.freedesktop.org/secret-service-spec/latest/) (`org.freedesktop.secrets`) through libraries such as `libsecret`. This interface provides secure, encrypted storage for sensitive data, including:

- OAuth session tokens and login credentials
- API keys and environment variables
- Extension secrets and authentication tokens

On full desktop environments like GNOME or KDE Plasma, a secret service daemon is integrated out of the box (`gnome-keyring` or `ksecretservice`). However, minimalist or standalone desktop environments/window managers—such as **COSMIC**, **Hyprland**, **Sway**, **i3**, or **bspwm**—do not ship with an active Secret Service provider by default.

When an application fails to find a registered provider on the user D-Bus session, it typically warns the user or falls back to insecure in-memory storage.

### Common Symptom Example

When launching an Electron/Chromium-based desktop application (such as the community package of **Claude Desktop**, **VS Code**, **Slack**, or **Discord**), you may encounter an alert similar to:

> *No system keyring is available. Your sign-in, SSH passwords, and environment variables can't be saved on this computer, and extension secrets aren't encrypted. Install and unlock GNOME Keyring, then restart the application.*

---

## 1. Prerequisites & Package Installation

On Arch Linux and derivatives, install `gnome-keyring` along with `libsecret` (the client library) and `seahorse` (an optional graphical management tool to inspect keys):

```bash
sudo pacman -S gnome-keyring libsecret seahorse
```

For Debian/Ubuntu-based distributions:
```bash
sudo apt install gnome-keyring libsecret-1-0 seahorse
```

For Fedora:
```bash
sudo dnf install gnome-keyring libsecret seahorse
```

---

## 2. PAM Configuration for Automatic Unlocking

To avoid entering a separate password prompt every time you log in or open an application, configure Pluggable Authentication Modules (PAM) to unlock the `login` keyring automatically using your system login credentials.

### Identifying Your Active Display Manager / PAM File

PAM configuration files are stored in `/etc/pam.d/`. The specific file you need to modify depends on the display manager or login method your system uses.

| Login / Display Manager | Primary File(s) in `/etc/pam.d/` |
| :--- | :--- |
| **greetd** *(common on Wayland / COSMIC / Hyprland)* | `/etc/pam.d/greetd` |
| **cosmic-greeter** | `/etc/pam.d/cosmic-greeter` |
| **GDM** *(GNOME Display Manager)* | `/etc/pam.d/gdm-password` |
| **SDDM** *(Simple Desktop Display Manager)* | `/etc/pam.d/sddm` |
| **LightDM** | `/etc/pam.d/lightdm` |
| **Ly** | `/etc/pam.d/ly` |
| **TTY / Console Login** *(startx / direct Wayland)* | `/etc/pam.d/login` |

> **Tip:** If you are unsure which display manager is running, run:
> ```bash
> systemctl status display-manager.service
> ```
> Inspect the active service name to locate the corresponding entry in `/etc/pam.d/`.

### Required PAM Rules

Open the appropriate file with root privileges (for example, `/etc/pam.d/greetd`):

```bash
sudo nano /etc/pam.d/greetd
```

1. **In the `auth` section:** Add `pam_gnome_keyring.so` at the end of the authentication stack.
2. **In the `session` section:** Add `pam_gnome_keyring.so auto_start` at the end of the session stack.

#### Example Configuration (`/etc/pam.d/greetd` or `/etc/pam.d/login`)

```pam
#%PAM-1.0

auth       include      system-login
auth       optional     pam_gnome_keyring.so

account    include      system-login

password   include      system-login

session    include      system-login
session    optional     pam_gnome_keyring.so auto_start
```

> **Important:** Your user login password and the GNOME Keyring password must be identical for automatic unlocking to succeed.

---

## 3. Starting the Daemon in the Session Environment

In modern setups, `gnome-keyring-daemon` is automatically triggered via D-Bus activation when an application requests `org.freedesktop.secrets`. 

If your environment does not propagate D-Bus session variables automatically, you can explicitly export the environment variables or launch the daemon in your shell profile (`~/.bash_profile`, `~/.zprofile`) or desktop autostart:

```bash
eval $(gnome-keyring-daemon --start --components=secrets,pkcs11,ssh)
export SSH_AUTH_SOCK
```

---

## 4. Verification

After saving the configuration, log out of your desktop session and log back in (or reboot). 

### Step 4.1: Verify D-Bus Registration

Run the following command to check if the Secret Service interface is owned on the user session bus:

```bash
busctl --user status org.freedesktop.secrets
```

Expected output:
- The service should be found.
- The unit or PID should correspond to `gnome-keyring-daemon`.

### Step 4.2: Test Secret Storage

You can verify that `libsecret` can read and write credentials using the `secret-tool` utility:

```bash
# Store a test secret
echo "mypassword" | secret-tool store --label="Test Secret" service test user testuser

# Retrieve the test secret
secret-tool lookup service test user testuser

# Clear the test secret
secret-tool clear service test user testuser
```

If the lookup returns `mypassword` without prompting you for a master password unlock screen, the keyring is active and properly unlocked.

---

## 5. Application Launch

Launch the affected program (e.g., Claude Desktop, VS Code, or Chromium). The missing keyring banner should no longer appear, and all session data, extensions, and OAuth tokens will persist securely across system restarts.