Some notes about using the SSH Jump feature:

```
centos@zoobox /home/centos/soft/terraform-aws-openshift.old [sshjumper] $ cat modules/openshift/10-sshconfig.tf
// Collect the vars needed for templating the sshconfig.cfg file
data "template_file" "sshconfig" {
  template = "${file("${path.cwd}/sshconfig.template.cfg")}"
  vars {
    bastion-public_ip = "${aws_eip.bastion_eip.public_ip}"
  }
}

//  Create the sshconfig.cfg file
resource "local_file" "sshconfig" {
  content     = "${data.template_file.sshconfig.rendered}"
  filename = "${path.cwd}/sshconfig.cfg"
}
centos@zoobox /home/centos/soft/terraform-aws-openshift.old [sshjumper] $ cat sshconfig.template.cfg
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
    LogLevel QUIET

Host bastion
    Hostname ${bastion-public_ip}
    User ec2-user
    IdentityFile ~/.ssh/id_rsa
    ForwardAgent yes

Host master
    Hostname master.openshift.local
    ProxyJump bastion
    User ec2-user

Host node1
    Hostname node1.openshift.local
    ProxyJump bastion
    User ec2-user

Host node2
    Hostname node2.openshift.local
    ProxyJump bastion
    User ec2-user
    
centos@zoobox /home/centos/soft/terraform-aws-openshift.old [sshjumper] $ cat makefile
MYSSH = ssh -F sshconfig.cfg
MYSCP = scp -F sshconfig.cfg

infra:
        # Get the modules, create the infrastructure.
        terraform init && terraform get && terraform apply

ssh:
        # Add our identity for ssh, add the host key to avoid having to accept the
        # te host key manually. Also add the identity of each node to the bastion.
        $(MYSSH) bastion "ssh-keyscan -t rsa -H master.openshift.local >> ~/.ssh/known_hosts"
        $(MYSSH) bastion "ssh-keyscan -t rsa -H node1.openshift.local >> ~/.ssh/known_hosts"
        $(MYSSH) bastion "ssh-keyscan -t rsa -H node2.openshift.local >> ~/.ssh/known_hosts"

        # Test to all hosts
        $(MYSSH) bastion whoami
        $(MYSSH) master whoami
        $(MYSSH) node1 whoami
        $(MYSSH) node2 whoami

        # set nicer hostnames
        $(MYSSH) master  "sudo hostnamectl set-hostname master.openshift.local"
        $(MYSSH) node1   "sudo hostnamectl set-hostname node1.openshift.local"
        $(MYSSH) node2   "sudo hostnamectl set-hostname node2.openshift.local"

# Installs OpenShift on the cluster.
openshift:
        # Copy our inventory to the master and run the install script.
        $(MYSCP) ./inventory.cfg bastion:~
        cat install-from-bastion.sh | $(MYSSH) bastion

        # Now the installer is done, run the postinstall steps on each host.
        cat ./scripts/postinstall-master.sh | $(MYSSH) master
        cat ./scripts/postinstall-node.sh   | $(MYSSH) node1
        cat ./scripts/postinstall-node.sh   | $(MYSSH) node2

# Destroy the infrastructure.
destroy:
        terraform init && terraform destroy -auto-approve

# Open the console.
browse-openshift:
        open $$(terraform output master-url)

# SSH onto the master.
ssh-bastion:
        $(MYSSH) bastion
ssh-master:
        $(MYSSH) master
ssh-node1:
        $(MYSSH) node1
ssh-node2:
        $(MYSSH) node2

# Create sample services.
sample:
        oc login $$(terraform output master-url) --insecure-skip-tls-verify=true -u=admin -p=123
        oc new-project sample
        oc process -f ./sample/counter-service.yml | oc create -f -

# Lint the terraform files. Don't forget to provide the 'region' var, as it is
# not provided by default. Error on issues, suitable for CI.
lint:
        terraform get
        TF_VAR_region="us-east-1" tflint --error-with-issues

# Run the tests.
test:
        echo "Simulating tests..."

# Run the CircleCI build locally.
circleci:
        circleci config validate -c .circleci/config.yml
        circleci build --job lint

.PHONY: sample
```
