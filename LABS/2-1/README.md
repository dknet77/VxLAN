## VxLAN
Leaf1 и Leaf2 имеют подключение серверов в одном влане, но нет транков между ними.

Тем не менее серверам требуется L2 подключение и такая задача решается с помощью VxLAN.

Underlay сеть нужна только для доступности ip интерфейсов (loopbacks, ptp).

Overlay передает необходимые данные об удаленном L2 сегменте поверх L3 сети.

Два роутера могут связать один влан через L3 cеть без настроек EVPN.

	feature vn-segment-vlan-based
	feature nv overlay
 
 	vlan 8
  		name SERVER-VLAN
  		vn-segment 10008
	
	interface Ethernet1/4
  		description *** LINK TO SERVER ***
  		switchport access vlan 8

	interface nve1 
 		 no shutdown
 		 source-interface loopback2
 		 member vni 10008
		ingress-replication protocol static
  			peer 10.1.0.12
	 		
 Интерфейс nve по сути похож на туннельный интерфейс GRE.
 
 <mark style="background-color: #FFFF00"> ingress-replication </mark> указывает каким способом будет расспостраняться BUM трафик. 
 
 В этом случае статически прописан второй роутер, у которого есть подключение в том же влане.
 
 При увеличении кол-ва Leaf-коммутаторов и вланов которые необходимо растянуть по всей сети, такой подход не маштабируем.
 
 EVPN решает эти задачи в топологии Spine&Leaf.
 
 Cущестуют важные нюансы конфигурации в зависимости от того какой протокол (iBGP or eBGP) исользует EVPN. 


## VxLAN + EVPN при iBGP (Overlay)

![4-1-1.png](4-1-1.png)

#### %%%%%%%%%%%%%%%%%%%%%%%%% LEAF (L-1-1) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%
 
    feature vn-segment-vlan-based ! Service Model: VlanBased - У каждого VLAN свой MAC-VRF (свои RD/RT) (N:N)
    feature nv overlay ! дла работы VxLAN
    nv overlay evpn ! в качестве overlay будет evpn

    interface loopback2
     ip address 10.1.0.11/32 ! надо сделать доступным через underlay
    
    interface Ethernet1/4
     description *** LINK TO VPC1 ***
     switchport access vlan 8

    vlan 8
     name VPC8
     vn-segment 10008 ! Ассоциируем VLAN с номер VNI 
    
    interface nve1 
     no shutdown
     host-reachability protocol bgp
     source-interface loopback2
     member vni 10008 ! Добавляем VNI 10000 для работы через интерфейс NVE. для инкапсуляции в VxLAN
        ingress-replication protocol bgp ! указываем, что для распространения BUM трафика используем BGP

    evpn
     vni 10008 l2
          rd auto    
		  ! auto-derived Route Distinguisher (RD): MAC-VRF with BGP Router ID 10.0.0.11 and VLAN 8 (32767+8) - RD 10.0.0.11:32775
          route-target import auto 
		  !  auto derived Route-Target (RT): MAC-VRF within ASN 64512 and L2VNI 10008 - Route-Target 64512:10008
          route-target export auto

    router bgp 64512
     router-id 10.0.0.11
     timers bgp 3 9
     bestpath as-path multipath-relax
     reconnect-interval 12

     address-family l2vpn evpn
          maximum-paths 10

     template peer SPINE-IPV4-OVERLAY
          remote-as 64512
          update-source loopback2
          address-family l2vpn evpn
               send-community
               send-community extended

     neighbor 10.1.11.1
          inherit peer SPINE-IPV4-OVERLAY
#### %%%%%%%%%%%%%%%%%%%%%%%%% LEAF (L-1-2) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%

    feature vn-segment-vlan-based 
    feature nv overlay 
    nv overlay evpn 

	interface loopback2
		ip address 10.1.0.12/32  
		
	interface Ethernet1/4
		description *** LINK TO VPC2 ***
		switchport access vlan 8

	vlan 8
	name VPC8
	vn-segment 10008

	interface nve1
		no shutdown
		host-reachability protocol bgp
		source-interface loopback2
		member vni 10008
		ingress-replication protocol bgp 

	evpn
		vni 10008 l2
		rd auto
		route-target import auto
		route-target export auto

	router bgp 64512
		router-id 10.0.0.12
		timers bgp 3 9
		bestpath as-path multipath-relax
		reconnect-interval 12

			address-family l2vpn evpn
				maximum-paths 10

		template peer SPINE-IPV4-OVERLAY
			remote-as 64512
			update-source loopback2
				address-family l2vpn evpn
					send-community
					send-community extended

		neighbor 10.1.11.1
			inherit peer SPINE-IPV4-OVERLAY

#### %%%%%%%%%%%%%%%%%%%%%%%%% SPINE (S-1-1) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%

	    nv overlay evpn

	interface loopback2
		ip address 10.1.11.1/32
 
	router bgp 64512
		router-id 10.0.11.1
		timers bgp 3 9
		reconnect-interval 12
		log-neighbor-changes

			address-family l2vpn evpn
			maximum-paths 10

		template peer LEAF-IPV4-OVERLAY
			remote-as 64512
			update-source loopback2
				address-family l2vpn evpn
					route-reflector-client
					send-community
					send-community extended

		neighbor 10.1.0.11
			inherit peer LEAF-IPV4-OVERLAY

		neighbor 10.1.0.12
			inherit peer LEAF-IPV4-OVERLAY

-------------------------------------------------------------------------------------------------------------


[PRINTOUT- with iBGP](https://github.com/dknet77/VxLAN/tree/main/LABS/2-1/OUPUT/VxLAN_iBGP.txt)

[PRINTOUT - with eBGP](https://github.com/dknet77/VxLAN/tree/main/LABS/2-1/OUPUT/eBGP.txt)

[REMARKS](https://github.com/dknet77/VxLAN/tree/main/LABS/2-1/APPENDIX/NB.txt)

[Адресное пространство IPv4 и IPv6](https://github.com/dknet77/VxLAN/tree/main/LABS/1-4/ip-plan.md)

-------------------------------------------------------------------------------------------------------------

 ## VxLAN + EVPN при eBGP (Overlay)

![4-1-2.png](4-1-2.png)

#### %%%%%%%%%%%%%%%%%%%%%%%%% LEAF (L-1-2) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%
	feature vn-segment-vlan-based
	feature nv overlay
	feature bgp
	nv overlay evpn
	
	interface loopback2
		ip address 10.1.0.12/32  
		ip router ospf 11 area 0.0.0.2 ! надо сделать доступным через underlay

	interface Ethernet1/4
		description *** LINK TO VPC2 ***
		switchport access vlan 8
		no shut

	vlan 8
		name VPC8
		vn-segment 10008
	
	interface nve1
		no shutdown
		host-reachability protocol bgp
		source-interface loopback2
		member vni 10008
			ingress-replication protocol bgp 

		evpn
			vni 10008 l2
			rd auto
			route-target import auto
			route-target export auto

		router bgp 64513
			router-id 10.0.0.12
			timers bgp 3 9
			bestpath as-path multipath-relax
			reconnect-interval 12
		address-family l2vpn evpn
			maximum-paths 10
		template peer SPINE-IPV4-OVERLAY
			remote-as 65000
			update-source loopback2
			ebgp-multihop 2
			address-family l2vpn evpn
			send-community
			send-community extended
			rewrite-evpn-rt-asn
		neighbor 10.1.12.1
			inherit peer SPINE-IPV4-OVERLAY
			
#### %%%%%%%%%%%%%%%%%%%%%%%%% LEAF (L-1-3) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%
	feature vn-segment-vlan-based
	feature nv overlay
	feature bgp
	nv overlay evpn

	interface loopback2
		ip address 10.1.0.13/32  
		ip router ospf 11 area 0.0.0.3 ! надо сделать доступным через underlay

	interface Ethernet1/4
		description *** LINK TO VPC2 ***
		switchport access vlan 8
		no shut

	vlan 8
		name VPC8
		vn-segment 10008

	interface nve1
		no shutdown
		host-reachability protocol bgp
		source-interface loopback2
		member vni 10008
		ingress-replication protocol bgp 

	evpn
		vni 10008 l2
			rd auto
			route-target import auto
			route-target export auto

	router bgp 64514
		router-id 10.0.0.13
		timers bgp 3 9
		bestpath as-path multipath-relax
		reconnect-interval 12
	address-family l2vpn evpn
		maximum-paths 10
		retain route-target all
	template peer SPINE-IPV4-OVERLAY
		remote-as 65000
		update-source loopback2
		ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
      rewrite-evpn-rt-asn
	neighbor 10.1.12.1
		inherit peer SPINE-IPV4-OVERLAY

#### %%%%%%%%%%%%%%%%%%%%%%%%% SPINE (S-2-1) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%
	nv overlay evpn

	interface loopback2
		ip address 10.1.12.1/32
		ip router ospf 11 area 0.0.0.0
	
	route-map NEXT-HOP-UNCHANGED permit 10
		set ip next-hop unchanged                   ! eBGP moment
  
	router bgp 65000
		router-id 10.0.12.1
		timers bgp 3 9
		reconnect-interval 12
		log-neighbor-changes
	ddress-family l2vpn evpn
		maximum-paths 10
	template peer LEAF-IPV4-OVERLAY
		update-source loopback2
		ebgp-multihop 5                             ! eBGP moment
    address-family l2vpn evpn
      send-community
      send-community extended
      route-map NEXT-HOP-UNCHANGED out  			! eBGP moment
	  rewrite-evpn-rt-asn 							! eBGP moment
	neighbor 10.1.0.12
		inherit peer LEAF-IPV4-OVERLAY
		remote-as 64513  							! eBGP moment
	neighbor 10.1.0.13
		inherit peer LEAF-IPV4-OVERLAY
		remote-as 64514  							! eBGP moment




