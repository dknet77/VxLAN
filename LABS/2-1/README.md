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

 ## VxLAN + EVPN при eBGP (Overlay)

![4-1-2.png](4-1-2.png)

<mark style="background-color: #FFFF00"> ОБЩИЙ ПОДХОД </mark>

#### iBGP
  	one AS для Leaf-ов
  	one AS для Spine (RR)
    и one AS для Super Spine (RR)

#### eBGP
  	each Leaf - свою ASN
   	все Spine одна ASN (двухуровневая топология)
	все Super Spine в одной ASN, а Spine одного POD в другой ASN.
 	при этом на каждый POD у Spine своя AS-NUM.

---
 #### Private AS range: от 64512 до 65534
---
- well-known mandatory: as-path, next-hop, origin
- well-known discretionary: local-pref, atomic aggregate
- optional transitive: aggregator, community
- optional non-transitive: MED, originator id, cluser id
---
#### BGP multipath conditions:
- same weight
- same local-pref
- exactly the same as-path (not only lenght)
  если учитывать только длину, а не содержимое as-path: L-1-1(config-router)# bestpath as-path multipath-relax
- same origin, MED
- same IGP metric
- different next-hop
---
#### BGP Timers:
- MRAI (Minimum Route Advertisement Interval or out-delay) = eBGP def.30 sec / iBGP=0 (Leaf/Spine --> eBGP MRAI=0)
- keepalive = 60 (3sec)
- hold = 180 (9 sec)
- scan timer (next-hop tracking) = 60 sec (имеет смысл если есть multihop, а DC -> ttl=1)
---


[Адресное пространство IPv4 и IPv6](https://github.com/dknet77/VxLAN/tree/main/LABS/1-4/ip-plan.md)

[Remarks](https://github.com/dknet77/VxLAN/tree/main/LABS/1-4/BN.md)

[iBGP: IP-CONNECTIVITY](https://github.com/dknet77/VxLAN/tree/main/LABS/1-4/iBGP-CHECK.txt)
