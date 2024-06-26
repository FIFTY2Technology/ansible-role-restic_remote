# Description
This role configures automated backups for remote backup clients which cannot reach the backup server directly, e.g. due to firewalls, NAT, etc. The backup server must be able to reach the remote client via SSH.

This role configures both restic remote clients (machines that need files backed up) and the restic REST server (which hosts the restic repositories). Once this role is deployed, the backup server starts a systemd service `restic-remote-backup@<remote-client>.service`, which initiates an SSH connection, forwarding the backup server's port to the remote restic client via SSH and then invoking the `restic-backup.service` there.

<details><p><summary>Flowchart</summary>

```mermaid
sequenceDiagram
    Backup Server->>+Backup Server: systemctl start restic-remote-backup@Client.service
    Backup Server->>+Remote Client: Open SSH tunnel, forward REST server port 3000,<br>execute sudo systemctl start restic-backup.service

    Remote Client->>+restic-backup.service: Starting restic-backup.service - Restic backup...
    restic-backup.service->>restic-backup.service: restic init ...<br>restic backup ...
    restic-backup.service->>Remote Client: restic: open repository at rest:http(s)://user:pass@Server:3000/Client
    Remote Client->>Remote Client: "Server" always resolves to IP 127.0.0.1
    restic-backup.service->>Backup Server: using parent snapshot d289ee09<br>...<br>snapshot fa15be6f saved
    restic-backup.service->>-Remote Client: restic-backup.service: Deactivated successfully.
    Remote Client->>-Backup Server: Finished restic-backup.service - Restic backup.
    Backup Server->>-Backup Server: restic-remote-backup@Client.service: Succeeded.
```
</p></details>

You can regard this role as kind of an extension to the `fifty2technology.restic_client` role, as they are both integrated into each other, but it once made more sense to unload the `restic_client` role and separate the "remote client" functionality from it. It is also coupled with the `fifty2technology.restic_server` role, since the backup server needs to be configured to connect to remote clients - this means it can only work with clients with a [restic REST server](https://github.com/restic/rest-server) account configured as a repository.

Once a week (on saturday night) the backup server also connects to each remote client and runs the `restic-prune.service`.

For monitoring purposes and if configured, the backup server connects to each remote machine once every day via `restic-remote-exporter@.service` to gather metrics about the latest backup snapshot. This cannot be done by the client itself since it requires access to the restic repository on the server.

# Requirements
* The variable `restic_client_is_remote` must be set to `true` to enable remote backups. It must also be set during `restic_client` role deployment so it configures and activates systemd services correctly.
* `backup_server` must be set to the `inventory_hostname` of the backup server, e.g. `backupserver.example.com`.
* `backup_server_password` must contain the login credentials for the backup server if SSH password-authentication is used. With key-based authentication, this can be left undefined (will default to empty string).
* `backup_server_become_password` must contain the privilege escalation password for the unprivileged user on the backup server. The same as above applies here in case of key-based authentication.

The `backup_server_*password` variables are necessary for password-only authentication, so that `delegate_to` tasks to the backup server are run correctly. It is not necessary if you use SSH login with keys and/or as privileged user.

# Role Variables
All variables which can be overridden are stored in defaults/main.yml file as well as in table below.

| Name | Default value | Description |
| ------ | ------ | ----- |
| `restic_remote_server_user` | `"{{ restic_server_user \| default('rest-server') }}"` | Username to use for initiating SSH connections as from server to client |
| `restic_remote_server_group` | `"{{ restic_server_group \| default('rest-server') }}"` | Groupname used to assert that a SSH private/public keypair exists with correct ownership on the server. The keypair should exist already (created in `restic_server` role). |
| `restic_remote_client_port` | `"{{ restic_server_listen_port \| default('3000') }}"` | Port to use on client side to terminate SSH tunnel coming from the backup server. Set to a port not used by something else on the client. Defaults to the same port the `restic_server` role uses to let the REST server listen on. |
| `restic_remote_client_user` | `"{{ restic_client_user \| default('restic') }}"` | Username used by the server to connect to the client via SSH. The server then starts the `restic-backup.service` as this user via `sudo systemctl start restic-backup.service`. |
| `restic_remote_ssh_port` | 22 | SSH port of the client, which will be used by the backup server to connect. This role does not configure 'Port' in SSHD config! |
| `restic_remote_client_group` | `"{{ restic_client_group \| default('restic') }}"` | Groupname getting granted `sudo` privileges on client to start systemd services `restic-backup.service` and `restic-prune.service`. The client's user must be in this group. |
| `restic_remote_client_is_remote` | `"{{ restic_client_is_remote \| default(false) }}"` | Repeat from `restic_client` role to check whether client should be configured to be backed up remotely by the backup server. |
| `restic_remote_no_log` | true | Do not show sensible content when printed by `ansible-playbook` runs. Set to `false` for debugging e.g. repository file templating. |

# Dependencies
The `restic_client` role must be run on targets beforehand, and the `restic_server` role for the restic rest-server must be deployed already.

# Example Playbook
```
- hosts: all
  gather_facts: false
  become: true

  vars:
    restic_client_is_remote: true
    backup_server: backupserver.example.com

  roles:
    - role: restic_client
    - role: restic_remote
```

## Testing with Molecule
Example how to fully test the whole `restic_remote` role from deploy to a remote backup trigger:
1. Run `molecule converge`
2. Run `molecule login -h backupserver` to login to the backupserver container.
   1. Run `systemctl start restic-remote-backup@restic_remote_debian12.service` to test if the backup is working.
   2. Run `journalctl -fe` to check for possible errors.
3. Run `molecule login -h restic_remote_debian12`
   1. Run `journalctl -ft restic` to check for possible errors.

Dependig on the container environment, molecule containers might not be able to resolve each others hostnames, breaking the SSH connection attempts in the `restic-remote-backup@<remote-client>.service`. In this case, add the `restic_remote_debian12` hostname to `/etc/hosts` manually on the `backupserver`. In a productive setup, hostnames must be resolvable via DNS.
