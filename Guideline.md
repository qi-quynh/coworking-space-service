1. Check IAM Permission
aws sts get-caller-identity

2. Create/Update an EKS Cluster
eksctl create cluster --name coworking-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2
aws eks --region us-east-1 update-kubeconfig --name coworking-cluster

3. Create YAML Configurations
kubectl apply -f pvc.yaml
kubectl apply -f pv.yaml
kubectl apply -f postgresql-deployment.yaml
kubectl apply -f postgresql-service.yaml

4. Set Up kubectl Port-forwarding
kubectl port-forward service/postgresql-service 5432:5432 
ps aux | grep 'kubectl port-forward' | grep -v grep | awk '{print $2}' | xargs -r kill

5. Run Seed Files
set PGPASSWORD=mypassword
psql --host=127.0.0.1 -U myuser -d mydatabase -p 5432 -f "1_create_tables.sql"
psql --host=127.0.0.1 -U myuser -d mydatabase -p 5432 -f "2_seed_users.sql"
psql --host=127.0.0.1 -U myuser -d mydatabase -p 5432 -f "3_seed_tokens.sql"

6. Build the Analytics Application Locally
pip install -r requirements.txt
$env:DB_USERNAME="myuser"
$env:DB_PASSWORD="mypassword"
$env:DB_HOST="127.0.0.1"
$env:DB_PORT="5432"
$env:DB_NAME="mydatabase"
python app.py

curl 127.0.0.1:5153/api/reports/daily_usage
curl 127.0.0.1:5153/api/reports/user_visits

7. Deploy the Analytics Application
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 310722232770.dkr.ecr.us-east-1.amazonaws.com
docker build -t coworking .
docker tag coworking:latest 310722232770.dkr.ecr.us-east-1.amazonaws.com/coworking:latest
docker push 310722232770.dkr.ecr.us-east-1.amazonaws.com/coworking:latest

8. Setup CloudWatch Logging
aws iam attach-role-policy `
    --role-name "eksctl-coworking-cluster-nodegroup-NodeInstanceRole-AKac5YZalJ9e" `
    --policy-arn "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
aws eks create-addon --addon-name amazon-cloudwatch-observability --cluster-name coworking-cluster