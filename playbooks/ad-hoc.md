# Run
ansible all -m ping -i inventory/hosts.ini
ansible-playbook playbooks/site.yml -i inventory/hosts.ini --check
ansible-playbook playbooks/site.yml -i inventory/hosts.ini

# Dynamic inventory from Terraform (after terraform apply):
# terraform output -json | jq -r '.ec2_public_ip.value' > /tmp/ip && sed -i "s/EC2_PUBLIC_IP/$ip/" inventory/hosts.ini
