# Use the willhallonline/ansible:latest Docker image as base, which is based on Alpine
FROM willhallonline/ansible:latest

# Install bash, docker, python3, pip, and requests
RUN apk add --no-cache bash docker python3 py3-pip && \
    pip3 install --upgrade pip && \
    pip3 install requests==2.31.0

# Install community.postgresql collection
RUN ansible-galaxy collection install community.postgresql

# Copy Ansible configuration files and playbooks
COPY ansible.cfg /ansible/ansible.cfg
COPY inventory /ansible/inventory
COPY playbooks /ansible/playbooks

# Command to keep the container running
CMD ["sleep", "infinity"]
