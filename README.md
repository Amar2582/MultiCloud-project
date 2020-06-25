# MultiCloud-project

![github,aws cloud and terraform img](https://user-images.githubusercontent.com/64667387/85639629-ecafec00-b6a6-11ea-91b0-68c36064ac4c.png)

## I am going to launch/create web-Applicatin infrastructure on AWS with Terraform.

so,First of all we know about AWS cloud and Terraform

- AWS cloud infrastructure is designed to be the most flexible and secured public cloud network. It provides scalable and highly reliable platform that enables customers to deploy applications and data quickly and securely.

- Terraform is an open-source infrastructure as code software tool created by HashiCorp. It enables users to define and provision a datacenter infrastructure using a high-level configuration language known as Hashicorp Configuration Language, or optionally JSON.

- Terraform by HashiCorp, is an “infrastructure as code” tool similar to AWS CloudFormation that allows you to create, update, and version your Amazon Web Services (AWS) infrastructure.


+ NOW I Have to create/launch Application using Terraform


 1. Create the key and security group which allow the port 80.

 2. Launch EC2 instance.

 3. In this Ec2 instance use the key and security group which we have created in step 1.

 4. Launch one Volume (EBS) and mount that volume into /var/www/html

 5. Developer have uploded the code into github repo also the repo has some images.

 6. Copy the github repo code into /var/www/html

 7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.

 8. Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to update in code in /var/www/html

- Step with code to complete this Task1
   - Create an IAM user in your AWS account and then configure it by using the command.
   
```aws configure --profile IAM_username
C:\Users\amar>aws configure --profile amar
AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]:
Default output format [None]:
```

- Step to solve the given Task

## 1. verify the provider as aws with profile and region
```
provider "aws" {
        region  = "ap-south-1"
        profile = "amar"
}
```  
## 2. Create the key pair and save it so as to use for the instance login
```
resource "tls_private_key" "task1_key" {
 algorithm  = "RSA"
}


resource "local_file" "mykey_file" {
  content     = tls_private_key.task1_key.private_key_pem
  filename    = "mykey.pem"
}


resource "aws_key_pair" "mygenerate_key" {
 key_name  = "mykey"
 public_key = tls_private_key.task1_key.public_key_openssh

}
```
![Screenshot (183)](https://user-images.githubusercontent.com/64667387/85637253-54af0400-b6a0-11ea-8bad-d770d7798eaf.png)

## 3. create a security group allowing port 22 for ssh login and allowing port 80 for HTTP protocol.

```
resource "aws_security_group" "task1_securitygroup" {
  name        = "task1_securitygroup"
  description = "allow ssh and http traffic"


  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }




  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
  }
}
```

![Screenshot (182)](https://user-images.githubusercontent.com/64667387/85637667-6d6be980-b6a1-11ea-96ec-a6068c27fd03.png)


## 4.In this Ec2 instance use the key and security group which we have created with automatic login into the instance and download the httpd and git.

```
variable "ami_id" {
  default = "ami-052c08d70def0ac62"
}
resource "aws_instance" "myos" {
  ami             = var.ami_id
  instance_type   = "t2.micro"
  key_name        = aws_key_pair.mygenerate_key.key_name
  security_groups = [aws_security_group.task1_securitygroup.name]
  vpc_security_group_ids  = [aws_security_group.task1_securitygroup.id]
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = tls_private_key.task1_key.private_key_pem
    port        = 22
    host        = aws_instance.myos.public_ip
}
  
  provisioner "remote-exec" {
       inline = [
              "sudo yum install httpd -y",
              "sudo systemctl start httpd",
              "sudo systemctl enable httpd",
              "sudo yum install git -y"
]
}
  tags = {
    Name = "task1 myos"
  }
}
```
![Screenshot (180)](https://user-images.githubusercontent.com/64667387/85638075-9476eb00-b6a2-11ea-8864-a037967e0c1a.png)

## 5.Create EBS volume

```
resource "aws_ebs_volume" "myvolume" {
  depends_on = [
         
         aws_instance.myos
     ]
     availability_zone = aws_instance.myos.availability_zone
  size              = 1


  tags = {
    Name = "ebsvol"
  }
}
```
![Screenshot (181)](https://user-images.githubusercontent.com/64667387/85638409-78277e00-b6a3-11ea-8a36-b2d4f05801ec.png)


## 6.Attaching created volume to existing ec2 instance

```
resource "aws_volume_attachment" "ebs_att" {
  device_name = "/dev/sdh"
  volume_id   = "${aws_ebs_volume.myvolume.id}"
  
  instance_id = "${aws_instance.myos.id}"
  force_detach = true
}
```
## 7. Mount the volume
```
resource "null_resource" "partition_and_mount"  {


depends_on = [
    aws_volume_attachment.ebs_att
  ]

 connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = tls_private_key.task1_key.private_key_pem
    host        = aws_instance.myos.public_ip
  }


provisioner "remote-exec" {
    inline = [
      "sudo mkfs.ext4  /dev/sdh",
      "sudo mount  /dev/sdh  /var/www/html",
      "sudo rm -rf /var/www/html/*",
      "sudo git clone https://github.com/Amar2582/MultiCloud-project.git /var/www/html/",
      "sudo setenforce 0"
    ]
  }
}
```

## 8.Creating a s3 bucket and make it public readable

```
resource "aws_s3_bucket" "mybucket1" {

  bucket = "amar2582"
  acl    = "public-read"


  tags = {
	Name = "taskbucket"
  }

}

locals {

s3_origin_id = "myS3Origin"
}
```
![Screenshot (184)](https://user-images.githubusercontent.com/64667387/85638733-4f53b880-b6a4-11ea-9bee-904d2ffd9035.png)

## 9.upload the image in this s3 bucket and give name is 'SQUAD'.

```
resource "aws_s3_bucket_object" "Squd" {
 depends_on = [
    aws_s3_bucket.mybucket1,
  ]
  bucket       =  "amar2582"
  key          = "Squd.jpg"
  source       = "C:/Users/AMAR/Pictures/squd.jpg"
  acl          = "public-read"
  content_type = "image or jpeg"
}
```
![Screenshot (191)](https://user-images.githubusercontent.com/64667387/85638939-e7ea3880-b6a4-11ea-8c56-c521c9206373.png)
![Screenshot (186)](https://user-images.githubusercontent.com/64667387/85638991-0cdeab80-b6a5-11ea-9ad2-e853fbe593ff.png)


## 10.creating CloudFront with S3 as origin to provide CDN(content delevery network)

```
resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name = aws_s3_bucket.mybucket1.bucket_regional_domain_name
    origin_id   = local.s3_origin_id

  }


  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Some comment"
  default_root_object = "index.php"


  logging_config {
    include_cookies = false
    bucket          = "amar2582.s3.amazonaws.com"
    prefix          = "myprefix"
  }


  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = local.s3_origin_id


    forwarded_values {
      query_string = false


      cookies {
        forward = "none"
      }
    }


    viewer_protocol_policy = "allow-all"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  ordered_cache_behavior {
    path_pattern     = "/content/immutable/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = local.s3_origin_id


    forwarded_values {
      query_string = false
      headers      = ["Origin"]


      cookies {
        forward = "none"
      }
    }


    min_ttl                = 0
    default_ttl            = 86400
    max_ttl                = 31536000
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }

  ordered_cache_behavior {
    path_pattern     = "/content/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = local.s3_origin_id


    forwarded_values {
      query_string = false


      cookies {
        forward = "none"
      }
    }


    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
  }


  price_class = "PriceClass_200"


  restrictions {
    geo_restriction {
      restriction_type = "none"
      
    }
  }


  tags = {
    Environment = "production"
  }


  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```
![Screenshot (188)](https://user-images.githubusercontent.com/64667387/85639232-ac9c3980-b6a5-11ea-975e-5dd10b52ef66.png)

## 11.Save the file and run the following command

```
terraform init                    
terraform validate                
terraform apply -auto-approve
```

# Finally we get own desire web server

![Screenshot (189)](https://user-images.githubusercontent.com/64667387/85639465-76ab8500-b6a6-11ea-8d33-91609c6656e3.png)




                                              # THAT`s ALL
                                              
                                              
                                              # THANK 
                                                      YOU




