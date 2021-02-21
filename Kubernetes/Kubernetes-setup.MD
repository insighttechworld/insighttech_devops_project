-------------------------- Setup Kubernetes (K8s) Cluster on AWS -----------------------


---------------------
1. Create Ubuntu EC2 instance


---------------------
2. Install AWSCLI

 curl https://s3.amazonaws.com/aws-cli/awscli-bundle.zip -o awscli-bundle.zip
 apt install unzip python
 unzip awscli-bundle.zip
 #sudo apt-get install unzip ---> if you dont have unzip in your system
 ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
aws --version       --> to confirm


----------------------
3. Install kubectl on ubuntu instance

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl      --->  to try kubectl command

Try kops command to see if kops is installed
kops


---------------------
4. Install kops on ubuntu instance

curl -LO  https://github.com/kubernetes/kops/releases/download/1.15.0/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
kops version    -->  (it should be 1.15.0)
kops

Note: Use the below command if you wish to use latest version. For now we could see latest version of kops. So ignore it until further update. 
# curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

5. Create an IAM user/role with Route53, EC2, IAM and S3 full access
Services -->  IAM  -->  Roles  -->  Create a role  -->  choose EC2  -->  Next Permission  -->  attach policies here (EC2 fullaccess, S3 fullaccess, Route53 fullaccess and IAM fullaccess)  --> Next Tags  -->  Name: k8s-role  --> Next: Review  -->  Role name: k8s-role  --> Create role  


-------------------
6. Attach IAM role to ubuntu EC2 instance

Goto EC2 Instance  --> select the EC2 instance  -->  Actions  -->  Instance settings  -->  Attach/Replace IAM Role  --> choose k8s-role  -->  Apply  --> Close

# Note: If you create IAM user with programmatic access then provide Access keys. Otherwise region information is enough

aws configure
root@ip-172-31-58-3:/home/ubuntu# aws configure
AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]: us-east-1   --> (enter your EC2 instance region)
Default output format [None]:


--------------------
7. Create a Route53 private hosted zone (you can create Public hosted zone if you have a domain)

Goto AWS console   --> services  -->  Networking  --> Route53  --> Get Started now  --> Create Hosted Zone  -->  Domain name: insighttech.net  -->  Private Hosted zone  -->  choose your region [us-east-1]  -->  choose VPC info ID --> Create

Route53 --> hosted zones --> created hosted zone  
Domain Name: insighttech.net
Type: Private hosted zone for Amazon VPC. Make sure you are chosing right VPC if you have multiple


--------------------
8. create an S3 bucket (on awsCLI or AWS Console)
goto AWS console  --> s3 to confirm existing buckets

aws s3 mb s3://demo.k8s.insighttech.net

root@ip-172-31-58-3:/home/ubuntu# aws s3 mb s3://demo.k8s.insighttech.net
make_bucket: demo.k8s.insighttech.net

go back to AWS console to confirm creation of the bucket.


--------------------
9. Expose environment variable: need to export the s3 bucket, so that kops could use it

export KOPS_STATE_STORE=s3://demo.k8s.insighttech.net


--------------------
10. Create sshkeys before creating cluster - because this will be used to login to our kubernetes cluster, without this, we won't be able to login to our kubernetes cluster

ssh-keygen
cd ~/.ssh
ls -l

the public key will be copied to kubernetes cluster, so by using the key, we can login tothe cluster seemlessly

11. Create kubernetes cluster definitions on S3 bucket  -- need to specify available region based on the region we created all our resources

kops create cluster --cloud=aws --zones=us-east-1a --name=demo.k8s.insighttech.net --dns-zone=insighttech.net --dns private 

Output:
Must specify --yes to apply changes

Cluster configuration has been created.

Suggestions:
 * list clusters with: kops get cluster
 * edit this cluster with: kops edit cluster demo.k8s.insighttech.net
 * edit your node instance group: kops edit ig --name=demo.k8s.insighttech.net nodes
 * edit your master instance group: kops edit ig --name=demo.k8s.insighttech.net master-us-east-1a

Finally configure your cluster with: kops update cluster --name demo.k8s.insighttech.net --yes


--------------------
12. Create kubernetes cluser, whenever this command is executed, the cluster get created

kops update cluster demo.k8s.insighttech.net --yes

OUTPUT:
Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.demo.k8s.insighttech.net
 * the admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons at: https://github.com/kubernetes/kops/blob/master/docs/addons.md.

kops validate cluster    ---> to validate the k8s clusters

OUTPUT:
root@ip-172-31-83-254:~/.ssh# kops validate cluster
Using cluster from kubectl context: demo.k8s.insighttech.net

Validating cluster demo.k8s.insighttech.net

INSTANCE GROUPS
NAME                    ROLE    MACHINETYPE     MIN     MAX     SUBNETS
master-us-east-1a       Master  m3.medium       1       1       us-east-1a
nodes                   Node    t2.medium       2       2       us-east-1a

NODE STATUS
NAME                            ROLE    READY
ip-172-20-52-147.ec2.internal   master  True
ip-172-20-55-91.ec2.internal    node    True
ip-172-20-61-111.ec2.internal   node    True

Your cluster demo.k8s.insighttech.net is ready

Note:
After executing this command, it will create other additional resources in AWS, new file in s3, IAM roles, Route53 records set and routes, EC2 instaces launched and Launched configuration as well as Auto-scalling group in case of any failure.

Next is to ssh to the master: 
ssh -i ~/.ssh/id_rsa admin@api.demo.k8s.insighttech.net

Now you are in the Master K8s cluster, you can confirm the with the master-us-east master on AWS console (the Private IP)

exit    --> to go back to Ubuntu environment


-------------------
13. To change the kubernetes master and worker instance sizes
You can edit the nodes and master nodes name and instance type:

##kops edit ig --name=<cluster_name> nodes
kops edit ig --name=demo.k8s.insighttech.net nodes 

##kops edit ig --name=<cluster_name> master-<zone_name>
kops edit ig --name=demo.k8s.insighttech.net master-us-east-1a


-------------------
14. Validate your cluster
kops validate cluster


-------------------
15. To list nodes
kubectl get nodes


------------------
16. To Delete cluster (try once your lab is done) once you delete, all definitions on AWS account will be deleted and will remove the resources as well
#kops delete cluster <cluster_name> --yes
kops delete cluster demo.k8s.insighttech.net --yes
