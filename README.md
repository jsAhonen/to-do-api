# To Do App API

This is the API codebase of the To Do App.

## Tech Stack

- Python (see `.python-version` for the current version in use)
- Django and Django REST Framework
- Pytest
- AWS Elastic Container Registry (ECR)
- AWS Elastic Container Service (ECS)
- AWS Fargate
- AWS CloudFormation
- AWS Application Load Balancer (EC2)
- AWS CloudFront

## How to set up the local development environment

There are two options for setting up and running the development environment locally:
- Python virtual environment (venv)
- Docker container

### Python virtual environment

Install Python (check the version in `.python-version`)

Go to `to-do-api/todo_api/` and create the virtual environment, open it and install dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
`.venv` could be substituted with any path, if you wish to create the virual env elsewhere

### Docker container

You need to have Docker installed so that you can run `docker` commands.

Go to `to-do-api/` directory, where the `Dockerfile` is, build the Docker image, create and run the container, and run the database migrations, with these commands:

```bash
docker build -t todo-api .
docker run -d --name todo-api -p 8000:8000 todo-api
docker exec -it todo-api python manage.py migrate
```

It's also recommended to create the Django super user for accessing the admin dashboard:

```bash
docker exec -it todo-api python manage.py createsuperuser
```

You will be asked to give a username, email (optional) and password (twice). After creating the super user, you can access the Django admin dashboard in `https://localhost:8000/admin/login` .

All commands you need to run inside the container should be prefixed with:

```bash
docker exec -it todo-api
```

and you can enter inside the container with this command:

```bash
docker exec -it todo-api bash
```

and exit by running `exit`.

## How to run the development environment

If you use VSCode, open a new terminal, and virtual environment should open automatically.

Run this command to start the development server:

```bash
python manage.py runserver
```

To use a regular terminal, go to the root directory of this project, open the virtual environment and start the server:

```bash
source .venv/bin/activate
python manage.py runserver
```

## Tests

The project uses Pytest unit tests.

To run the tests, run the following command:

```bash
pytest
```

## Database

The project uses a SQLite3 database.

To open it, you need to install SQLite3: see [SQLite homepage](https://sqlite.org/)

Linux installation:

```bash
sudo apt install sqlite3
```

If you have SQLite3 installed, run the dbshell command in the virtual env:

```bash
python manage.py dbshell
```

Some commands to navigate the DB shell:

```bash
.tables # view the list of tables
.help # view the help file
```

### Changing the data model

If you make changes to the data models, you must run a database migration to update the database according to those changes:

```bash
python manage.py makemigrations # create the migration files based on the code changes
python manage.py migrate # migrate the database to the latest migrations
```

You can view the current state of the migrations by running this:

```bash
python manage.py showmigrations
```

## AWS Deployment

### Setup

#### Prepare AWS Environment

Install AWS CLI v2

Create IAM user `main-admin` with the following permissions policies:

- AdministratorAccess

Create an access key and a secret access key for `main-admin`.

Configure credentials and set the keys and choose a default region (e.g. us-east-1) to the AWS CLI configuration:

```bash
aws configure
```

#### Build & Push Docker Image

Replace the `<region>` and `<account_id>` with the region of your choice and the account id of the AWS account id.

```bash
aws ecr create-repository --repository-name todo-api
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com
docker build -t todo-api .
docker tag todo-api:latest <account_id>.dkr.ecr.<region>.amazonaws.com/todo-api:latest
docker push <account_id>.dkr.ecr.<region>.amazonaws.com/todo-api:latest
```

#### Deploy the CloudFormation Stacks

First you need to configure the IaC and deploy for the Application Load Balancer (ALB) and then configure and deploy the API.

Create a copy of `iac-alb.yaml.template` and name it as `iac-alb.yaml`.

Replace the placeholder values following the instructions in the comment at the beginning of each file.

Deploy the Application Load Balancer with CloudFormation:

```bash
aws cloudformation deploy \
  --profile main-admin \
  --template-file iac-alb.yaml \
  --stack-name todo-api-stack \
  --capabilities CAPABILITY_NAMED_IAM
```

Go to EC2 target groups, and look up the `todo-api-tg` ARN to use it for `iac.yaml`

Do the same for `iac.yaml.template` as you did for `iac-alb.yaml.template`.

Deploy the API with CloudFormation:

```bash
aws cloudformation deploy \
  --profile main-admin \
  --template-file iac.yaml \
  --stack-name todo-api-stack \
  --capabilities CAPABILITY_NAMED_IAM
```

#### Run Database Migrations

Look up the EC2 security group named `todo-api-sg` and a subnet id and insert them in the placeholders `<sg_id>` and `<subnet_id>` below, and then run the command:

```bash
aws ecs run-task \
  --cluster todo-api-cluster \
  --launch-type FARGATE \
  --task-definition todo-api-task \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet_id>],securityGroups=[<sg_id>],assignPublicIp=ENABLED}" \
  --overrides '{"containerOverrides":[{"name":"todo-api","command":["python","manage.py","migrate"]}]}'
```

#### Access the App

The CloudFront has a domain with the `<DNS_NAME>` by which you can access the API

```code
https://<DNS_NAME>/api
```

The admin UI is here

```code
https://<DNS_NAME>/api/admin/
```

### Update

Once the project is set up, this is how you update it.

```bash
# Update the Docker Image
docker build -t todo-api .
docker tag todo-api:latest 937253175635.dkr.ecr.us-east-1.amazonaws.com/todo-api:latest
docker push 937253175635.dkr.ecr.us-east-1.amazonaws.com/todo-api:latest

# Update the service
aws ecs update-service \
  --profile main-admin \
  --cluster todo-api-cluster \
  --service todo-api-service \
  --force-new-deployment
```

### Teardown

```bash
# Delete the CloudFormation Stacks
aws cloudformation delete-stack --profile main-admin --stack-name todo-api-stack
aws cloudformation delete-stack --profile main-admin --stack-name todo-alb-stack

# Delete ECR repo
aws ecr delete-repository --profile main-admin --repository todo-api --force
```
