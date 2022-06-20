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

<h4># Добавить в Vagrantfile еще дисков</h4>

<p>В домашней директории создадим директорию disksystem, в которой будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./disksystem
[user@localhost otus]$</pre>

<p>Перейдём в директорию disksystem:</p>

<pre>[user@localhost otus]$ cd ./disksystem/
[user@localhost disksystem]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost disksystem]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :otuslinux => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.101',
        :disks => {
                :sata1 => {
                        :dfile => './sata1.vdi',
                        :size => 250,
                        :port => 1
                },
                :sata2 => {
                        :dfile => './sata2.vdi',
                        :size => 250, # Megabytes
                        :port => 2
                },
                :sata3 => {
                        :dfile => './sata3.vdi',
                        :size => 250,
                        :port => 3
                },
                :sata4 => {
                        :dfile => './sata4.vdi',
                        :size => 250, # Megabytes
                        :port => 4
                }

        }


  },
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
                  vb.customize ["modifyvm", :id, "--memory", "1024"]
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

<p>После блока sata4 добавим блок sata5:</p>

<pre>
...
                :sata4 => {
                        :dfile => './sata4.vdi',
                        :size => 250, # Megabytes
                        :port => 4
                },
                :sata5 => {
                        :dfile => './sata5.vdi',
                        :size => 250, # Megabytes
                        :port => 5
                }
...
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost disksystem]$ vagrant up</pre>

<p>и войдём в неё:</p>

<pre>[user@localhost disksystem]$ vagrant ssh
[vagrant@otuslinux ~]$</pre>

<p>Заходим под правми root:</p>

<pre>[vagrant@otuslinux ~]$ sudo -i
[root@otuslinux ~]#</pre>

<h4># Собрать RAID0/1/5/10 - на выбор</h4>

<p>Далее нужно определиться какого уровня RAID будем собирать. Для это посмотрим какие блочные устройства у нас есть и исходя из их кол-во, размера и поставленной задачи определимся.<br />Сделать это можно несколькими способами:</p>
