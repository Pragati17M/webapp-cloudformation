# webapp-cloudformation

To launch the infrastructure for executing a web application using **AWS CloudFormation**, you'll need a **CloudFormation template** that automates the process of setting up a VPC, EC2 instances, a Load Balancer, and other resources. Below are the steps and an example CloudFormation template.

### **Steps to Launch Infrastructure Using CloudFormation:**

1. **Go to the AWS Management Console**:
   - Navigate to the **CloudFormation** service in the AWS Management Console.

2. **Create a New Stack**:
   - Click on **Create Stack** and choose **"With new resources (standard)"**.
   - You can either upload your template as a `.yaml` file or paste it directly into the editor.

3. **Configure Stack Details**:
   - Enter a **Stack Name** (e.g., `WebAppInfrastructure`).
   - Provide any **parameters** required by the template (e.g., instance type, key pair name).

4. **Review and Create**:
   - Review the stack configuration, ensure the permissions are appropriate, and click **Create Stack**.
   - Wait for CloudFormation to provision all the resources. You can monitor the process in the **Events** tab.

5. **Access and Test Your Web Application**:
   - Once the stack is created, access your web application using the **Load Balancer's DNS** or **Elastic IP**.
   - CloudFormation will automatically create and link all required resources like EC2, Security Groups, VPC, and Load Balancer.

---

### **CloudFormation Template for a Simple Web Application**

Here’s a basic **CloudFormation template** that launches a VPC, an EC2 instance in a public subnet, and an Application Load Balancer.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "CloudFormation Template to launch infrastructure for a web application."

Parameters:
  InstanceType:
    Description: "EC2 Instance Type"
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
    ConstraintDescription: "must be a valid EC2 instance type."
  
  KeyName:
    Description: "Name of an existing EC2 KeyPair to enable SSH access"
    Type: AWS::EC2::KeyPair::KeyName

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: MyVPC

  MyInternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref MyInternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true

  MyRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref MyRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref MyInternetGateway

  AssociateRouteTable:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref MyRouteTable

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Allow HTTP and SSH traffic"
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
      SubnetId: !Ref PublicSubnet
      ImageId: ami-0ebfd941bbafe70c6  # Amazon Linux 2023 AMI
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "Hello World from $(hostname -f)" > /var/www/html/index.html

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: WebAppLoadBalancer
      Subnets:
        - !Ref PublicSubnet
      SecurityGroups:
        - !Ref WebServerSecurityGroup

  LoadBalancerListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref WebAppTargetGroup

  WebAppTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref MyVPC
      Port: 80
      Protocol: HTTP
      TargetType: instance
      HealthCheckProtocol: HTTP
      HealthCheckPath: /
      Matcher:
        HttpCode: 200
```

### **Explanation of Key Sections**:
1. **VPC and Networking**:
   - **VPC** is created with a CIDR block of `10.0.0.0/16`.
   - A **Public Subnet** with CIDR block `10.0.1.0/24` is provisioned.
   - **Internet Gateway** and **Route Table** are set up to allow internet traffic.

2. **Security Groups**:
   - The **WebServerSecurityGroup** opens port 22 (SSH) and port 80 (HTTP) to the internet.

3. **EC2 Instance**:
   - **Amazon Linux 2023 AMI** is used to launch an instance.
   - **UserData** is used to install and start **Apache HTTP Server** (`httpd`), making the instance a simple web server.

4. **Application Load Balancer**:
   - An **Application Load Balancer** (ALB) is created to distribute traffic across EC2 instances.
   - A **Listener** is set up for port 80 to forward traffic to the web server.

5. **Auto Scaling Group (Optional)**:
   - You can add an Auto Scaling Group with rules to automatically scale EC2 instances based on traffic or resource usage.

---

This template sets up a basic web application infrastructure on AWS. Let me know if you need further customization or additional services like databases or Auto Scaling.
