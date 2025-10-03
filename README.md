Complete Step-by-Step Guide: Setting up Zuul with External Node
Prerequisites
- Two machines: Main Machine (Zuul server) and Node Machine (execution node)

- Both machines should be on the same network and able to ping each other

- Docker and Docker Compose installed on Main Machine

- SSH access between machines

# On Main Machine (Zuul Server)
Step 1: Clone and Setup Zuul
# Clone Zuul repository
```
git clone https://opendev.org/zuul/zuul
```
```
cd zuul/doc/source/examples
```

# Generate SSH keys for nodepool
mkdir -p playbooks/files
ssh-keygen -t rsa -f ./playbooks/files/nodepool -N "" -q
chmod 600 playbooks/files/nodepool
chmod 644 playbooks/files/nodepool.pub
Step 2: Configure Docker Compose
Edit docker-compose.yml and find the executor service. Add volume mounts:

yaml
executor:
  # ... existing configuration ...
  volumes:
    - ./playbooks/files/nodepool:/root/.ssh/id_rsa:ro
    - ./playbooks/files/nodepool.pub:/root/.ssh/id_rsa.pub:ro
Step 3: Start Zuul Services
bash
# Stop any running services
sudo docker-compose down

# Clean everything (optional)
sudo docker system prune -af
sudo docker volume prune -f

# Start services
sudo docker-compose up -d

# Wait for full startup
sleep 90
Step 4: Verify Setup
bash
# Check if nodepool recognizes the node
sudo docker exec examples_launcher_1 nodepool list

# You should see your node in 'ready' state
On Node Machine (Execution Node)
Step 1: Prepare User and SSH Directory
bash
# Create SSH directory (replace 'minson' with your username)
sudo mkdir -p /home/minson/.ssh
sudo chmod 700 /home/minson/.ssh

# Get the PUBLIC KEY from Main Machine and add to authorized_keys cat playbooks/files/nodepool.pub
# Copy the content of playbooks/files/nodepool.pub from Main Machine
echo 'PASTE_PUBLIC_KEY_HERE' | sudo tee /home/minson/.ssh/authorized_keys

# Set proper permissions
sudo chmod 600 /home/minson/.ssh/authorized_keys
sudo chown -R minson:minson /home/minson/.ssh
Step 2: Verify Node Accessibility
bash
# Find your node's IP address
ip addr show

# Make sure SSH service is running
sudo systemctl status ssh
Back on Main Machine - Final Verification
Step 1: Test SSH Connection
bash
# Replace IP with your Node Machine's IP
sudo docker exec examples_executor_1 ssh -o StrictHostKeyChecking=no minson@NODE_IP "echo SSH test successful"

# Test more commands
sudo docker exec examples_executor_1 ssh -o StrictHostKeyChecking=no minson@NODE_IP "whoami && pwd"
Step 2: Verify Nodepool Status
bash
sudo docker exec examples_launcher_1 nodepool list

# Expected output:
# +------------+------------+---------------+---------------+---------------+------+-------+-------------+----------+
# | ID         | Provider   | Label         | Server ID     | Public IPv4   | IPv6 | State | Age         | Locked   |
# +------------+------------+---------------+---------------+---------------+------+-------+-------------+----------+
# | 0000000005 | static-vms | external-node | NODE_IP       | NODE_IP       |      | ready | 00:00:10:00 | unlocked |
# +------------+------------+---------------+---------------+---------------+------+-------+-------------+----------+
Troubleshooting Common Issues
If SSH fails:
bash
# 1. Check if keys match
cat playbooks/files/nodepool.pub
sudo docker exec examples_executor_1 cat /root/.ssh/id_rsa.pub

# 2. Manual key copy (if volumes aren't working)
sudo docker cp playbooks/files/nodepool examples_executor_1:/root/.ssh/id_rsa
sudo docker cp playbooks/files/nodepool.pub examples_executor_1:/root/.ssh/id_rsa.pub
sudo docker exec examples_executor_1 chmod 600 /root/.ssh/id_rsa
sudo docker exec examples_executor_1 chmod 644 /root/.ssh/id_rsa.pub

# 3. Debug SSH connection
sudo docker exec examples_executor_1 ssh -vvv -o StrictHostKeyChecking=no minson@NODE_IP "echo test"
If node doesn't appear in nodepool:
bash
# Check Zuul logs
sudo docker-compose logs launcher
sudo docker-compose logs executor

# Restart services
sudo docker-compose restart
Configuration Files Summary
Main Machine Files:
docker-compose.yml - Zuul services configuration

nodepool.yaml - Node definitions

playbooks/files/nodepool - SSH private key

playbooks/files/nodepool.pub - SSH public key

Node Machine Files:
/home/minson/.ssh/authorized_keys - Contains the public key from main machine

Quick Test Commands
Once setup is complete, run these to verify:

bash
# On Main Machine
sudo docker exec examples_launcher_1 nodepool list
sudo docker exec examples_executor_1 ssh -o StrictHostKeyChecking=no minson@NODE_IP "echo 'Zuul setup successful!'"
This complete guide should get anyone from zero to a working Zuul setup with external nodes!

