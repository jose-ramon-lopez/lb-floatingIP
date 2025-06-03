# lb-floatingIP (aka Direct Server Return)
Explicación de lo que hace el parámetro "Enable Floating IP".
  
- Si no está puesto el load balancer cambia la IP de destino de la suya y pone la IP del back end.
- Si está puesto el load balancer deja como IP de destino la IP del load balancer.

# Infraestructura

vm1 (10.1.0.5) -- lb (10.1.0.4) -- vm2(10.1.0.6)


# Sin enable float IP
VM de origen. Ataco a la IP del load balancer
<pre>
root@vm1:/home/jose# telnet 10.1.0.4 22
Trying 10.1.0.4...
Connected to 10.1.0.4.
Escape character is '^]'.
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.11
</pre>
  

VM de destino. Veo que la IP de destino es la del back end
<pre>
root@vm2:/home/jose# tcpdump -n src 10.1.0.5
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:13:58.350210 IP 10.1.0.5.38958 > 10.1.0.6.22: Flags [S], seq 2863431547, win 64240, options [mss 1410,sackOK,TS val 3854993917 ecr 0,nop,wscale 7], length 0
21:13:58.352101 IP 10.1.0.5.38958 > 10.1.0.6.22: Flags [.], ack 1760649477, win 502, options [nop,nop,TS val 3854993919 ecr 2398501899], length 0
21:13:58.362420 IP 10.1.0.5.38958 > 10.1.0.6.22: Flags [.], ack 44, win 502, options [nop,nop,TS val 3854993930 ecr 2398501910], length 0
21:14:03.817764 ARP, Reply 10.1.0.5 is-at 12:34:56:78:9a:bc, length 28
</pre>


# Con enable float IP
VM de origen. Ataco a la IP del load balancer
<pre>
telnet 10.1.0.4 22
Trying 10.1.0.4...
</pre>
  
  
VM de destino. Veo que la IP de destino es la del load balancer
<pre>
tcpdump -n src 10.1.0.5
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:17:59.275887 IP 10.1.0.5.49270 > 10.1.0.4.22: Flags [S], seq 3669347256, win 64240, options [mss 1410,sackOK,TS val 3855234844 ecr 0,nop,wscale 7], length 0
21:18:00.288965 IP 10.1.0.5.49270 > 10.1.0.4.22: Flags [S], seq 3669347256, win 64240, options [mss 1410,sackOK,TS val 3855235858 ecr 0,nop,wscale 7], length 0
</pre>



# Jugemos a intentar que con floating IP habilitado podamos logearnos
Miro mi interfaz de red
<pre>
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 7c:ed:8d:27:f1:46 brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.6/24 metric 100 brd 10.1.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::7eed:8dff:fe27:f146/64 scope link
       valid_lft forever preferred_lft forever
</pre>
  

Añado la IP del load balancer a mi interfaz de red
<pre>
ip addr add 10.1.0.4/32 dev eth0
</pre>
  

Habilito reverse path filtering, en caso Ubuntu piense que hay tráfico asimétrico y no lo bloquee
<pre>
echo 2 | sudo tee /proc/sys/net/ipv4/conf/all/rp_filter
echo 2 | sudo tee /proc/sys/net/ipv4/conf/eth0/rp_filter
</pre>
  

Compruebo que llego incluso con floating IP
<pre>
telnet 10.1.0.4 22
Trying 10.1.0.4...
Connected to 10.1.0.4.
Escape character is '^]'.
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.11
</pre>
  

Si quiero guardar los cambios
<pre>
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.eth0.rp_filter=2
<pre>

aplico los cambios
<pre>
sudo sysctl -p
</pre>

