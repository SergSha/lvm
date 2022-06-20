<h3>### LVM ###</h3>

<h4>Описание домашнего задания</h4>

<ol>
  <li>уменьшить том под / до 8G</li>
  <li>выделить том под /home</li>
  <li>выделить том под /var (/var - сделать в mirror)</li>
  <li>для /home - сделать том для снэпшотов</li>
  <li>прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)</li>
  <li>Работа со снапшотами:<br />
  <ul>
    <li>сгенерировать файлы в /home/</li>
    <li>снять снэпшот</li>
    <li>удалить часть файлов</li>
    <li>восстановиться со снэпшота (залоггировать работу можно утилитой script, скриншотами и т.п.)</li>
    <li>на нашей куче дисков попробовать поставить btrfs/zfs:<br />
    <ul>
      <li>с кешем и снэпшотами</li>
      <li>разметить здесь каталог /opt</li>
    </ul>
    </li>
  </ul>
  </li>
</ol>
<br />

<h4># Уменьшить том под / до 8G</h4>

<p>В домашней директории создадим директорию lvm, в которой будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./lvm
[user@localhost otus]$</pre>

<p>Перейдём в директорию lvm:</p>

<pre>[user@localhost otus]$ cd ./lvm/
[user@localhost lvm]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost disksystem]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :lvm => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        :ip_addr => '192.168.56.101',
    :disks => {
        :sata1 => {
            :dfile => home + '/VirtualBox VMs/sata1.vdi',
            :size => 10240,
            :port => 1
        },
        :sata2 => {
            :dfile => home + '/VirtualBox VMs/sata2.vdi',
            :size => 2048, # Megabytes
            :port => 2
        },
        :sata3 => {
            :dfile => home + '/VirtualBox VMs/sata3.vdi',
            :size => 1024, # Megabytes
            :port => 3
        },
        :sata4 => {
            :dfile => home + '/VirtualBox VMs/sata4.vdi',
            :size => 1024,
            :port => 4
        }
    }
  },
}

Vagrant.configure("2") do |config|

    config.vm.box_version = "1804.02"
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "256"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                  needsController =  true
                            end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                           vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
  
        box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
            yum install -y mdadm smartmontools hdparm gdisk
          SHELL
  
        end
    end
  end
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost lvm]$ vagrant up</pre>

<p>и войдём в неё:</p>

<pre>[user@localhost lvm]$ vagrant ssh
[vagrant@lvm ~]$</pre>

<p>Заходим под правми root:</p>

<pre>[vagrant@lvm ~]$ sudo -i
[root@lvm ~]#</pre>

<p>Перед началом работы поставьте пакет xfsdump - он будет необходим для снятия копии / тома.:</p>

<pre>[root@lvm ~]# yum -y install xfsdump
...
Installed:
  xfsdump.x86_64 0:3.1.7-1.el7                                                  

Dependency Installed:
  attr.x86_64 0:2.4.46-13.el7                                                   

Complete!
[root@lvm ~]#</pre>

<p>Посмотрим текущее состояние системы:</p>

<pre>[root@lvm ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0 
[root@lvm ~]#</pre>

<pre>[root@lvm ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
[root@lvm ~]#</pre>

<p>Подготовим временный том для / раздела:</p>

<pre>[root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@lvm ~]#</pre>

<pre>[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
[root@lvm ~]#</pre>

<pre>[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
[root@lvm ~]#</pre>

<p>Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:</p>

<pre>[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]#</pre>

<pre>[root@lvm ~]# mount /dev/vg_root/lv_root /mnt
[root@lvm ~]#</pre>

<p>Следующей командой скопируем все данные с / раздела в /mnt:</p>

<pre>[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
...
xfsrestore: restore complete: 9 seconds elapsed
xfsrestore: Restore Status: SUCCESS
[root@lvm ~]#</pre>

<p>Убедимся, что всё скопировалось:</p>

<pre>[root@lvm ~]# ls -l /mnt/
total 12
lrwxrwxrwx.  1 root    root       7 Jun 20 18:50 bin -> usr/bin
drwxr-xr-x.  2 root    root       6 May 12  2018 boot
drwxr-xr-x.  2 root    root       6 May 12  2018 dev
drwxr-xr-x. 79 root    root    8192 Jun 20 18:23 etc
drwxr-xr-x.  3 root    root      21 May 12  2018 home
lrwxrwxrwx.  1 root    root       7 Jun 20 18:50 lib -> usr/lib
lrwxrwxrwx.  1 root    root       9 Jun 20 18:50 lib64 -> usr/lib64
drwxr-xr-x.  2 root    root       6 Apr 11  2018 media
drwxr-xr-x.  2 root    root       6 Apr 11  2018 mnt
drwxr-xr-x.  2 root    root       6 Apr 11  2018 opt
drwxr-xr-x.  2 root    root       6 May 12  2018 proc
dr-xr-x---.  3 root    root     149 Jun 20 18:23 root
drwxr-xr-x.  2 root    root       6 May 12  2018 run
lrwxrwxrwx.  1 root    root       8 Jun 20 18:50 sbin -> usr/sbin
drwxr-xr-x.  2 root    root       6 Apr 11  2018 srv
drwxr-xr-x.  2 root    root       6 May 12  2018 sys
drwxrwxrwt.  8 root    root     193 Jun 20 18:48 tmp
drwxr-xr-x. 13 root    root     155 May 12  2018 usr
drwxrwxr-x.  2 vagrant vagrant   25 Jun 20 18:22 vagrant
drwxr-xr-x. 18 root    root     254 Jun 20 18:22 var
[root@lvm ~]# </pre>

<p>Затем переконфигурируем grub для того, чтобы при старте перейти в новый /</p>

<p>Сымитируем текущий root -> сделаем в него chroot и обновим grub:</p>

<pre>[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]#</pre>

<pre>[root@lvm ~]# chroot /mnt/
[root@lvm /]#</pre>

<pre>[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg 
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
[root@lvm /]#</pre>

<p>Обновим initrd image:</p>

<pre>[root@lvm /]# cd /boot/
[root@lvm boot]#</pre>

<pre>[root@lvm boot]# for i in $(ls initramfs-*.img); do mkinitrd -f -v $i $(echo $i | sed "s/initramfs-//g; s/.img//g"); done
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
[root@lvm boot]#</pre>

<p>Для того чтобы при загрузке был смонтирован root нужно в файле
/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root:</p>

<pre>[root@lvm boot]# sed -i "s/VolGroup00\/LogVol00/vg_root\/lv_root/g" /boot/grub2/grub.cfg 
[root@lvm boot]#</pre>

<p>Перезагружаемся с новым root томом:<br /> выходим с нового тома / с /mnt</p>

<pre>[root@lvm boot]# exit
exit
[root@lvm ~]#</pre>

<p>перезагружаемся</p>

<pre>[root@lvm ~]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
[user@localhost lvm]$</pre>

<p>Снова заходим в систему:</p>

<pre>[user@localhost lvm]$ vagrant ssh
Last login: Mon Jun 20 18:24:22 2022 from 10.0.2.2
[vagrant@lvm ~]$</pre>

<p>и заходим под правми root:</p>

<pre>[vagrant@lvm ~]$ sudo -i
[root@lvm ~]#</pre>
