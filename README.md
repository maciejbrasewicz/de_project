# de_project

## 1.   Design

  Airflow was used to orchestrate the following tasks:

 - Classifying movie reviews with Apache Spark.
 - Loading classified movie reviews into the data warehouse.
 -   Extracting user purchase data from an OLTP database and loading it into a data warehouse.
 - Joining the classified movie review data and user purchase data to get  `user behavior metric`  data.
 - And metabase to visualize our data.

 ![Screenshot from 2022-12-21 19-09-07_edit_150679334669713](https://user-images.githubusercontent.com/49028274/208976026-1331dc5e-01a4-4ca8-bfbc-85a9d551cd93.png)

## 2. Setup
### 2.1 Pre-requisites

 -  [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
 -  [AWS account](https://aws.amazon.com/)
 -  [AWS CLI installed](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)  
 -  [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
 -  [Docker](https://docs.docker.com/engine/install/)  and  [Docker Compose plugin](https://docs.docker.com/compose/install/)  
 - > **Docker compose v2:** The original python project, called `docker-compose`, aka v1 of docker/compose repo, has now been deprecated and development has moved over to v2. To install the v2 `docker compose` as a CLI plugin on Linux, supported distribution can now install the `docker-compose-plugin` package. E.g. on debian, I run `apt-get install docker-compose-plugin`. 

- > **AWS Configuration and credential file settings:** Read more at [ credential file settings](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) & [configuration settings and precendence](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-precedence)

### 2.2 Start
### Setup your project locally and on the cloud.

```shell
# Clone the code as shown below.
https://github.com/maciejbrasewicz/de_project.git
cd de_project
```
**Replace content in the following files:**

1.  **[CODEOWNERS](https://github.com/maciejbrasewicz/de_project/blob/main/.github/CODEOWNERS)**: change the user id from  `@maciejbrasewicz`  to your GitHub user id.
2.  **[cd.yml](https://github.com/maciejbrasewicz/de_project/blob/main/.github/workflows/cd.yml)**: In this file change the  `de_project`  part of the  `TARGET` to your repository name. If you haven't changed the name, leave it like that
3. -   **[main.tf](https://github.com/maciejbrasewicz/de_project/blob/main/terraform/main.tf)**: make sure `Clone git repo to EC2` param is customized for your needs (line 248).

**[main.tf](https://github.com/maciejbrasewicz/de_project/blob/main/terraform/main.tf)** defines all the services we need. In this file we create an EC2 instance, security group where we configure access and a cost alert. For instance, the security group for access to EC2: 

```shell
# Create security group for access to EC2 from your Anywhere
resource "aws_security_group" "sde_security_group" {
  name        = "sde_security_group"
  description = "Security group to allow inbound SCP & outbound 8080 (Airflow) connections"

  ingress {
    description = "Inbound SCP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
...
```


The following commands will allow us to configure the project:

```shell
make up # docker containers & runs migrations under ./migrations
make ci # Runs auto formatting, lint checks, & all the test files under ./tests

# Create AWS services with Terraform
make tf-init
make infra-up # type in yes after verifying the changes TF will make or `terraform apply -auto-approve` to avoid interactive prompt in the future.
```
Wait until the EC2 instance is initialized, you can check this via your AWS UI
See "Status Check" on the EC2 console, it should be "2/2 checks passed" before proceeding. 
  
In the main.tf file we have created security group for access to EC2 . Now, just create an inbound rule for traffic on `port 3000`  to open Metabase at `http://your_ec2_public_dns:3000`. You can customize it to accept traffic from a particular IP, a particular IP range or Open to public. 

```bash
# Create Redshift Spectrum tables (tables with data in S3)
make spectrum-migration
# Create Redshift tables
make redshift-migration
```

```bash
make cloud-airflow # this command will allow you to open the airflow at `http://your_ec2_public_dns:3000`
```

```bash
make cloud-metabase # this command will allow you to open the metabase at `http://your_ec2_public_dns:3000`
```

To get Redshift connection credentials for `metabase` use the following commands:

```bash
make infra-config
# use redshift_dns_name as host
# use redshift_user & redshift_password
# dev as database
```
Create database migrations as shown below.

```bash
make db-migration # enter a description, e.g., create some schema
make warehouse-migration # to run the new migration on your warehouse
```
## 3.  [Continuous delivery:](https://github.com/maciejbrasewicz/de_project/blob/main/.github/workflows/cd.yml)  

Set up the infrastructure with terraform, & defined the following repository secrets. You can set up the [repository secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) by going to  `Settings > Secrets > Actions > New repository secret`.


1.  **`SERVER_SSH_KEY`**: We can get this by running  `terraform -chdir=./terraform output -raw private_key`  in the project directory and paste the entire content in a new Action secret called SERVER_SSH_KEY.
2.  **`REMOTE_HOST`**: Get this by running  `terraform -chdir=./terraform output -raw ec2_public_dns`  in the project directory.
3.  **`REMOTE_USER`**: The value for this is  **ubuntu**.

## 4. Tear down infra

Make sure to destroy your cloud infrastructure.

```shell
make down # Stop docker containers on your computer
make infra-down # type in yes after verifying the changes TF will make
```


Readme in progress. I will post details on [my blog](macbrasewicz.io)
