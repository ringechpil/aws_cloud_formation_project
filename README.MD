### PRELIMINARY README (work in progress...)

## Project Flow:

We use CloudFormation to create and deploy an infrastructure in the cloud.


1. We first deploy a VPC containing:

 - 2 Public Subnets
 - 2 Private Subnets
 - Internet Gateway
 - Appropriate Route Tables
 - Route Table Associations 

To deploy this from the `.yaml` file, use:

`aws cloudformation deploy --stack-name cloud-stack --template-file .\formation_template.yaml`

We can always check the deployed resources with:

aws cloudformation describe-stack-resources \
--stack-name myStack \
--query 'StackResources[*].[ResourceType,LogicalResourceID,ResourceStatus]' \
--output table

We can always delete the deployed stack with:

`aws cloudformation delete-stack --stack-name cloud-stack`


2. We deploy and configure Security Groups with appropriate inbound/outbound rules.

3. We deploy a Web Server EC2 instance in the Public Subnet, with the appropriate Security Group that allows SSH connection only via personal local machine, but also can be accessed via HTTP.

From now on, we deploy the stack from `.yaml` using:

`aws cloudformation deploy --stack-name cloud-stack --template-file .\formation_template.yaml --parameter-overrides MyIpAddress=<yourIP>`

To get info on the Web Server EC2 instance, use

`aws ec2 describe-instances --filters "Name=tag:aws:cloudformation:stack-name,Values=cloud-stack" --query "Reservations[*].Instances[*].{Name:Tags[?Key=='Name']|[0].Value,Status:State.Name,PublicIp:PublicIpAddress,PrivateIp:PrivateIpAddress,Subnet:SubnetId,SecurityGroup:SecurityGroups[0].GroupName,AMI:ImageId,InstanceType:InstanceType}" --output table`


We then connect to this instace. We connect via terminal on the local machine by using

`ssh -i .\labsuser.pem ec2-user@<instanceIP>`

(where we need to get appropriate `.pem` file beforehand)

4. We install and configure LAMP stack on the Web Server EC2 instance by running the following commands:

- System update

sudo dnf upgrade -y

- Install LAMP stack

sudo dnf install -y httpd wget php php-devel php-fpm php-mysqlnd php-mysqli php-json php-gd php-mbstring php-xml

sudo dnf install -y mariadb105-server

- Start and enable services

sudo systemctl start httpd

sudo systemctl enable httpd

sudo systemctl start mariadb

sudo systemctl enable mariadb

- Secure MariaDB (interactive)

sudo mysql_secure_installation

- Create database and user

sudo mysql -u root -p <<EOF

CREATE DATABASE \`wordpress_db\`;

CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'YourStrongPassword123!';

GRANT ALL PRIVILEGES ON \`wordpress_db\`.* TO 'wp_user'@'localhost';

FLUSH PRIVILEGES;

EOF

- Download and extract WordPress

cd ~

wget https://wordpress.org/latest.tar.gz

tar -xzf latest.tar.gz

- Configure WordPress

cp wordpress/wp-config-sample.php wordpress/wp-config.php

- Edit wp-config.php with your DB name, user, password, and salt keys

nano wordpress/wp-config.php

- Deploy to web root

sudo cp -r wordpress/* /var/www/html/

- Set permissions

sudo chown -R apache:apache /var/www/html/

sudo chmod 2775 /var/www

find /var/www -type d -exec sudo chmod 2775 {} \;

find /var/www -type f -exec sudo chmod 0644 {} \;

- Enable .htaccess

sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

- Restart Apache

sudo systemctl restart httpd

- Cleanup

rm -f ~/latest.tar.gz

rm -rf ~/wordpress/

5. After this is complete, we conncect to the Web Server instance via HTTP using our Web Browser (`http:\\<instanceIP>`).
There we register to WordPress, and make a Webpage or small Website. 

