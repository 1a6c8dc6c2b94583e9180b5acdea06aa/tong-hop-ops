###Overlapping network using namespace:
- Openstack l� m� h�nh multitenancy -> m?i tenant c� th? t?o ri�ng nhi?u private network, router, firewall, loadbalancer� Neutron c� kh? nang t�ch bi?t c�c t�i nguy�n m?ng gi?a c�c tenant s? d?ng linux namespace.
- m?i network namespace c� ri�ng cho m�nh routes, firewall rules, interface devices. M?i network hay router do tenant t?o ra d?u hi?n h?u du?i d?ng 1 network namespace -> cho ph�p c�c tenant t?o dc c�c network tr�ng nhau ( overlapping) nhung v?n d?c l?p m� k b? xung d?t (isolated)
- C�c namespace hi?n th? du?i d?ng:
     - qdhcp- <network UUID>
     - qrouter- <router UUID>
     - qlbaas- <load balancer UUID>
- d? list c�c namespace dang c� s? d?ng l?nh:

        #ipnetns

- xem c?u h�nh 1 network namespace s? d?ng l?nh:
	
         #ipnetns exec NAMESPACE command

c�c l?nh c� th? s? d?ng ? d�y nhu ip, route, iptables, telnet ho?c ping�

-------------------------------------

###ML2 plugin:

- Nh? ki?n tr�c plugin, Neutron c� kh? nang m? r?ng th�ng qua ML2 plugin
- LB v� OVS l� c�c c?u tr�c nguy�n kh?i, c� nghia ch�ng ch? ho?t d?ng ri�ng r? m� k s? d?ng dc d?ng th?i v?i c�c c�ng ngh? kh�c. Nh? ML2 m� c�c c�ng ty c� th? t? t?o ra c�c plugin c?a ri�ng m�nh v� cho ph�p neutron s? d?ng ch�ng. ML2 t�ch c�c ch?c nang l�i c?a m?ng nhu IPAM, ID management� do d� c�c vendor k c?n l�m l?i c�c ch?c nang n�y m� ch? c?n t?p trung ph�t tri?n t�nh nang s?n ph?m. 
- ML2 plugin tach bi?t network type v� mechanism type. network type v� mechanism type c� kh? nang t�ch bi?t th�ng qua drivers (pluggable)
     - Type driver qu?n l� c�c m� h�nh m?ng c?a c? provider network v� tenant network nhu : local, flat, vlan, gre hay vxlan�
     - Mechanism driver l� 1 l?p ch?a driver c?a h?u h?t c�c plugin network c?a c�c vendor kh�c nhau nhu OVS, LB, HyperV, Arista, CiscoNexus�

---------------------------- 

###Enable packet forwading:

- Khi c?u h�nh Neutron, c?n c?u h�nh 3 kernel parameter v? forward g�i tin:
      - Net.ipv4.ip_forward = 1
      - Net.ipv4.conf.all.rp_filter = 0
      - Net.ipv4.conf.default.rp_filter = 0

- Net.ipv4.ip_forward cho ph�p host v?t l� c� kh? nang forward traffic t? VM ra internet . 2 tham s? sau l� 1 co ch?  nh?m ngan ch?n t?n c�ng t? ch?i d?ch v? b?ng c�ch ch?n c�c d?a ch? IP gi? m?o. Khi dc c?u h�nh, linux kernel s? ki?m tra t?ng packet d? d?m b?o d?a ch? IP ngu?n l� c� th? d?nh tuy?n ngu?c t? interface m� n� g?i traffic d?n, t?c c�c g�i tin nh?n dc tr�n 1 interface t? IP d� s? c� kh? nang ph?n h?i l?i -> k ph?i IP gi? m?o.
2 th�ng s? n�y du?c set v? 0, co ch? n�y dc tri?n khai thay b?ng iptables rules

----------------

###C?u h�nh neutron trong file /etc/neutron/neutron.conf:

####s? d?ng Keystone d? x�c th?c :
    # crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone  
    # crudini --set /etc/neutron/neutron.conf DEFAULT api_paste_config /etc/neutron/api-paste.ini 
    # crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_host controller 
    # crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_port 35357 
    # crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_protocol http 66 
    # crudini --set /etc/neutron/neutron.conf keystone_authtoken admin_tenant_name service 
    # crudini --set /etc/neutron/neutron.conf keystone_authtoken admin_user neutron
    # crudini --set /etc/neutron/neutron.conf keystone_authtoken admin_password neutron

####s? d?ng file /etc/neutron/api-paste.ini l�m m�i tru?ng d? x�c th?c:
    # crudini --set /etc/neutron/api-paste.ini filter:authtoken auth_host controller 
    # crudini --set /etc/neutron/api-paste.ini filter:authtoken auth_uri http://controller:5000 
    # crudini --set /etc/neutron/api-paste.ini filter:authtoken admin_tenant_name service 
    # crudini --set /etc/neutron/api-paste.ini filter:authtoken admin_user neutron 
    # crudini --set /etc/neutron/api-paste.ini filter:authtoken admin_password neutron

####C?u h�nh Neutron s? d?ng d?ch v? message queue
    # crudini --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
    # crudini --set /etc/neutron/neutron.conf DEFAULT rabbit_host = controller
    # crudini --set /etc/neutron/neutron.conf DEFAULT rabbit_password = RABBIT_PASS

####C?u h�nh nova s? d?ng neutron thay cho nova-network trong /etc/nova/nova.conf

    # crudini --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API 
    # crudini --set /etc/nova/nova.conf DEFAULT neutron_url http://controller:9696
    # crudini --set /etc/nova/nova.conf DEFAULT neutron_auth_strategy keystone 
    # crudini --set /etc/nova/nova.conf DEFAULT neutron_admin_tenant_name service 
    # crudini --set /etc/nova/nova.conf DEFAULT neutron_admin_username neutron 68 
    # crudini --set /etc/nova/nova.conf DEFAULT neutron_admin_password neutron 
    # crudini --set /etc/nova/nova.conf DEFAULT neutron_admin_auth_url http://controller:35357/v2.0

(nova s? d?ng firewall_driver d? c?u h�nh firewall cho nova-network, khi s? d?ng neutron, d�ng n�y c?n dc c?u h�nh l� nova.virt.firewall.NoopFirewallDriver d? nova k s? d?ng firewall_driver n?a )

####C?u h�nh nova s? d?ng neutron api d? tri?n khai security group:
    # crudini --set /etc/nova/nova.conf DEFAULT security_group_api neutron

####C?u h�nh core_plugin
�? th�ng b�o cho neutron bi?t s? d?ng c�ng ngh? n�o d? t?o switch ?o, c?n khai b�o trong core_plugin. M?i c�ng ngh? c� 1 driver ri�ng:

  - LinuxBridge: neutron.plugins.linuxbridge.lb_neutron_plugin.LinuxBridgePluginV2
  - Open vSwitch: neutron.plugins.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2

? d�y ta s? d?ng ML2 plugin d? m? r?ng kh? nang s? d?ng c�c c�ng ngh? kh�c nhau, chi ti?t ph?n khai b�o c?u h�nh c�c c�ng ngh? du?c c?u h�nh trong file /etc/neutron/plugins/ml2/ml2_config.ini

####DHCP v� ti?n tr�nh Dnsmasq
Khi DHCP du?c enable, ti?n tr�nh dnsmasq du?c kh?i ch?y b�n trong m?i dhcp namespace, c� nhi?m v? d�ng vai tr� nhu 1 dhcp server c?p ip d?ng cho c�c VM trong 1 tenant. 
M?i dhcp namespace du?c g�n 1 port tap v� n?i t?i br-int tr�n node network. Show dhcp namespace port b?ng c�u l?nh:

    ip netns exec {dhcp-namespace-ID} ip a
ta s? th?y port tap d�.

####Quy u?c d?t t�n port trong openstack:
**Tr�n compute node**

- Linux bridge: qbr-ID
Linux bridge n?m gi?a VM v� br-int, g?m 2 port:
 - port tap g?n v?i VM: tap-ID
 - port veth pair g?n v?i br-int: qvb-ID
- Br-int :
 - port veth pair g?n v?i linux bridge: qvo-ID
 - port patch g?n v?i br-tun
 
Tr�n 1 network th� c�c port c?a c�c thi?t b? n�y c� chung ID l� ID c?a network d�.

**Tr�n network node**

- Br-int: cung c?p router ?o v� DHCP cho instance. g?m c�c port:
 - port tap g?n v?i DHCP namespace: tap-ID
 - port qr g?n v?i router namespace: qr-ID
 
- Br-ex: cung c?p external connection. G?m port qg g?n v?i router namespace: qg-ID
