# Trouble Shooting 및 개선점

## 1. Trouble Shooting

### 새 EC2 인스턴스의 퍼블릭 주소 접근

<img src="https://github.com/rlatkd/terraform/blob/main/assets/disconnect.jpg">

#### 인바운드 8080 port를 허용하는 보안 그룹을 추가

```
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {
  ami                    = "LINUX AMI IMAGE"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.webserversg.id]
  user_data              = <<-EOF
    #!/bin/bash
    echo "Hello, World" > index.html
    nohup busybox httpd -f -p 8080 &
  EOF

  user_data_replace_on_change = true

  tags = {
    Name = "terraform-example"
  }
}

resource "aws_security_group" "webserversg" {
  name = "terraform-example-webserversg"

# 이 부분 추가 #
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
################
```

### 그래도 접속이 안됨

<img src="https://github.com/rlatkd/terraform/blob/main/assets/disconnect.jpg">

```
...
...
user_data              = <<-EOF
    #!/bin/bash
    echo "Hello, World" > index.html
    nohup busybox httpd -f -p 8080 &
  EOF
...
...
```

- 이 부분의 `nohup`이 Amazon Linux 에서 다이렉트로 사용되려면 key를 통해 인증하거나 머신 내부에서 다운로드 받아야함

#### AMI Image를 Ubuntu로 교체

```
provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_instance" "example" {

# 이 부분 수정 #
  ami                    = "ami-086cae3329a3f7d75"
################

  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.webserversg.id]
  user_data              = <<-EOF
    #!/bin/bash
    echo "Hello, World" > index.html
    nohup busybox httpd -f -p 8080 &
  EOF

  user_data_replace_on_change = true

  tags = {
    Name = "terraform-example"
  }
}

resource "aws_security_group" "webserversg" {
  name = "terraform-example-webserversg"

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "service_port" {
  description = "Service Port for HTTP Request"
  type        = number
  default     = 8080
}
```

#### 연결이 정상적으로 되는 것을 확인

<img src="https://github.com/rlatkd/terraform/blob/main/assets/connected.jpg">

## 2. 개선점

### 환경변수를 이용하여 ip와 포트를 하드코딩하지 않고 자동으로 연동되게 설정

```
...
...
  user_data              = <<-EOF
    #!/bin/bash
    echo "Hello, World" > index.html
    nohup busybox httpd -f -p ${var.service_port} &
  EOF
...
...
resource "aws_security_group" "webserversg" {
  name = "terraform-example-webserversg"

  ingress {
    from_port   = var.service_port
    to_port     = var.service_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "service_port" {
description = "Service Port for HTTP Request"
type = number
default = 8080
}
...
...
```

### outputs를 이용해 terraform apply 후 자동으로 `IPAddr:PORT` 를 print하게 설정

```
...
...
output "public_ip" {
value = aws_instance.example.public_ip
description = "Public IP for Web Server"
}

output "service_url" {
value = "http://${aws_instance.example.public_ip}:${var.service_port}"
description = "Web Server Service URL"
}
...
...
```

#### terraform apply 실행 후 terminal 출력물

<img src="https://github.com/rlatkd/terraform/blob/main/assets/outputs.jpg">
