#HA NAT Gateways

Even with the advent of the NAT Gateway service, running your own NAT instances can still be a cost-effective option, particularly if your traffic pattern is high inbound, and low outbound - As EC2 isn't charged on the inbound traffic, it works out cheaper.

This is based this work: http://www.cakesolutions.net/teamblogs/making-aws-nat-instances-highly-available-without-the-compromises

And expanded to

 - Work on any Region
 - Multi-AZ aware
 - t2-nano aware for lab setups.

Cautions:

 - Only extensively tested in Tokyo, which has a unique 1a & 1c AZ set up
 - Only extensively tested on t2.nano instances - check my mappings for image IDs before trusting that 
 - Limitation of the way the networking interface attachment is handled: you cn't really patch these boxes as if you update them and reboot, eth0 comes back up and you screw routing. as others have said, this needs a little work to use a better mechanism 

Step-by-Step
=============

1. create EIPs for each AZ
2. create ENI for each AZ - give the ENI a distinct description, you'll use this later...
3. associate the EIP to the ENI
4. disable src/dst checking on ENI
5. set the default route for the associated private routing table to point at the ENI for that AZ
6. create an IAM Role for HA-NAT
7. create the IAM Policy for HA-NAT
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": [
            "ec2:AttachNetworkInterface",
            "ec2:DescribeNetworkInterfaces",
        ],
        "Resource": [
            "*"
        ]
    }
}
8. associate the policy to the Role
9. create an AutoScaling Launch Config that uses the following UserData as text passed to the instance launching, with some voodoo to figure out the region.

#!/bin/bash
 my_eni_id=$(/usr/bin/aws ec2 describe-network-interfaces  --region us-east-1 --filters Name='description',Values='<distinct description here>' --query 'NetworkInterfaces[*].{ID:NetworkInterfaceId}' --output text)
 my_instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
 /usr/bin/aws ec2 attach-network-interface --network-interface-id ${my_eni_id} --instance-id ${my_instance_id} --device-index 1 --region 'us-east-1'
 /bin/sleep 60
 /sbin/ifdown eth0
 /sbin/iptables -t nat -A POSTROUTING -j MASQUERADE
 /sbin/iptables -A FORWARD -j ACCEPT
 /sbin/sysctl -w "net.ipv4.ip_forward=1"

10. create autoscaling group
