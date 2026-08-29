# Configuration Management with Ansible

Configures EC2 instances provisioned by Terraform.

## Structure
- `ansible.cfg` — defaults
- `inventory/hosts.ini` — replace EC2_PUBLIC_IP with `terraform output ec2_public_ip`
- `playbooks/site.yml` — main playbook
- `roles/webserver` — nginx install + hardening
- `roles/hardening` — ssh hardening, dnf-automatic

## Usage
```bash
# Install ansible (WSL / control node)
pip install ansible

# Test connectivity
ansible all -m ping -i inventory/hosts.ini

# Dry-run
ansible-playbook playbooks/site.yml -i inventory/hosts.ini --check --diff

# Apply
ansible-playbook playbooks/site.yml -i inventory/hosts.ini
```

## From single control node to many EC2s
Add more hosts to `[webservers]` group and re-run — idempotent.
