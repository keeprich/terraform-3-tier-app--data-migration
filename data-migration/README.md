
Migrating Oracle to Postgresql with AWS DMS and Terraform

Todd BernsonClick here to view Todd Bernson’s profile
Todd Bernson
CTO at BSC | Multi cloud expert | Awarded #1 AWS Ambassador North America 2022
Published Jan 19, 2022
+ Follow
Purpose:

We will use Terraform and AWS to migrate for Oracle to Aurora Postgresql Serverless with Infrastructure as Code (IaC). In this case we are using Terraform from HashiCorp as it is open source, cross platform compatible and also an industry standard that can handle cloud platforms as well as on premise infrastructure through an extensive set of providers. We can remove licensing and take advantage of some of the cloud built solutions Aurora offers.

What services will we be using?

IaC is the industry standard in all cloud environments. Repeatability, disaster recovery, and standardization are a few of the benefits here along with enforcement of a peer review process before changes are committed to the master branch. This, along with a complete historical record of any changes to the environment in the source controlled repo, ensures logical, reviewed changes and easier audits if the environment may be subject to a regulatory framework.

RDS (Relational Database Service) is where we are hosting our Oracle database. RDS allows for easy, managed setup of a relational database (in our case - Oracle) that can be set up in a highly available way.

Aurora is open source (MySQL or Postgresql) compatible relational database that was built for the cloud. Storage, networking, and clustering allow for global clusters with incredibly small latency.

DMS (Database Migration Service) is AWS' managed service to migrate from one place to another, and/or from one database platform to another. All mappings and jobs are handled on managed instances and can be kept up to date with CDC until cutover is desired.

Let's build it!

Source database is Oracle RDS built in private database subnets behind a NAT.

module "oracle" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 3.0"

  identifier = "${var.environment}-oracle"

  engine               = "oracle-se2"
  engine_version       = "19.0.0.0.ru-2021-10.rur-2021-10.r1"
  family               = "oracle-se2-19"
  major_engine_version = "19"
  instance_class       = "db.t3.medium"
  license_model        = "license-included"

  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true

  name                   = var.database_name
  username               = "admin"
  create_random_password = true
  random_password_length = 16
  port                   = 1521

  multi_az               = false
  subnet_ids             = var.database_subnets
  vpc_security_group_ids = [aws_security_group.oracle.id]

  backup_retention_period = 0
  skip_final_snapshot     = true
  deletion_protection     = false

  character_set_name = "AL32UTF8"

  tags = var.tags
}
We've also added in an AWS secret that holds our credentials and a security group allowing least privilege.

resource "aws_secretsmanager_secret_version" "oracle" {
  secret_id = aws_secretsmanager_secret.oracle.id
  secret_string = jsonencode(
    {
      username = module.oracle.db_instance_username
      password = module.oracle.db_instance_password
    }
  )

  depends_on = [module.oracle]
}
Target database is RDS Postgresql Aurora running serverless for even more cost savings built in private database subnets behind a NAT.

module "postgresql" {
  source  = "terraform-aws-modules/rds-aurora/aws"
  version = "~> 6.0"

  allowed_cidr_blocks    = [data.aws_vpc.this.cidr_block]
  apply_immediately      = true
  copy_tags_to_snapshot  = true
  create_monitoring_role = true
  database_name          = var.database_name
  db_subnet_group_name   = "${var.environment}_postgresql"
  deletion_protection    = false
  engine                 = "aurora-postgresql"
  engine_mode            = "serverless"
  engine_version         = "10.12"
  monitoring_interval    = 10
  name                   = "${var.environment}-postgresql"
  storage_encrypted      = true
  subnets                = var.database_subnets
  tags                   = var.tags
  vpc_id                 = data.aws_vpc.this.id

  scaling_configuration = {
    auto_pause               = true
    max_capacity             = 8
    min_capacity             = 2
    seconds_until_auto_pause = 600
    timeout_action           = "ForceApplyCapacityChange"
  }
}
Again, we've also added in an AWS secret that holds our credentials and a security group allowing least privilege.

Now comes the fun part! Saving money and time while increasing our resilience and availability through migration.

DMS has a few resources that we will need to set up.

IAM - Several roles will need to be created and referenced.



data "aws_iam_policy_document" "dms_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      identifiers = ["dms.amazonaws.com"]
      type        = "Service"
    }
  }
}

resource "aws_iam_role" "dms-access-for-endpoint" {
  assume_role_policy = data.aws_iam_policy_document.dms_assume_role.json
  name               = "dms-access-for-endpoint"
}

resource "aws_iam_role_policy_attachment" "dms-access-for-endpoint-AmazonDMSRedshiftS3Role" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonDMSRedshiftS3Role"
  role       = aws_iam_role.dms-access-for-endpoint.name
}

resource "aws_iam_role" "dms-cloudwatch-logs-role" {
  assume_role_policy = data.aws_iam_policy_document.dms_assume_role.json
  name               = "dms-cloudwatch-logs-role"
}

resource "aws_iam_role_policy_attachment" "dms-cloudwatch-logs-role-AmazonDMSCloudWatchLogsRole" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonDMSCloudWatchLogsRole"
  role       = aws_iam_role.dms-cloudwatch-logs-role.name
}

resource "aws_iam_role" "dms-vpc-role" {
  assume_role_policy = data.aws_iam_policy_document.dms_assume_role.json
  name               = "dms-vpc-role"
}

resource "aws_iam_role_policy_attachment" "dms-vpc-role-AmazonDMSVPCManagementRole" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole"
  role       = aws_iam_role.dms-vpc-role.name
}
Then the DMS resources of

source endpoint
target endpoint
DMS subnet group
DMS replication instance
DMS task(s)
DMS security group
Giving a few examples below.

resource "aws_dms_endpoint" "source" {
  database_name = var.database_name
  endpoint_id   = "oracle"
  endpoint_type = "source"
  engine_name   = "oracle"
  password      = var.oracle_password
  username      = var.oracle_username
  port          = var.oracle_port
  server_name   = var.oracle_server_name
  tags          = var.tags
}

resource "aws_dms_replication_instance" "this" {
  allocated_storage            = 500
  apply_immediately            = true
  auto_minor_version_upgrade   = true
  engine_version               = "3.4.6"
  multi_az                     = false
  preferred_maintenance_window = "sun:10:30-sun:14:30"
  publicly_accessible          = false
  replication_instance_class   = "dms.t3.medium"
  replication_instance_id      = var.environment
  replication_subnet_group_id  = aws_dms_replication_subnet_group.this.id
  vpc_security_group_ids       = [aws_security_group.dms.id]
  tags                         = var.tags
}

resource "aws_dms_replication_task" "this" {
  migration_type           = "full-load"
  replication_instance_arn = aws_dms_replication_instance.this.replication_instance_arn
  replication_task_id      = var.environment
  source_endpoint_arn      = aws_dms_endpoint.source.endpoint_arn
  table_mappings           = file("${path.module}/files/mapping_rules.json")
  target_endpoint_arn      = aws_dms_endpoint.target.endpoint_arn
  tags                     = var.tags

  replication_task_settings = jsonencode(SETTINGS_HERE)
}
We initialize, plan, and apply our Terraform scripts

No alt text provided for this image
We are all set for our infrastructure!

In our Terraform, we have added an S3 bucket, access to that bucket on our bastion host, and several important files that are downloaded with our userdata script. I've included connection strings, table seeding, and row insertions to populate our oracle database.



create table person (
        person_id int primary key,
        person_first_name varchar2(100),
        person_last_name  varchar2(100),
        state             char(2)
);


insert into person (person_id, person_first_name, person_last_name, state)
values (1, 'John','Doe','DC');
insert into person (person_id, person_first_name, person_last_name, state)
values (2, 'Mary','Doe','VA');
insert into person (person_id, person_first_name, person_last_name, state)
values (3, 'Todd','Doe','SC');
insert into person (person_id, person_first_name, person_last_name, state)
values (4, 'Lee','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (5, 'Kenneth','Doe','FL');
insert into person (person_id, person_first_name, person_last_name, state)
values (6, 'Jeff','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (7, 'Will','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (8, 'Sam','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (9, 'Alexis','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (10, 'Michael','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (11, 'Joseph','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (12, 'Kai','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (13, 'Malia','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (14, 'Jolie','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (15, 'Robin','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (16, 'Jonathan','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (17, 'Max','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (18, 'Hiren','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (19, 'Nate','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
values (20, 'James','Doe','NC');
insert into person (person_id, person_first_name, person_last_name, state)
We can test this on our Oracle RDS instance.

No alt text provided for this image
We are ready to check out DMS. The first thing we need to do is confirm our endpoints have access to our databases.

No alt text provided for this image
No alt text provided for this image
No alt text provided for this image
We have now confirmed that we have a working source and target endpoint for the DMS task to work with.

We will kick off the task and wait for the load, transformation, and migration.

No alt text provided for this image
Looks like it worked. Let's check it out by querying the Aurora database.

No alt text provided for this image
Great! Confirmation!

Conclusion:

We have successfully built our entire infrastructure down to the objects in the S3 buckets with IaC. We have successfully migrated an Oracle database to Amazon Aurora Postgresql serverless.