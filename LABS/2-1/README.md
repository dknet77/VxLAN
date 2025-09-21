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
 
  	ip prefix-list CONNECTED seq 10 permit 10.8.0.0/24
 	ip prefix-list CONNECTED seq 20 permit 10.0.0.11/32
  	ip prefix-list CONNECTED seq 30 permit 10.2.1.1/31
 	ip prefix-list CONNECTED seq 40 permit 10.2.2.1/31

 	 route-map CONNECTED permit 10
   		 match ip address prefix-list CONNECTED
   		 set origin igp

 	 ip as-path access-list OWN-AS seq 10 permit "^$"
	
	  route-map SPINE-IPV4-IN permit 10
  		  match as-path OWN-AS
   		 set local-preference 150

 	 route-map SPINE-IPV4-OUT permit 10

	  route-map SPINE-IPV6-IN permit 10
  		  match as-path OWN-AS
  		  set local-preference 150

  	route-map SPINE-IPV6-OUT permit 10

	router bgp 64512
	  router-id 10.0.0.11
 	 timers bgp 3 9
 	 reconnect-interval 12
 	 address-family ipv4 unicast
	  	  redistribute direct route-map CONNECTED
   		 maximum-paths ibgp 2
 	 address-family ipv6 unicast
	    network fd12:3456:789a::bb:11/128
  	    maximum-paths ibgp 2

	template peer-session SPINE-SESSION
  	    bfd
    	remote-as 64512
   		 password cisco

	template peer-policy SPINE-IPV4-POLICY
 	   send-community extended
   	   route-map SPINE-IPV4-IN in
	   route-map SPINE-IPV4-OUT out
  	   maximum-prefix 30000
       soft-reconfiguration inbound always

	template peer-policy SPINE-IPV6-POLICY
	    send-community extended
	    route-map SPINE-IPV6-IN in
	    route-map SPINE-IPV6-OUT out
 	   maximum-prefix 30000
  	  soft-reconfiguration inbound always
	
    template peer SPINE-IPV4
 	   inherit peer-session SPINE-SESSION
   		 address-family ipv4 unicast
    	inherit peer-policy SPINE-IPV4-POLICY 10 ! the preference value for this peer policy

	template peer SPINE-IPV6
  	  inherit peer-session SPINE-SESSION
  	  address-family ipv6 unicast
      inherit peer-policy SPINE-IPV6-POLICY 10

	neighbor 10.2.1.0
	    inherit peer SPINE-IPV4
 	   description *** SPINE-1-1 ***
	
	neighbor fd12:3456:789a:ffff:ffff:dddd:1:0
	    inherit peer SPINE-IPV6
		description *** SPINE-1-1 ***

#### %%%%%%%%%%%%%%%%%%%%%%%%% LEAF (L-1-2) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%

	ip prefix-list CONNECTED seq 10 permit 10.9.0.0/24
	ip prefix-list CONNECTED seq 20 permit 10.0.0.12/32
	ip prefix-list CONNECTED seq 30 permit 10.2.1.3/31
	ip prefix-list CONNECTED seq 40 permit 10.2.2.3/31
	!
	route-map CONNECTED permit 10
	  match ip address prefix-list CONNECTED
	  set origin igp
	!
	ip as-path access-list OWN-AS seq 10 permit "^$"
	!
	route-map SPINE-IPV4-IN permit 10
	  match as-path OWN-AS
 	 set local-preference 150
	!
	route-map SPINE-IPV4-OUT permit 10
	!
	route-map SPINE-IPV6-IN permit 10
	  match as-path OWN-AS
	  set local-preference 150
	!
	route-map SPINE-IPV6-OUT permit 10
	!
	router bgp 64512
	  router-id 10.0.0.12
 	 timers bgp 3 9
  	reconnect-interval 12
 	 address-family ipv4 unicast
  	  redistribute direct route-map CONNECTED
  	  maximum-paths ibgp 2
  	address-family ipv6 unicast
	    network fd12:3456:789a::bb:12/128
 	   maximum-paths ibgp 2

	template peer-session SPINE-SESSION
	    ! bfd
  	   remote-as 64512
 	   password cisco

	template peer-policy SPINE-IPV4-POLICY
 	   send-community extended
 	   route-map SPINE-IPV4-IN in
 	   route-map SPINE-IPV4-OUT out
  	   maximum-prefix 30000
 	   soft-reconfiguration inbound always

	template peer-policy SPINE-IPV6-POLICY
 	   send-community extended
 	   route-map SPINE-IPV6-IN in
  	   route-map SPINE-IPV6-OUT out
  	  maximum-prefix 30000
  	  soft-reconfiguration inbound always
	
	template peer SPINE-IPV4
	    inherit peer-session SPINE-SESSION
  	   address-family ipv4 unicast
       inherit peer-policy SPINE-IPV4-POLICY 10 ! the preference value for this peer policy

	template peer SPINE-IPV6
	    inherit peer-session SPINE-SESSION
 	    address-family ipv6 unicast
        inherit peer-policy SPINE-IPV6-POLICY 10

	neighbor 10.2.1.2
	    inherit peer SPINE-IPV4
 	   description *** SPINE-1-1 ***
	
	neighbor fd12:3456:789a:ffff:ffff:dddd:1:2
	    inherit peer SPINE-IPV6
		description *** SPINE-1-1 ***

#### %%%%%%%%%%%%%%%%%%%%%%%%% SPINE (S-1-1) %%%%%%%%%%%%%%%%%%%%%%%%%%%%%

	route-map CONNECTED permit 10
	  match interface loopback1
 	 set origin igp
	!
	router bgp 64512
	  router-id 10.0.11.1
	  timers bgp 3 9
	  reconnect-interval 12
	  address-family ipv4 unicast
	    redistribute direct route-map CONNECTED
  	    maximum-paths ibgp 2
 	 address-family ipv6 unicast
   	   network fd12:3456:789a::aa:11/128
       maximum-paths ibgp 2
	!	
	 neighbor fd12:3456:789a:ffff::/64  ! range
	    remote-as 64512
	    password cisco
	    maximum-peers 5
	    address-family ipv6 unicast
   	    route-reflector-client
        next-hop-self all
	! 
	 neighbor 10.0.0.0/8  ! range
       remote-as 64512
 	   password cisco
   	   maximum-peers 5
  	  address-family ipv4 unicast
      route-reflector-client
      next-hop-self all

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
