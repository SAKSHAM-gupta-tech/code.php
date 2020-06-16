#creating private and public key pair
resource "tls_private_key" "my_key" {
  algorithm   = "RSA"
  rsa_bits = 4096
}

#creating private key
resource "local_file" "private_key" {
 depends_on = [tls_private_key.my_key]
 content = tls_private_key.my_key.private_key_pem
 filename = "keyforterraform.pem"
 file_permission = 0400
}

#creating public key
resource "aws_key_pair" "webserver_key"{
  depends_on = [local_file.private_key]
  key_name = "keyforterraform.pem"
  public_key = tls_private_key.my_key.public_key_openssh
}


#creating security group
resource "aws_security_group" "sg2" {
  name        = "security_02"
  description = "https,ssh"

  ingress {
    description = "ssh"
    from_port   = 0
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  

  ingress {
    description = "http"
    from_port   = 0
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "security_02"
  }
}



#creating instance
resource "aws_instance" "web" {
  depends_on  = [aws_cloudfront_distribution.cloud_distribution]
  ami           = "ami-052c08d70def0ac62"
  instance_type = "t2.micro"
  key_name          = aws_key_pair.webserver_key.key_name
  security_groups = ["security_02"]
  
    connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = tls_private_key.my_key.private_key_pem
    host     = aws_instance.web.public_ip
  }
     provisioner "remote-exec" {
     inline = [
      "sudo yum install httpd php git -y",
      "sudo systemctl start httpd",
      "sudo systemctl enable httpd"
   ]
  }

  tags = {
    Name = "Helloinstance"
  }
}




#creating volume
resource "aws_ebs_volume" "volume01" {
  depends_on = [aws_instance.web]
  availability_zone = aws_instance.web.availability_zone
  size              = 1

  tags = {
    Name = "hardisk"
  }
}




#attaching volume
resource "aws_volume_attachment" "ebs_att" {
  depends_on = [aws_ebs_volume.volume01]
  device_name = "/dev/sdh"
  volume_id   = aws_ebs_volume.volume01.id
  instance_id = aws_instance.web.id
  force_detach= true
}


resource "null_resource" "mounting01" {
  depends_on = [
    aws_volume_attachment.ebs_att,
  ]
  

   connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = tls_private_key.my_key.private_key_pem
    host     = aws_instance.web.public_ip
  }

    provisioner "remote-exec" {
     inline = [
      "sudo mkfs.ext4 /dev/xvdh",
      "sudo mount /dev/xvdh /var/www/html",
      "sudo rm -rf /var/www/html/*",
      "sudo git clone https://github.com/SAKSHAM-gupta-tech/index.git /var/www/html"
     ]
  } 
} 


#creating s3 bucket
resource "aws_s3_bucket" "myproject_01" {
  depends_on = [aws_security_group.sg2]
  acl    = "private"
  region = "ap-south-1"
  bucket = "mysecond-bucket"
}


#adding objects in s3 bucket
resource "aws_s3_bucket_object" "object" {
  depends_on = [aws_security_group.sg2]
  bucket = aws_s3_bucket.myproject_01.bucket
  key    = "pubg.jpg"
  acl    = "private"
  source = ("C:/Users/munesh/Desktop/pubg.jpg")
  etag   = filemd5("C:/Users/munesh/Desktop/pubg.jpg")
}
resource "aws_s3_bucket_public_access_block" "public_storage"{
 depends_on = [aws_s3_bucket.myproject_01]
 bucket = aws_s3_bucket.myproject_01.bucket
 block_public_acls = false
 block_public_policy = false
}


#creating cloudfront
resource"aws_cloudfront_distribution"  "cloud_distribution"{
   origin {
     domain_name = aws_s3_bucket.myproject_01.bucket_domain_name
     origin_id   = "s3-mysecondbucket"
    }

  enabled = true
  viewer_certificate {
   cloudfront_default_certificate = true
   }

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-mysecondbucket"

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "allow-all"
    
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
      
    }
  }
  origin {
    domain_name = aws_s3_bucket.myproject_01.bucket_domain_name
    origin_id   = "s3-mysecondbucket"
    }

  tags = {
    Environment = "production"
  }
}


#storing the result 
resource "null_resource" "storing_ip_file"{
     depends_on = [null_resource.mounting01]
     provisioner "local-exec"{
         command = "echo the webs successful and >> result.txt &&echo the ip is ${aws_instance.web.public_ip}>>result.txt"
  }
}


resource "null_resource" "website"{
     depends_on = [null_resource.storing_ip_file]
     provisioner "local-exec"{
         command = "echo chrome ${aws_instance.web.public_ip}"
  }
}


 
