# REX-Ray on Barge with Vagrant

[REX-Ray](https://github.com/emccode/rexray) allows us to create and attach a VMDK (Virtual Machine Disk) dynamically with [VirtualBox](https://www.virtualbox.org/) as a persistent disk for a container.

This shows how to use REX-Ray on [Barge](https://atlas.hashicorp.com/ailispaw/boxes/barge) with [Vagrant](https://www.vagrantup.com/) instantly.

It's inspired by
- [Discovering Docker Volume Plugins and Applications with VirtualBox | EMC {code} – Blog](https://blog.emccode.com/2016/01/06/discovering-docker-volume-plugins-and-applications-with-virtualbox/)
- [Volume Plugins with Docker Toolbox and Boot2Docker | EMC {code} – Blog](https://blog.emccode.com/2016/01/19/volume-plugins-with-docker-toolbox-and-boot2docker/)

Please read them for further details.

## Requirements

- [VirtualBox](https://www.virtualbox.org/)
- [Vagrant](https://www.vagrantup.com/)
- [vagrant-triggers](https://github.com/emyl/vagrant-triggers)
  It's used to start/stop vboxwebsrv automatically.  
  I have tested it on Mac OSX, so you may need to modify Vagrantfile for the other platforms.

## Boot up

```bash
$ vagrant up
```

That's it.

## A sample use case

### Create a volume with REX-Ray

```bash
$ docker volume create --driver=rexray --name=mysql --opt=size=10
mysql
$ docker volume ls
DRIVER              VOLUME NAME
rexray              barge-packer-disk1.vmdk
rexray              mysql
$ ls -l "$HOME/VirtualBox Volumes"
total 2688
-rw-------  1 ailispaw  staff   1.3M Jun  9 09:55 mysql
```

`$HOME/VirtualBox Volumes` is defined in [assets/config.yml](assets/config.yml). You can change the location anywhere you want.

### Run a MariaDB container with the volume

```bash
$ docker run -d --volume-driver=rexray -e MYSQL_ROOT_PASSWORD=test -v mysql:/var/lib/mysql -p 3306:3306 --name mariadb mariadb
$ docker exec mariadb ls -l /var/lib/mysql
total 110644
-rw-rw---- 1 mysql mysql    16384 Jun  9 15:02 aria_log.00000001
-rw-rw---- 1 mysql mysql       52 Jun  9 15:02 aria_log_control
-rw-rw---- 1 mysql mysql 50331648 Jun  9 16:04 ib_logfile0
-rw-rw---- 1 mysql mysql 50331648 Jun  9 15:00 ib_logfile1
-rw-rw---- 1 mysql mysql 12582912 Jun  9 16:04 ibdata1
-rw-rw---- 1 mysql mysql        0 Jun  9 15:00 multi-master.info
drwx------ 2 mysql mysql     4096 Jun  9 15:00 mysql
drwx------ 2 mysql mysql     4096 Jun  9 15:00 performance_schema
-rw-rw---- 1 mysql mysql    24576 Jun  9 16:04 tc.log
```

### Check the contents of the volume

```bash
$ docker run --rm --volume-driver=rexray -v mysql:/mysql busybox ls -l /mysql
total 110644
-rw-rw----    1 999      999          16384 Jun  9 15:02 aria_log.00000001
-rw-rw----    1 999      999             52 Jun  9 15:02 aria_log_control
-rw-rw----    1 999      999       50331648 Jun  9 16:04 ib_logfile0
-rw-rw----    1 999      999       50331648 Jun  9 15:00 ib_logfile1
-rw-rw----    1 999      999       12582912 Jun  9 16:04 ibdata1
-rw-rw----    1 999      999              0 Jun  9 15:00 multi-master.info
drwx------    2 999      999           4096 Jun  9 15:00 mysql
drwx------    2 999      999           4096 Jun  9 15:00 performance_schema
-rw-rw----    1 999      999          24576 Jun  9 16:04 tc.log
```

## Notes

- To dettach the volume from the VM, stop all containers with the volume.
- You have to stop containers with the volume before `vagrant destoroy`, because the volume has been attached to the VM and it will be removed.
