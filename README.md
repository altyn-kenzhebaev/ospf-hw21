# OSPF
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/ospf-hw21.git`
В текущей директории появится папка с именем репозитория. В данном случае ospf-hw21. Ознакомимся с содержимым:
```
cd ospf-hw21
ls -l
README.md
ansible
Vagrantfile
```
Здесь:
- ansible - папка с плэйбуками
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```
## Поднять три виртуалки
Поднимаем 3 ВМ с 3 дополнительными сетевыми картами:
```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
:router1 => {
        :box_name => "ubuntu/jammy64",
        :nets => {
                  :net2 => {
                    :ip => '10.0.10.1', 
                    :adapter => 2, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r1-r2"
                  },
                  :net3 => {
                    :ip => '10.0.12.1', 
                    :adapter => 3, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r1-r3"
                  },
                  :net4 => {
                    :ip => '192.168.10.1', 
                    :adapter => 4, 
                    :netmask => "255.255.255.0", 
                    :virtualbox__intnet => "net1"
                  },
                 }
  },

  :router2 => {
        :box_name => "ubuntu/jammy64",
        :nets => {
                  :net2 => {
                    :ip => '10.0.10.2', 
                    :adapter => 2, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r1-r2"
                  },
                  :net3 => {
                    :ip => '10.0.11.2', 
                    :adapter => 3, 
                    :netmask => "255.255.255.252", 
                    :virtualbox__intnet => "r2-r3"
                  },
                  :net4 => {
                    :ip => '192.168.20.1', 
                    :adapter => 4, 
                    :netmask => "255.255.255.0", 
                    :virtualbox__intnet => "net2"
                  },
                 }
  },

  :router3 => {
    :box_name => "ubuntu/jammy64",
    :nets => {
              :net2 => {
                :ip => '10.0.11.1', 
                :adapter => 2, 
                :netmask => "255.255.255.252", 
                :virtualbox__intnet => "r2-r3"
              },
              :net3 => {
                :ip => '10.0.12.2', 
                :adapter => 3, 
                :netmask => "255.255.255.252", 
                :virtualbox__intnet => "r1-r3"
              },
              :net4 => {
                :ip => '192.168.30.1', 
                :adapter => 4, 
                :netmask => "255.255.255.0", 
                :virtualbox__intnet => "net3"
              },
             }
  },
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        boxconfig[:nets].each do |netname, netconf|
          box.vm.network "private_network", 
          ip: netconf[:ip], 
          adapter: netconf[:adapter], 
          netmask: netconf[:netmask], 
          virtualbox__intnet: netconf[:virtualbox__intnet]
        end

        if boxname.to_s == "router3"
          box.vm.provision "ansible" do |ansible|
            ansible.playbook = 'ansible/ospf.yml'
            ansible.inventory_path = "ansible/hosts"
            ansible.compatibility_mode = "2.0"
          end
        end

      end

  end
  
end
```

## Объединить их разными vlan
### Поднять OSPF между машинами на базе Quagga
Для данной настройки запущен ansible playbook, где устанавливается ПО, настраивается по шаблону frr.conf.j2, и применив кофигурацию перезапускает сервис:
```
# tasks/main.yml
---
  - name: disable ufw service
    service:
      name: ufw
      state: stopped
      enabled: false

  # Добавляем gpg-key репозитория
  - name: add gpg frrouting.org
    apt_key:
      url: "https://deb.frrouting.org/frr/keys.asc"
      state: present

  # Добавляем репозиторий https://deb.frrouting.org/frr
  - name: add frr repo
    apt_repository:
      repo: 'deb https://deb.frrouting.org/frr {{ ansible_distribution_release }} frr-stable'
      state: present
  
  # Обновляем пакеты и устанавливаем FRR
  - name: install FRR packages
    apt:
      name: 
        - frr
        - frr-pythontools
      state: present
      update_cache: true

  # Включаем маршрутизацию транзитных пакетов
  - name: set up forward packages across routers
    sysctl:
      name: net.ipv4.conf.all.forwarding
      value: '1'
      state: present
  
  # Копируем файл daemons на хосты, указываем владельца и права
  - name: base set up OSPF 
    template:
      src: daemons
      dest: /etc/frr/daemons
      owner: frr
      group: frr
      mode: 0640

  # Копируем файл frr.conf на хосты, указываем владельца и права
  - name: set up OSPF 
    template:
      src: frr.conf.j2
      dest: /etc/frr/frr.conf
      owner: frr
      group: frr
      mode: 0640
    tags:
      - setup_ospf

  # Перезапускам FRR и добавляем в автозагрузку
  - name: restart FRR
    service:
      name: frr
      state: restarted
      enabled: true
    tags:
      - setup_ospf
...
# frr.conf.j2
# default to using syslog. /etc/rsyslog.d/45-frr.conf places the log in
# /var/log/frr/frr.log
#
# Note:
# FRR's configuration shell, vtysh, dynamically edits the live, in-memory
# configuration while FRR is running. When instructed, vtysh will persist the
# live configuration to this file, overwriting its contents. If you want to
# avoid this, you can edit this file manually before starting FRR, or instruct
# vtysh to write configuration to a different file.

frr version 8.5
frr defaults traditional
hostname {{ ansible_hostname }}
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
interface enp0s8
 description {{ net1_desc }}
 ip address {{ net1_ip }}
 ip ospf mtu-ignore
{% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
{% elif ansible_hostname == 'router2' and symmetric_routing == true %}
 ip ospf cost 1000
{% else %}
 !ip ospf cost 450
{% endif %}
 ip ospf hello-interval 10
 ip ospf dead-interval 30
!
interface enp0s9
 description {{ net2_desc }}
 ip address {{ net2_ip }}
 ip ospf mtu-ignore
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface enp0s10
 description {{ net3_desc }}
 ip address {{ net3_ip }}
 ip ospf mtu-ignore
 ip ospf hello-interval 10
 ip ospf dead-interval 30 
!
router ospf
 {% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
 network {{ net1_net }} area 0
 network {{ net2_net }} area 0
 network {{ net3_net }} area 0 
 neighbor {{ net1_neighbor }}
 neighbor {{ net2_neighbor }}

log file /var/log/frr/frr.log
default-information originate always
```
### Изобразить ассиметричный роутинг
```
# tasks/main.yml
  # Отключаем запрет ассиметричного роутинга 
  - name: set up asynchronous routing
    sysctl:
      name: net.ipv4.conf.all.rp_filter
      value: '0'
      state: present

# frr.conf.j2
{% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
{% else %}
 !ip ospf cost 450
{% endif %}
```

### Сделать один из линков "дорогим", но что бы при этом роутинг был симметричным
```
# frr.conf.j2
{% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
{% elif ansible_hostname == 'router2' and symmetric_routing == true %}
{% else %}
 !ip ospf cost 450
{% endif %}

# vars/main.yml
symmetric_routing: true
```
