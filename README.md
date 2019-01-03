## 首先声明一点，本教程中配置例子，均为生产环境直接复制，但关键数据都已经修改
# FireWall
解析防火墙的配置的一些思路和参数



防火墙的解析都差不多，配置文件中都包含以下几个方面：地址定义，接口定义，服务定义，策略定义，静态路由定义（转发方向）把握好这几个定义


# 山石防火墙参数说明： #

地址：address id 名字 网段,定义一个网络对象的名字

    eg:
      address id 37 "127.0.0.0/8"
        ip 127.0.0.0/8

如果某个ip对应了两个IP段，即对应2个address，这两个address都要去策略中去匹配，如果都在策略中命中，取最前面的策略
    
服务：定义协议类型和端口号，定义协议+端口号

      eg: 
      service "TCP3000"
        tcp dst-port 3000 
  
 策略：

       rule id 59
        action 允许还是拒绝：permit/deny
        src-zone "trust"
        dst-zone "untrust"
        src-addr 网络对象名字
        dst-addr 网络对象名字
        service  协议+端口号名字
       exit


接口：

    interface ethernet0/0
      zone  "mgt"
      ip address 11.161.241.5 255.255.255.255
      description "connect.................... YQA-OPN-F05DWGL-SW2 G1/0/9"
      manage ip 112.11.21.3
      manage ssh
      manage ping
      manage snmp
      manage https
    exit
    
这里配置了出入物理接口，我们根据入口ip，判断入口ip是否在ip address定义的网段中，是的话，接口名字就是ethernet0/0，出接口同样
特别注意，入接口只需要对比精确ip就可以了，就是比较是否等于11.161.241.5，是的话就可以了，出接口才需要判断网段

转发规则：

  如果数据报文可以过墙，那么就要确定转发的方向，以下是转发规则的格式：ip route IP网段  转发IP

       ip vrouter "trust-vr"
          ip route 11.160.128.0/17 11.160.240.14
          ip route 0.0.0.0/0 11.160.240.22
       exit

  0.0.0.0是默认转发的地址，如果dstZone为untrust转发的地址就是11.160.240.22，即默认地址
  如果dstZone是trust，我们去查询其他网段
  我们可以根据目的地址得出是否目的地址在转发规则列出的网段中，如，11.160.128.0在11.160.128.0/17中，那么转发的地址就是11.160.240.14
  如果目的地址在转发规则列出的网段中找不到，我们就确定转发地址是默认转发地址


如果判断IP在不在网段中，可以使用如下方法，效率很慢很慢

    ip = IP('74.0.193.20/32')
     for x in ip:
       print(str(x) == '74.0.193.20')
         if x == '74.0.193.20':
          print("yes")
         
         
 推荐一种高效方法：

 先将ip转换成32位二进制,再按照正反掩码适配，在网段中返回0，不在返回1

    # 转换成32位二进制
    def ipTransByte(ipParam):
        addrIpTemp = ipParam.split(".")
        tNo0 = bin(int(addrIpTemp[0])).__str__()[2:]
        tNo0 = "0" * (8 - len(tNo0)) + tNo0
        tNo1 = bin(int(addrIpTemp[1])).__str__()[2:]
        tNo1 = "0" * (8 - len(tNo1)) + tNo1
        tNo2 = bin(int(addrIpTemp[2])).__str__()[2:]
        tNo2 = "0" * (8 - len(tNo2)) + tNo2
        tNo3 = bin(int(addrIpTemp[3])).__str__()[2:]
        tNo3 = "0" * (8 - len(tNo3)) + tNo3
        return tNo0 + tNo1 + tNo2 + tNo3
        
      # 函数：IP地址与地址段进行适配比较(正掩码)in=>0  not in=>1
        def ipAdd_Match_Mask(ip1, ip2, mask):
            ipResult = 0
            for i in range(0, 32):
                if (mask[i] == "0"):
                    continue
                elif ((mask[i] == "1") and (ip1[i] == ip2[i])):
                    continue
                else:
                    ipResult = 1
                    break
            return ipResult

        # 函数：IP地址与地址段进行适配比较(反掩码)
        def ipAdd_Match_WMask(ip1, ip2, wmask):
            ipResult = 0
            for i in range(0, 32):
                if (wmask[i] == "1"):
                    continue
                elif ((wmask[i] == "0") and (ip1[i] == ip2[i])):
                    continue
                else:
                    ipResult = 1
                    break
            return ipResult
         
         
         
         
  # 飞塔防火墙配置参数说明 #

  地址：

      config firewall address   ---------------- 地址配置开始
        edit "all"  ---------------- 地址名字
        set associated-interface ''
        set color 0
        set comment ''
        set type ipmask
        set subnet 0.0.0.0 0.0.0.0                  --------------IP+掩码（单一的ip主机地址）
    next
    edit "SSLVPN_TUNNEL_ADDR1"
        set associated-interface ''
        set color 0
        set comment ''
        set type iprange
        set end-ip 11.222.124.20                    -----------------ip段
        set start-ip 11.222.124.200
    next
    edit "128.0.0.0/8"
        set associated-interface "port2"
        set color 0
        set comment ''
        set type ipmask
        set subnet 128.0.0.0 255.0.0.0
    next

举例：128.*.160-191.255（128.0.160.0-128.255.191.255）范围的地址可以定义如下

      edit "128.0.160.0/255.0.224.0"
       set associated-interface port2
        set type wildcard                      -------------反掩码的标识，默认是ipmask
        set wildcard 128.0.160.0 255.0.224.0   -------------实际地址+反掩码
    end                                        ------------------地址配置结束
    
  服务：
  
  单个短连接服务，如TCP/7001要求输出格式如下：
    
        config firewall service custom
          edit 服务名称
                set protocol TCP/UDP/SCTP
                set 服务类型-portrange 服务端口
          end
          edit 服务名称
                set protocol TCP/UDP/SCTP
                set 服务类型-portrange 服务端口1  服务端口2  服务端口n
           end
       eg.
       config firewall service custom
          edit UDP/7001-9001
                set protocol TCP/UDP/SCTP
                set udp-portrange 7001-9001
           end
           edit UDP/7001
                set protocol TCP/UDP/SCTP
                set udp-portrange 7001
            end
            edit UDP/9801-9820/1380/80/7001-9001
                set protocol TCP/UDP/SCTP
                set udp-portrange 9801-9820 1380 80 7001-9001
              end
    
策略：

普通策略定义格式如下：

        config firewall policy
            edit 策略数字ID
                set srcintf 源接口
                set dstintf 目的接口
                    set srcaddr 源地址名称1 源地址名称2 源地址名称n
                    set dstaddr 目的地址名称1 目的地址名称2 目的地址名称n
                set action accept
                set schedule always
                    set service 服务端口名称1 服务端口名称2  服务端口名称n
            end
         eg.
         config firewall policy
            edit 100
                set srcintf port2
                set dstintf port1
                    set srcaddr  128.64.183.25/32 128.64.183.3/32
                    set dstaddr all //注：这里目的地址为所有地址
                set action accept
                set schedule always
                    set service TCP/7001-9001 TCP/7001/8001/9001 
               end
         
         
在普通策略的基础上再定义set session-ttl 43200

        config firewall policy
            edit 策略数字ID
                set session-ttl 43200
         end
         eg.
             config firewall policy
                edit 100
                    set srcintf port2
                    set dstintf port1
                        set srcaddr  128.64.183.25/32 128.64.183.3/32
                        set dstaddr all //注：这里目的地址为所有地址
                    set action accept
                    set schedule always
                        set service ANY //注：这里服务端口为所有端口
                    set session-ttl 43200
                end
                
                
                
                
物理接口：

        config system interface
            edit "port1"
                set ip 114.12.214.190 255.255.255.248
             next
        end
        上个设备的下一跳是否在set ip的网段中，如果在的话，物理接口就是port1



# SRX防火墙配置及参数说明 #

服务：

根据目的端口及协议，获取服务（协议+端口）

    set applications application tcp7016 protocol tcp
    set applications application tcp7016 source-port 0-65535
    set applications application tcp7016 destination-port 7016-7016
    set applications application tcp7005 protocol tcp
    set applications application tcp7005 source-port 0-65535
    set applications application tcp
	7005 destination-port 7005-7005


接口：

获取接口的物理接口（出入接口的IP判断比较和山石一致）

    set interfaces reth0 unit 31 vlan-id 31
    set interfaces reth0 unit 31 family inet address 11.168.25.30/29
    set interfaces reth0 unit 32 vlan-id 32
    set interfaces reth0 unit 32 family inet address 11.168.22.38/29

地址：

获取地址的定义（主要还是看源或者目的是否在网段中）

    set security zones security-zone trust address-book address 11.18.0.0/16 11.168.0.0/16
    set security zones security-zone trust address-book address 11.16.124.0/24 11.168.14.0/24
    set security zones security-zone trust address-book address 11.8.158.1/32 11.168.15.1/32

策略：
查找是否有策略

    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match source-address 11.133.122.0/24
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match source-address 11.133.111.0/24
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match destination-address 11.118.0.0/16
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match application tcp26
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match application tcp22
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match application tcp338
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 match application tcp1389
    set security policies from-zone untrust to-zone trust policy nas20161009_94443_1 then deny

其实思路和山石是一致的，首先查找前三个参数：协议（服务定义），源地址+目的地址的定义，出入接口的定义
然后去ACL(访问控制列表)中查找是否受控制


转发方向：

    set routing-options static route 1.130.0.0/16 next-hop 11.170.226.254
    set routing-options static route 0.0.0.0/0 next-hop 11.168.225.33
    set routing-options static route 1.18.0.0/16 next-hop 11.168.225.25


使用目的地址判断转发方向，看目的地址是在网段中，如果在，后面的next-hop就是下一跳（本设备出口后的转发地址）
还是和山石一致，0.0.0.0就是默认的转发方向，如果都没有匹配到，最后转发到0.0.0.0/0所指定的nexthop

注：外网是untrust , DMZ(或者说内网)是trust 



图解：
    防火墙通过源地址，目的地址，只是为了判断过不过墙
    
   通过入口，为了判断转发的方向即出口（下一跳）
   
  ** 源地址----------n层设备-------------入口--防火墙--出口------------n层设备------目的地址**
   
**   策略是判断从源到目的通不通，通的话才有下一跳，当前设备的出口就是下一个设备的入口，当前设备的入口就是上一个设备的出口**
