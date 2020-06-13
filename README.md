########################################### Cloud_task1###########################################################

# Why Cloud ?
  
Many companies have a hard time maintaining their data centres. It's also inconvenient for new startups to spend a huge amount on infrastructures. A data centre would mean buying a whole system with lots of RAM, CPU & other necessary hardware. Then, hiring some expert guys to setup the whole system & to maintain it. Security, electricity, etc. would add on to the expenditure. 

To make things easy, many companies rely on Cloud-Computing. Here, they just have to think about their work & not worry about un-necessary expenditure. Most of the Cloud Providers work on the agreement of **Pay-as-we-go**, which means that startups don't need a huge amount to setup their business.


# How to work on Clouds ?

Almost all clouds provide a nice GUI interface. However, companies don't prefer a GUI coz it can't automate things. For automation, CLI is used coz commands can easily be scheduled and hence, things can be automated.

# The Problem 

The problem here arises that different clouds have different CLI commands. Hence, it poses a problem for Cloud Engineers.

# The Solution

The solution lies in using a single method which can be used for all the clouds. One such tool is Terraform. A Terraform code is similar for all clouds and it also helps in maintaining records of what all has been done.


# The Project 

In this project, I have launched a web server using terraform code. 


**Step - 1**  First of all, configure your AWS profile in your local system using cmd. Fill your details & press Enter.

                  aws configure --profile Sparsh
                  AWS Access Key ID [****************WO3Z]:
                  AWS Secret Access Key [****************b/hJ]:
                  Default region name [ap-south-1]:
                  Default output format [None]:

            



**Step - 2**  Launch an ec2 instance using Terraform. Here, I have used a Redhat 8 AMI. I have also installed and configured Apache Web Services in the instance using the Remote Execute Provisioner. I have used a pre-created key and security group. If you wish to create a new one, you can do the same. Make sure that the security group has SSH enabled on port 22 & HTTP enabled on port 80.
The terraform code is -

          provider  "aws" {
            region   = "ap-south-1"
            profile  = "Sparsh"
          }

          resource "aws_instance" "test_ins" {
            ami             =  "ami-052c08d70def0ac62"
            instance_type   =  "t2.micro"
            key_name        =  "newk11"
            security_groups =  [ "launch-wizard-1" ]

           connection {
              type     = "ssh"
              user     = "ec2-user"
              private_key = file("C:/Users/AAAA/Downloads/newk11.pem")
              host     = aws_instance.test_ins.public_ip
            }

            provisioner "remote-exec" {
              inline = [
                "sudo yum install httpd  php git -y",
                "sudo systemctl restart httpd",
                "sudo systemctl enable httpd",
                "sudo setenforce 0"
              ]
            }

            tags = {
              Name = "my_os"
            }
          }
          
          
**Step - 2**  Create an EBS volume. Here, I have created a volume of 1 GiB. A problem that would arise here is that we don't know that our instance is launched in which availability zone. But, we need to launch our EBS volume in the same zone otherwise they can't be connected. To resolve this, I have retrieved the availability zone of the instance & used it here. 

          resource "aws_ebs_volume" "my_vol" {
            availability_zone  =  aws_instance.test_ins.availability_zone
            size               =  1

            tags = {
              Name = "my_ebs"
            }
          }
          
          
**Step - 3**  Attach your EBS volume to the instance. 

          resource "aws_volume_attachment"  "ebs_att" {
            device_name  = "/dev/sdd"
            volume_id    = "${aws_ebs_volume.my_vol.id}"
            instance_id  = "${aws_instance.test_ins.id}"
            force_detach =  true
          }
         
I have also retrieved the public ip of my instance and stored it in a file locally as it may be used later.

          resource "null_resource" "ip_store"  {
            provisioner "local-exec" {
                command = "echo  ${aws_instance.test_ins.public_ip} > public_ip.txt"
              }
          }


          
**Step - 4** Now, we need to mount our EBS volume to the folder /var/ww/html so that it can be deployed by the Apache Web Server. I have downloaded the code from Github at the same location.

        resource "null_resource" "mount"  {

        depends_on = [
            aws_volume_attachment.ebs_att,
          ]


          connection {
            type     = "ssh"
            user     = "ec2-user"
            private_key = file("C:/Users/AAAA/Downloads/newk11.pem")
            host     = aws_instance.test_ins.public_ip
          }

        provisioner "remote-exec" {
            inline = [
              "sudo mkfs.ext4  /dev/xvdd",
              "sudo mount  /dev/xvdd  /var/www/html",
              "sudo rm -rf /var/www/html/*",
              "sudo git clone https://github.com/sparshpnd23/Cloud_task1.git /var/www/html/"
            ]
          }
        }


I have also downloaded all the code & images from Github in my local system, so that I can automate the upload of images in s3 later.

        resource "null_resource" "git_copy"  {
          provisioner "local-exec" {
            command = "git clone https://github.com/sparshpnd23/Cloud_task1.git C:/Users/AAAA/Pictures/" 
            }
        }
        
   
   
**Step - 5**  Now, we create an S3 bucket on AWS. The code snippet for doing the same is as follows -

          resource "aws_s3_bucket" "sp_bucket" {
            bucket = "sparsh23"
            acl    = "private"

            tags = {
              Name        = "sparsh2301"
            }
          }
          
          
**Step - 6**  Now that the S3 bucket has been created, we will upload the images that we had downloaded from Github in our local system in the above step. Here, I have uploaded just one pic. You can upload more if you wish.

            resource "aws_s3_bucket_object" "object" {
              bucket = "${aws_s3_bucket.sp_bucket.id}"
              key    = "test_pic"
              source = "C:/Users/AAAA/Pictures/pic1.jpg"
              acl    = "public-read"
            }
