# Ansible Playbooks for FreeBSD/Debian/elementary OS

A very small collection of flexible Ansible playbooks for initial configuration of FreeBSD and Debian/elementary OS hosts.

The YAML examples below show the minimal properties needed to configure each specific playbook. See the [vars](vars) directory for complete example files.

These playbooks are documented in the order they should be applied to a fresh system.

- [bootstrap.yaml](#bootstrapyaml): Bootstrap a FreeBSD/Linux system to be managed by Ansible.
- [initial-debian-config.yaml](#initial-debian-configyaml): Perform initial Debian configuration.
- [initial-elementary-config.yaml](#initial-elementary-configyaml): Perform initial elementary OS configuration.
- [firefox.yaml](#firefoxyaml): Configure the Firefox profile for a user.
- [enable-sync.yaml](#enable-syncyaml): Install and configure sync tools for a user.

### bootstrap.yaml

Bootstrap a FreeBSD/Linux system to be managed by Ansible.

Targets systems that are accessible over SSH but are not yet under Ansible management. Python must already be installed on the remote host.

- Create a group for the ansible user.
- Allow the group to use sudo without a password.
- Create the ansible user and add it to the passwordless sudo group.
- On FreeBSD, sync the password database.
- Set SSH authorized key for the ansible user.

#### Usage

Example _vars/users.yaml_:
```
---
users:
  ansible:
    group: sudo-nopasswd
    name: ansible
    public_key: '{{ lookup("file", lookup("env", "HOME") + "/.ssh/ansible.id_rsa.pub") }}'
```

Running the playbook:
```
ansible-playbook bootstrap.yaml -l <newHost> -u <existingUser> --ask-pass --ask-become-pass
```

### initial-debian-config.yaml

Perform initial Debian configuration.

- Update apt cache and all installed packages.
- Install essential packages.
- Uninstall unused packages.
- Install required Python modules.
- Enable shell history searching.

#### Usage

Example _vars/systems.yaml_:
```
---
systems:
  all:
    packages:
      install: []
      uninstall: []
  laptops:
    packages:
      install:
        - firefox
        - guake
        - keepassxc
        - krita
        - libreoffice-writer
        - python3-pip
        - rdesktop
      uninstall:
        - libreoffice-math
    python_modules:
      - psutil # Required by dconf plugin.
  lenny:
    category: laptops
```

Running the playbook:
```
ansible-playbook initial-debian-config.yaml -l <newHost>
```

### initial-elementary-config.yaml

Perform initial elementary OS configuration.

Includes [initial-debian-config.yaml](#initial-debian-configyaml) plus the following tasks:

- Allow limited SSH connections.
- Deny all other incoming traffic by default.
- If desired, enable the Guest account.
- Enable local network DNS resolution.
- Check whether Bluetooth is blocked.
- Enable Bluetooth if it is blocked.
- Create shortcuts for rdesktop access.
- Create required directories.
- Enable autostart applications.
- Apply dconf settings.
- Remove default icons from Plank.
- Add desired icons to Plank.
- Configure KeePassXC if installed.

#### Usage

Example _vars/systems.yaml_:
```
---
systems:
  all:
    packages:
      install: []
      uninstall: []
  laptops:
    firewall:
      ssh_from_ips:
        - 192.168.1.100/32 # Specific host
        - 192.168.1.0/24 # LAN segment
    hardware:
      bluetooth:
        enabled: true
    packages:
      install:
        - firefox
        - guake
        - keepassxc
        - krita
        - libreoffice-writer
        - python3-pip
        - rdesktop
      uninstall:
        - libreoffice-math
    python_modules:
      - psutil # Required by dconf plugin.
  lenny:
    category: laptops
```

Example _vars/users.yaml_:
```
---
users:
  guest:
    enabled: no
  all:
    autostart: []
    directories:
      - path: ~/.config/autostart
    plank:
      add:
        - firefox
        - io.elementary.files
        - org.keepassxc.KeePassXC
      remove:
        - gala-multitaskingview
        - io.elementary.appcenter
        - io.elementary.calendar
        - io.elementary.switchboard
        - io.elementary.videos
        - org.gnome.Epiphany
        - org.pantheon.mail
    # Boolean values need to be in single quotes so they are lowercase in the prefs file.
    # String values need to include the double quotes to include them in the prefs file.
    preferences:
      dconf:
        - option: /apps/guake/general/use-popup
          value: 'false'
        - option: /apps/guake/general/window-ontop
          value: 'false'
        - option: /apps/guake/general/window-refocus
          value: 'true'
        - option: /apps/guake/general/window-width
          value: 50
        - option: /apps/guake/keybindings/global/show-hide
          value: "'F10'"
        - option: /io/elementary/desktop/wingpanel/power/show-percentage
          value: 'true'
        - option: /net/launchpad/plank/docks/dock1/icon-size
          value: 64
        - option: /org/gnome/desktop/peripherals/touchpad/natural-scroll
          value: 'false'
        - option: /org/gnome/desktop/peripherals/touchpad/speed
          value: 0.5
        - option: /org/gnome/desktop/privacy/remove-old-temp-files
          value: 'true'
        - option: /org/gnome/settings-daemon/plugins/color/night-light-enabled
          value: 'true'
        # Automatically adjust display brightness (doesn't work in a VM):
        - option: /org/gnome/settings-daemon/plugins/power/ambient-enabled
          value: 'true'
      keepassxc:
        - option: AutoSaveOnExit
          section: General
          value: 'true'
        - option: AutoTypeStartDelay
          section: General
          value: 500
        - option: HideWindowOnCopy
          section: General
          value: 'true'
        - option: MinimizeOnCopy
          section: General
          value: 'true'
    rdesktop_hosts:
      - Beastie
  chris:
    autostart:
      - dest: '~/.config/autostart/guake.desktop'
        src: '/usr/share/guake/data/guake.template.desktop'
    plank:
      add:
        - Beastie
      remove:
        - io.elementary.music
        - io.elementary.photos
  crystal:
    plank:
      add:
        - org.kde.krita
```

Running the playbook:
```
ansible-playbook initial-elementary-config.yaml -l <newHost> --extra-vars "user=chris"
```

### firefox.yaml

Configure the Firefox profile for a user.

- Get Firefox profile name.
- Fail if Firefox profile was not detected.
- Make Firefox the default browser.
- Configure Firefox preferences.

#### Usage

Example _vars/users.yaml_:
```
---
users:
  all:
    # Boolean values need to be in single quotes so they are lowercase in the prefs file.
    # String values need to include the double quotes to include them in the prefs file.
    preferences:
      firefox:
        - option: browser.startup.page
          value: 3 # Restore tabs.
        - option: browser.search.widget.inNavBar
          value: 'true'
        - option: browser.urlbar.placeholderName
          value: '"DuckDuckGo"'
        - option: browser.urlbar.suggest.searches
          value: 'false'
        - option: browser.contentblocking.category
          value: '"custom"'
        - option: privacy.annotate_channels.strict_list.enabled
          value: 'true'
        - option: privacy.trackingprotection.enabled
          value: 'true'
        - option: privacy.trackingprotection.socialtracking.enabled
          value: 'true'
        - option: privacy.donottrackheader.enabled
          value: 'true'
        - option: signon.rememberSignons
          value: 'false'
        - option: extensions.formautofill.creditCards.enabled
          value: 'false'
        - option: browser.discovery.enabled # Personalized extension recommendations.
          value: 'false'
        - option: datareporting.healthreport.uploadEnabled
          value: 'false'
        - option: app.shield.optoutstudies.enabled
          value: 'false'
        - option: dom.security.https_only_mode
          value: 'true'
        - option: dom.security.https_only_mode_ever_enabled
          value: 'true'
        - option: browser.uiCustomization.state # Navigation bar / toolbar.
          value: '"{\"placements\":{\"widget-overflow-fixed-list\":[],\"nav-bar\":[\"back-button\",\"forward-button\",\"stop-reload-button\",\"home-button\",\"urlbar-container\",\"downloads-button\",\"search-container\",\"ublock0_raymondhill_net-browser-action\",\"jid1-mnnxcxisbpnsxq_jetpack-browser-action\",\"_c2c003ee-bd69-42a2-b0e9-6f34222cb046_-browser-action\"],\"toolbar-menubar\":[\"menubar-items\"],\"TabsToolbar\":[\"tabbrowser-tabs\",\"new-tab-button\",\"alltabs-button\"],\"PersonalToolbar\":[\"import-button\",\"personal-bookmarks\"]},\"seen\":[],\"dirtyAreaCache\":[\"nav-bar\",\"PersonalToolbar\",\"toolbar-menubar\",\"TabsToolbar\"],\"currentVersion\":16,\"newElementCount\":4}"'
  chris:
    preferences:
      firefox:
        - option: network.cookie.cookieBehavior
          value: 2 # Block all cookies.
  crystal:
    preferences:
      firefox:
        - option: network.cookie.cookieBehavior
          value: 1 # Block all 3rd-party cookies.
```

Running the playbook:
```
ansible-playbook firefox.yaml -l <newHost> --extra-vars "user=chris"
```

### enable-sync.yaml

Install and configure sync tools for a user.

This playbook is geared toward elementary OS.

- Add an apt signing key for Syncthing.
- Add PPA for Nextcloud and Syncthing.
- Install Nextcloud and Syncthing.
- Create directories for config/sync.
- Configure the Nextcloud client.
- Enable autostart for the Nextcloud client.
- Copy Syncthing config files.

#### Usage

Example _vars/users.yaml_:
```
---
users:
  chris:
    nextcloud:
      host: cloud.example.com
      journal_path: ._sync_012345abcdef.db
      username: chris
  crystal:
    nextcloud:
      host: cloud.example.net
      journal_path: ._sync_abcdef012345.db
      username: crystal
```

Running the playbook:
```
ansible-playbook enable-sync.yaml -l <newHost> --extra-vars "user=chris"
```
