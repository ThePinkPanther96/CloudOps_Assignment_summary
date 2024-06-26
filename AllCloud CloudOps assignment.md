1. enter console
2. change region
3. Navigate to s3 > cloudops-test-keys
4. download the key from the bucket
6. used this command (In Windows CMD) to log in after downloading the pem key:

```shell
ssh -i "C:\Users\galro\Downloads\CloudOpsKey.pem" ec2-user@54.247.10.194
```

7. Elevating to root: ``sudo su -``
8. Navigating to ``/var/local/aws-test``
9. Initialize the Terraform environment:
```bash
terraform init
```

```Output
Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your environment. If you forget, other
commands will detect it and remind you to do so if necessary.
```

10. Run Terraform plan to create execution plan and show changes will be made:
```bash
[root@ip-10-0-250-167 aws-test]# terraform plan
var.aws_region
  The AWS region to create things in.

  Enter a value: eu-west-1

var.full_name
  Your full name (without spaces).

  Enter a value: gr
```

Got an error:
```Error Output
Error locking state: Error acquiring the state lock: could not write lock info for "terraform.tfstate": write .terraform.tfstate.lock.info: no space left on device

Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.
```

11. After checking disk space with ``df -H`` I saw that the disk is 100% full:

```Output
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M   56K  488M   1% /dev
tmpfs           497M     0  497M   0% /dev/shm
/dev/xvda1      7.8G  7.8G     0 100% /
```


13. So, I deleted large log files in attempt to free up some space: 

```bash
sudo rm -rf /var/log/DELETEME
```

14. Checked the disk space again with ``df -H``
```Output
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M   56K  488M   1% /dev
tmpfs           497M     0  497M   0% /dev/shm
/dev/xvda1      7.8G  1.4G  6.4G  18% /
```

15. Tried ``terraform plan`` again. It worked now (Summarized output):
```Output
+ aws_autoscaling_group.web-asg
...
+ aws_elb.web-elb
...
+ aws_internet_gateway.igw
...
+ aws_launch_configuration.web-lc
...
+ aws_route.r
...
+ aws_security_group.default
...
+ aws_security_group.elb-sg
...
+ aws_subnet.subnet.0
...
+ aws_subnet.subnet.1
...
+ aws_vpc.main
...
<= data.template_file.bootstrap
...
+ random_id.unique-id
    b64:         "<computed>"
    b64_std:     "<computed>"
    b64_url:     "<computed>"
    byte_length: "4"
    dec:         "<computed>"
    hex:         "<computed>"

Plan: 11 to add, 0 to change, 0 to destroy.
```

16. Proceeding to apply Terraform configuration to create the resources that will be defined in the Terraform files:
```bash
[root@ip-10-0-250-167 aws-test]# terraform apply
var.aws_region
  The AWS region to create things in.

  Enter a value: eu-west-1

var.full_name
  Your full name (without spaces).

  Enter a value: gr
```

```Output
...

Apply complete! Resources: 11 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path:

Outputs:

address = elb-web-example-gr-0095dee7-101026811.eu-west-1.elb.amazonaws.com



```

17. I noticed that a the web server "web-example-gr-0095dee7" was created, how ever access to the site is not possible beacuse "This page isn't working". 
18. So, I logged in to the machine: 
```bash
ssh -i "C:\Users\galro\Downloads\CloudOpsKey.pem" ubuntu@34.255.0.246
```

19. The Apache service was down:
```bash
sudo systemctl status apache2
sudo: unable to resolve host ip-10-0-163-180
● apache2.service - LSB: Apache2 web server
   Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
  Drop-In: /lib/systemd/system/apache2.service.d
           └─apache2-systemd.conf
   Active: inactive (dead)
     Docs: man:systemd-sysv-generator(8)
```

20. So  I started it:
```bash
ubuntu@ip-10-0-163-180:~$ sudo systemctl status apache2
sudo: unable to resolve host ip-10-0-163-180
● apache2.service - LSB: Apache2 web server
   Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
  Drop-In: /lib/systemd/system/apache2.service.d
           └─apache2-systemd.conf
   Active: active (running) since Wed 2024-06-26 14:00:46 UTC; 3s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1596 ExecStart=/etc/init.d/apache2 start (code=exited, status=0/SUCCESS)
    Tasks: 55
   Memory: 8.4M
      CPU: 49ms
   CGroup: /system.slice/apache2.service
           ├─1620 /usr/sbin/apache2 -k start
           ├─1623 /usr/sbin/apache2 -k start
           └─1624 /usr/sbin/apache2 -k start
```

21. The site was reachable with Error 404 (Not Found) the HTML page did not load correctly.
22. I checked the security groups and everything was fine. 
23. So, I checked the configuration file: ``/etc/apache2/sites-available/000-default.conf``
24. And I noticed that `` DocumentRoot`` is set as ``DocumentRoot /var/www/html12`` which is not correct because because the directory is named   ``/var/www/html``. so I fixed it.
25. I also tried to change the permissions of ``/var/www/html`` to rule out permission:
```bash
sudo chmod -R 755 /var/www/html
sudo chown -R www-data:www-data /var/www/html
```

27. Then I got this output:
```Output
root@ip-10-0-163-180:~# sudo chmod -R 755 /var/www/html own -R www-data:www-data /var/www/html sudo: unable to resolve host ip-10-0-163-180 root@ip-10-0-163-180:~# sudo chown -R www-data:www-data /var/www/html sudo: unable to resolve host ip-10-0-163-180 root@ip-10-0-163-180:~#
```

29. And when I tried to log  back to the website I saw a different error: "This page isn't working" (HTTP ERROR 504)
30. So, I turn to check ``/etc/hosts`` as the above error suggested and I added the instance hostname to the file:
```
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

# Added:
127.0.0.1 ip-10-0-163-180
```

28. I restarted the Apache service again.
29. when I tried to connect to the webpage again I finally was able to see the HTML page: 
	http://elb-web-example-gr-0095dee7-101026811.eu-west-1.elb.amazonaws.com/

